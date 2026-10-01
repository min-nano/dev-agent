# dev-agent 設計書 — クラウドエージェントとローカルエージェントの分業

> 状態: **草案（v0.1）**。方針をユーザーと合意するための文書で、実装はまだ無い。
> 合意できた節から `docs/design.md`（設計の「なぜ」）と `CLAUDE.md`（規則と手順）へ
> 切り出していく。

## 0. 要約

- **クラウドエージェント（Claude Code のクラウドセッション／ルーティン）が土台を書き、
  ローカルエージェントが実機で確かめて直す**。両者をつなぐ唯一の連絡路は **GitHub（PR の
  ラベル・コメント・リリース・チェック）** にする。クラウドから Mac へ入ってくる口は開けない。
- 開発版の配布・入れ替え・実機テストの駆動・結果の投稿を、**各アプリ／プラグインから
  1 つの Mac アプリ `dev-agent` へ引き上げる**。アプリ側に残すのは「外から呼べる入口」だけ。
- ローカルエージェントの「頭脳」は新しく作らず、**Claude Code（`claude -p` と
  Remote Control）を基本に、簡単な直しだけ Ollama 経由のローカル LLM に回す**。
  同じハーネス（`CLAUDE.md`・hooks・permissions）が両方で効くので、頭脳を切り替えても
  作法が変わらない。
- **リポジトリごとの「期待する動作」は、そのリポジトリの `dev-agent.yaml`（マニフェスト）に
  宣言しておき**、dev-agent が PR ごとの依頼と合わせてローカルエージェントへ渡す。
- 既存サービスで置き換えられるものは置き換える（第 3 節）。dev-agent 自身は、既存サービスが
  持たない 3 つ——**アプリ固有の入れ替えと起動（アダプタ）・マニフェスト・結果の投稿**——
  に絞って薄く作る。

## 1. 背景と動機

### 1.1 いまの開発フロー

| リポジトリ | 種類 | 開発版の配布 | 実機での確認 | 結果の戻し方 |
| --- | --- | --- | --- | --- |
| `vectorworks-plugin-import-ifc-homeskz` | Vectorworks 2026 の C++ プラグイン（殻＋本体のホットリロード） | CI が PR ごとに dev プレリリースを公開。**プラグイン自身**がアップデータを持ち、同梱スクリプトで入れ替える | 利用者が Vectorworks で取り込みを実行して目視 | **プラグイン自身**が「実機テストを実行…」コマンドで結果を PR へ投稿（`<!-- homeskz-ifc-feedback v1 … -->`） |
| `vectorworks-developer-sdk-reference` | SDK の実測知見＋実機確認プラグイン（VwSdkProbes） | 転がりタグ `probes` のプレリリース。**プラグイン自身**がピッカー先頭から入れ替える | 利用者がプローブを選んで走らせる | **プラグイン自身**が結果を PR／issue へ投稿（`<!-- vw-probes-result v1 … -->`） |
| `photogrammetry` | macOS アプリ（Swift・RealityKit）＋ CLI | タグ `stable` / `dev-<slug>` のリリース。**アプリ自身**が `UpdateFeed` / `UpdaterService` で入れ替える | 利用者が Mac で生成を目視 | 無し（チャットで所見） |
| `agent-mlx` | iOS / macOS アプリ（Swift・MLX）＋ CLI | 同上（iOS は未署名 .ipa で手動） | 利用者が実機で速度と会話を目視 | 無し |
| `portal` | Web（Vite＋FastAPI＋Firebase Hosting／Cloud Run） | PR ごとのプレビュー環境（`preview.yml` が URL を PR へコメント） | 利用者がブラウザで確認 | 無し |

共通しているのは次の 3 点で、これが今回の出発点になる。

1. **「実機でしか分からないこと」が多い。** Vectorworks の描画・Metal を使う推論・
   RealityKit の生成・iPhone のメモリ上限・ブラウザでのログインは、クラウドの Linux
   コンテナからは見えない。クラウドエージェントは推測で進めるしかなく、往復 1 回ごとに
   CI（30 分以上のビルドを含む）と人の目視が挟まる。
2. **配布・入れ替え・結果投稿の仕組みを、アプリごとに実装している。** 4 つのリポジトリに
   同じ形のアップデータがあり（photogrammetry → agent-mlx は移植）、Vectorworks 側は殻の
   中にアップデータと往復の駆動を抱えている。アプリが重くなるだけでなく、**仕組みを直す
   たびに利用者のアプリの再起動や再インストールを強いる**（Vectorworks の殻 ID の問題は
   その典型）。
3. **人が「入れ替えて走らせる」係になっている。** 新しい dev ビルドが出るたびに、人が
   ピッカーを開き、入れ替え、走らせ、所見をチャットへ書く。ここを自動化しない限り、
   クラウドエージェントがどれだけ賢くても往復は人の手の速さで止まる。

### 1.2 変えたいこと

- **クラウドで作り、ローカルで仕上げる** 分業にする。クラウドエージェントは引き続き設計・
  実装・PR 作成を担い、ローカルエージェントは「入れ替える → 走らせる → 観る → 小さく
  直す → 結果を返す」を人の代わりに回す。
- **実行基盤を 1 つのアプリ（dev-agent）へ切り出す。** 各アプリ／プラグインはアップデータと
  往復の駆動を手放し、**外から呼べる入口（CLI・URL スキーム・メニューコマンド）だけ**を
  残す。
- **リポジトリごとに期待する動作を宣言し、ローカルエージェントへ渡す。** プラグイン・
  独立アプリ・Web で制約も最適な手順も違い、しかも同時並行で動く。だから「何を入れ替え、
  どう起動し、何をもって合格とし、どこまで直してよいか」はリポジトリ側に書いておく。

### 1.3 変えないこと

- **GitHub 中心の流れ**（PR・CI・dev プレリリース・PR コメントによる機械可読な往復）は
  そのまま使う。既存の `<!-- … -->` マーカー方式は実績があり、dev-agent もこれを踏襲する。
- **各リポジトリの CI がビルドする**。dev-agent は Mac でビルドしない（CI の成果物を
  入れ替えるだけ）。clang-tidy や署名を含む「正しいビルド」は CI の責務のまま。
- **人が最終判断する。** ローカルエージェントが直せるのは、マニフェストで許した範囲の
  小さな直し（ビルド・テスト・lint の失敗、数値のずれ）だけで、設計判断と「確認できた」の
  宣言は人が行う。`draw/` を含む PR をユーザーの実機確認なしにマージしない、という既存の
  規約は変えない。

## 2. 目標と非目標

### 目標

| # | 目標 | 測り方 |
| --- | --- | --- |
| G1 | 新しい dev ビルドが出てから実機の結果が PR に載るまで、**人の操作ゼロ** | dev プレリリース公開 → 結果コメントまでの時間（目標 15 分以内、ビルド時間を除く） |
| G2 | **ビルド・テスト・lint の失敗の一次対応をローカルで閉じる** | クラウドエージェントが CI の赤に対応する回数が減る |
| G3 | 各アプリから**アップデータと往復の駆動を撤去**し、アプリを軽くする | Vectorworks の殻の `VW_SHELL_INPUTS` が減る。Swift アプリから `*Updater` ターゲットが消える |
| G4 | **リポジトリの種類が増えても dev-agent の本体を変えない** | 新しい種類は「アダプタ 1 つ＋マニフェスト」で足せる |
| G5 | **止められる・絞れる・見える** | 停止の合図 1 つで全リポジトリの往復が止まる。何を入れ替え何を走らせたかが PR と GUI の両方で読める |

### 非目標

- クラウドエージェントの置き換え。ローカル LLM に設計や大きな実装をさせない。
- Mac 以外（Windows 版 Vectorworks）の実機を dev-agent から動かすこと。マニフェストの
  形式は OS を限定しないが、最初の実装は macOS だけ。
- 汎用の CI ランナーを作ること。CI は GitHub Actions のまま。
- GUI を作り込むこと。人が見るのは PR が主で、GUI は「今なにをしているか」と
  停止・入れ替えの操作に絞る。

## 3. 既存サービス・既存アプリで賄えるもの（まず検討した）

「ローカルで動くエージェント」「クラウドからローカルへ仕事を渡す経路」「ローカル LLM の
実行基盤」の 3 つは、2026 年 10 月時点でそれぞれ既製のものがある。**作る前に、何を既製で
済ませ、何が残るかを切り分けた。** 確認水準は、公式ドキュメントを読んだ範囲（【文書根拠】）
で、実機での確認は M0（第 6 節）で行う。

### 3.1 候補の一覧

| 候補 | 何ができるか | 使えるか | 使い方／使わない理由 |
| --- | --- | --- | --- |
| **Claude Code Remote Control**（`claude remote-control`） | Mac 上で動く Claude Code のセッションを claude.ai／クラウドセッションから操作できる。実行とファイルアクセスは Mac 側。Pro / Max で使える。外向き HTTPS だけで、Mac に口を開けない | **採用（ローカルの「賢い頭脳」）** | クラウドエージェントが「実機で確かめて」と依頼する相手になる。Mac で常駐させ（`--spawn worktree` で PR ごとに worktree）、クラウドセッションの `send_message` で依頼が届くかを M0 で確かめる。制約: `ANTHROPIC_BASE_URL` を向け替えると使えない（＝ローカル LLM とは別プロセスになる）。初回は TTY で信頼の確認が要る |
| **Claude Code 非対話モード**（`claude -p --bare`） | スクリプトから 1 回分の仕事を投げられる。`--allowedTools` / `--permission-mode` / `--json-schema` で縛れる。終了コードと JSON で結果が取れる | **採用（定型の直しの実行器）** | dev-agent が「失敗ログ＋マニフェストの期待＋許した範囲」を渡して走らせる。頭脳は環境変数で切り替える（次項） |
| **Ollama の Anthropic 互換 API**（`/v1/messages`） | `ANTHROPIC_BASE_URL=http://localhost:11434` で **Claude Code をそのままローカル LLM で動かせる**。tool use 対応。推奨モデルは `qwen3-coder` / `gpt-oss:20b`。Apple Silicon で MLX 最適化あり | **採用（ローカルの「安い頭脳」）** | 同じハーネス（`CLAUDE.md`・hooks・permissions・skills）が効くので、頭脳を変えても作法が変わらない。LM Studio（OpenAI 互換）や oMLX も候補だが、Anthropic 互換で Claude Code を直接つなげる点で Ollama を第 1 候補にする。`tool_choice` 強制・プロンプトキャッシュは無い |
| **Claude Code ルーティン**（クラウド。API／GitHub イベント／スケジュール） | GitHub の PR イベントや HTTP POST でクラウドセッションを起こせる。SDK リファレンスの `issue-webhook` が既に使っている | **採用（クラウド側の起点。既に使用中）** | 「dev-agent の結果が PR に載った → クラウドが読む」は、PR 購読（`subscribe_pr_activity`）とルーティンで賄える。ローカルでは走らない |
| **Claude Code Desktop のローカル・スケジュールタスク** | Mac 上で、Desktop アプリが開いている間、最短 1 分間隔で Claude のセッションを起こせる。ローカルのファイルと道具に届く | **補助（M0 の足場）** | 何も作らずに「ローカルで Claude が PR を見に行く」を試せる。ただし毎回 LLM が起きるので**待ち受けのポーリングには向かない**（使用量を食う）。常用の監視は dev-agent が LLM 無しで行う |
| **Claude Code セルフホスト環境** | クラウドセッションを自前のランナーで走らせる | **使えない** | Team / Enterprise 限定（公開ベータ）。個人のプランでは対象外。該当するなら最有力なので、プラン次第で再検討 |
| **GitHub Actions のセルフホストランナー**（Mac） | PR のワークフローを Mac で走らせられる。ログと結果が GitHub のチェックとして残る。実績が多い | **第 2 段階で併用を検討** | 「Mac でビルド・テストを走らせる」だけなら最短。欠点: (1) GUI アプリ（Vectorworks）を動かすにはログイン中のユーザーセッションで動かす必要がある、(2) **公開リポジトリ（portal）では使ってはいけない**（誰の PR でも任意コードが走る）、(3) 入れ替え・起動・観測のアプリ固有の処理は結局スクリプトとして書く。dev-agent の `run --pr N` をランナーのジョブから呼ぶ形なら両立するので、ワークフロー側に寄せたくなったときに足す |
| **OpenCode / Goose / Aider / Codex CLI**（ローカル LLM 対応のコーディングエージェント） | いずれも Ollama 等で動く。OpenCode は `opencode serve` で HTTP API から駆動できる | **採用しない（当面）** | Claude Code＋Ollama で同じことができ、しかもハーネスを 1 つに保てる。Claude Code をローカル LLM で動かした成績が悪ければ、OpenCode の serve モードを第 2 候補として試す |
| **agent-mlx（自作の MLX チャットアプリ）** | MLX でローカル推論。CLI あり | **頭脳としては使わない** | HTTP サーバも tool-call も無く、CLI は 1 往復ごとにモデルを読み直す。エージェントの推論基盤にするには Ollama 相当を作ることになり、本題から外れる。agent-mlx は「dev-agent が実機で確かめる対象」の 1 つとして扱う |
| **Tailscale / Cloudflare Tunnel / ngrok** | クラウドから Mac へ届く口を作る | **使わない** | クラウドセッションの egress は許可リスト制で、しかも Mac に口を開ける必要が無い。連絡は GitHub（ラベル・コメント・リリース）と Remote Control の外向き接続で足りる |

### 3.2 既製で賄えないもの＝dev-agent が担うもの

上の表から、残るのは次の 3 つだけで、**dev-agent はこの 3 つに絞って薄く作る**。

1. **アプリ固有の入れ替え・起動・観測（アダプタ）。** 「dev プレリリースの zip を取ってきて
   Plug-Ins に置き、Vectorworks を起動し、取り込みを走らせ、本文を回収する」はどの既製品も
   知らない。しかも種類ごとに違う（第 4.5 節）。
2. **リポジトリごとの期待する動作（マニフェスト）。** 何を入れ替え、どう起動し、何をもって
   合格とし、どこまで直してよいかの宣言と、それをローカルエージェントへ渡す仕組み。
3. **往復の進行と結果の投稿。** 新しいビルドの検知・周の管理・停止の合図・結果コメントの
   投稿・トークンの保管。今は 4 つのリポジトリが別々に持っているものを 1 か所にする。

逆に、**「考える」部分は一切作らない**。頭脳は Claude Code（Remote Control／`claude -p`）
と Ollama で、dev-agent は材料を揃えて渡し、結果を回収するだけにする。

## 4. 全体像

### 4.1 役割分担

```
        ┌──────────────────────────────── GitHub（唯一の連絡路）─────────────────────────────┐
        │  PR（ラベル・コメント）   dev プレリリース（CI が公開）   チェック・レビュー   issue  │
        └──▲────────────┬──────────────────────▲─────────────────────────▲──────────────────┘
           │依頼        │結果                   │                         │
           │（ラベル＋   │（コメント）           │push                     │
           │ コメント）  ▼                       │                         │
   ┌───────┴────────────────┐          ┌─────────┴──────────────────────────┴─────────────────┐
   │ クラウドエージェント     │          │ Mac                                                   │
   │ Claude Code クラウド     │          │  dev-agent（常駐。LLM を持たない）                     │
   │ セッション／ルーティン   │          │   ├ Watcher   : PR と Releases を見張る（認証付き）    │
   │  - 設計・実装・PR        │          │   ├ Manifest  : repo の dev-agent.yaml を読む          │
   │  - 依頼を書く            │          │   ├ Adapters  : vw-plugin / mac-app / ios-app / web … │
   │  - 結果を読んで次へ      │          │   ├ Rounds    : 入れ替え → 起動 → 走らせる → 回収      │
   └───────┬────────────────┘          │   ├ Reporter  : 結果コメント・停止の合図               │
           │ Remote Control 経由の       │   └ AgentBridge: 下の 2 つへ材料を渡す                │
           │ 直接の依頼（任意）          │                                                       │
           └────────────────────────────┼──▶ Claude Code（Remote Control / claude -p）         │
                                        │  └▶ Claude Code + Ollama（ローカル LLM）              │
                                        │                                                       │
                                        │  確かめる対象: Vectorworks 2026 / Photogrammetry.app /│
                                        │  MLXChat.app / iPhone（devicectl）/ ブラウザ          │
                                        └───────────────────────────────────────────────────────┘
```

- **連絡路は GitHub だけ。** クラウド→ローカルの「依頼」は PR のラベルとコメント、
  ローカル→クラウドの「結果」は PR のコメント（既存の `<!-- … -->` マーカー方式）。
  クラウドセッションは PR を購読しているので、結果が載れば起きる。
- **Remote Control は近道。** 対話的に細かく指示したいときだけ、クラウドセッションが
  Mac 上の Claude Code セッションへ直接話しかける。記録を PR に残したい往復は GitHub を
  通す。
- **dev-agent は LLM を持たない。** 見張る・入れ替える・走らせる・回収する・投稿するは
  全部決定的な処理で、テストできる。考える必要があるときだけ AgentBridge が Claude Code を
  起こす。

### 4.2 1 周の流れ（往復の単位）

```
 1. クラウドエージェントが PR を作る／push する
 2. CI が dev プレリリースを公開する（既存のまま）
 3. クラウドエージェントが PR にラベル `dev-agent:verify` を付け、依頼コメントを書く
      <!-- dev-agent v1 request … -->（何を確かめ、どこまで直してよいか）
 4. dev-agent の Watcher が気付く（60 秒ごと・ETag 付きの条件付き GET）
 5. dev-agent が PR の head の dev-agent.yaml を読み、アダプタで
      入れ替え → （要れば再起動）→ 起動 → 走らせる → 本文・ログ・画面を回収
 6. 結果を PR へ投稿する
      <!-- dev-agent v1 result … result=ok|failed -->（アプリが作った本文をそのまま載せる）
 7. 失敗していて、マニフェストが直しを許していれば AgentBridge が Claude Code を起こす
      - worktree で PR のブランチを取り、失敗ログ＋期待＋許した範囲を渡す
      - 直せたらその PR のブランチへ push する（→ 2 へ戻る。周数と時間に上限）
      - 直せなければ、試したことを結果コメントの続きに書いて止まる
 8. クラウドエージェントが結果を読み、次の手を決める（設計変更・人へ質問・マージ待ち）
 9. 人は PR と dev-agent の画面で経過を見て、「確認できた」を宣言する／止める
```

**3 の依頼が無くても、新しい dev ビルドが出れば 5〜6 は回る**（今の Vectorworks の往復と
同じ）。依頼コメントは「今回は特にここを見て」「ここまでは直してよい」を足すためのもの。

### 4.3 連絡の書式（GitHub 上の約束）

ラベル（PR に付ける。付けられるのは write 権限のある人＝本人かその代理のクラウドセッション
だけなので、これが安全弁を兼ねる）

| ラベル | 意味 |
| --- | --- |
| `dev-agent:verify` | この PR の dev ビルドを実機で確かめて結果を返す（dev ビルドが更新されるたび） |
| `dev-agent:fix` | `verify` に加えて、マニフェストで許した範囲の直しを試みてよい |
| `dev-agent:stop` | この PR の往復を止める（付けた時点で走っている周は最後まで行き、結果だけ投稿する） |

コメントの 1 行目のマーカー（既存の `homeskz-ifc-feedback` / `vw-probes-result` と同じ流儀）

```
<!-- dev-agent v1 request repo=<owner/repo> pr=<N> -->          … クラウド → ローカル（任意）
<!-- dev-agent v1 result  repo=<owner/repo> pr=<N> round=<N> build=<sha7> platform=<macos-arm64> result=ok|failed|aborted -->
<!-- dev-agent v1 fix     repo=<owner/repo> pr=<N> round=<N> commit=<sha7> outcome=pushed|gave-up -->
<!-- dev-agent v1 ended   repo=<owner/repo> pr=<N> reason=<stop|closed|merged|budget> -->
<!-- dev-agent v1 control=stop -->                               … 緊急停止（その行だけの行）
```

**本文はアプリが作ったものをそのまま載せる。** Vectorworks の往復なら今の
`## 実機フィードバック round N …` 以下、プローブなら `## 実機プローブ: …` 以下がそのまま
続く。クラウド側の「届いたコメントの読み方」（各リポジトリの `docs/DEVELOPMENT.md`）を
書き換えずに済ませるため、**移行期間はアプリ固有のマーカー行も 2 行目に残す**（読み手は
「`<!-- homeskz-ifc-feedback` で始まる行」を探す実装なので、1 行目でなくても拾える
かどうかは移行時に確かめる。駄目なら dev-agent のマーカーを末尾のトレーラーにする）。

### 4.4 マニフェスト `dev-agent.yaml`（リポジトリごとの「期待する動作」）

リポジトリの直下に置き、**コードと一緒に版管理する**。dev-agent は PR の head のものを
読む（PR が自分の確かめ方を変えてよい）。CI が走らせる `build.yml` と同じ信頼水準——
本人の PR だけが対象（第 4.7 節）。

```yaml
version: 1
name: min-nano_structure                    # 表示名。結果コメントの見出しに使う
kind: vectorworks-plugin                    # アダプタの種類（第 4.5 節）
platforms: [macos-arm64]                    # 今はこれだけ。windows は将来

artifact:                                   # 何を入れ替えるか（CI の公開形式に合わせる）
  source: github-release
  dev_tag: "dev-{slug}"                     # slug は build.yml と同じ変換（tr '/:@ ' '----' …）
  stable_tag: stable
  notes: key-value                          # channel= / branch= / commit= / built=
  assets:
    macos-arm64: "min-nano_structureDev.vwlibrary.zip"

install:
  adapter: vectorworks-plugin
  app: "Vectorworks 2026"
  plugins_dir: "~/Library/Application Support/Vectorworks/2026/Plug-Ins"
  name: min-nano_structureDev
  installer: vw-install.sh                  # zip 直下。--machine --from <dir> --name <name> --plugins-dir <dir>
  restart_when: shell-id-changed            # installed-shell= と Info.plist の VWShellId を比べる

round:
  trigger: [new-build, request]             # 新しい dev ビルド／依頼コメントで 1 周
  prepare:
    - open_document: "{fixtures}/round.vwx"  # 無ければ新規
  run:
    - adapter: vectorworks-plugin
      command: run-test                      # 本体の runTestRound を外から呼ぶ（第 5.1 節）
      args: { ifc: "{fixtures}/sample.ifc", options: default }
      timeout: 15m
  collect:
    - kind: comment-body                     # アプリが作った本文（必須）
    - kind: screenshot                       # 任意。画面全体
    - kind: file
      path: "{tmp}/HomeskzIfcImport-diagnostic.log"
  report: pr-comment

expect:                                     # 人とエージェントが読む「合格の定義」
  - 全フィクスチャで取り込みが例外なく終わり、`結果:` が成功になる
  - 要素の内訳で「描けた」が「命令」と一致する
  - 「注意（描画側の異常）」の節が空である
  - 前の周と比べて要素数が減っていない（減るのは依頼コメントが予告したときだけ）

agent:                                      # 直してよい範囲
  brain: claude                             # claude（Anthropic）| local（Ollama）| none
  local_model: qwen3-coder
  escalate: local -> claude                 # local で駄目なら claude（周数の範囲で）
  allowed: [build-error, test-failure, lint, expect-mismatch]
  forbidden_paths:
    - "src/Extensions/**"                   # 殻（再起動を強いる）
    - "src/Payload*"
    - "scripts/vw-*"
    - ".github/**"
  local_build: "scripts/build-local.sh"     # 任意。push 前に手元でビルドして確かめる
  budget: { rounds: 3, minutes: 60 }
  push_to: pr-branch                        # PR のブランチへ直接 push（クラウドと同じ運用）

context:                                    # エージェントへ渡す追加の読み物
  - CLAUDE.md
  - docs/DEV-NOTES.md#実機確認の作法
```

決めごと:

- **`expect` は自然言語でよい。** 機械で判定できるものはアダプタが `result=` に畳み、
  判定できないもの（絵が崩れていないか）は人が見る。`expect` はエージェントと人の両方が
  読む「合格の定義」で、**依頼コメントの個別の指示より弱い**（依頼が勝つ）。
- **`agent.allowed` に無い失敗は直さない。** 設計判断・SDK の未知の挙動・殻の変更は
  クラウドと人に返す。`forbidden_paths` は各リポジトリの既存の歯止め（殻・境界・
  インストーラ）をそのまま写す。
- **`brain` の既定は `claude`。** ローカル LLM は、M0 で成績を測ってから
  `allowed` の一部（lint・単純なビルドエラー）に `local` を割り当てる。最初から
  ローカル LLM に全部任せない。
- **スキーマの検証は dev-agent が無 SDK・無ネットワークで行い**、エラーは結果コメントに
  載せる（マニフェストの typo で黙って何もしない、を避ける）。

### 4.5 アダプタ（インフラの種類ごとの違い）

| kind | 入れ替え | 起動と実行 | 回収 | 固有の制約 |
| --- | --- | --- | --- | --- |
| `vectorworks-plugin` | zip → `vw-install.sh --machine`（既存）。殻が変わったら Vectorworks を終了・再起動（AppleScript `quit` → `open -a`）。**再起動を外から行えるので、今の「切り離したヘルパーから終了要求」は要らなくなる** | 本体の `runTestRound` を**外から**呼ぶ。経路は既存の MCP ブリッジのスプール（`$TMPDIR/min-nano_structure-mcp/<id>.req.json`）に `vw_run_test` を足す（第 5.1 節）。公式文書の `Vectorworks -t <Test> -ab -l out.txt`（自動テストの CLI）は未確認なので、SDK リファレンスで issue を立てて確かめる | 本体が作る結果本文（今と同じ）・診断ログ・`screencapture` | GUI アプリ。ログイン中のユーザーセッションが要る。未署名プラグインの警告を 1 度許可しておく。1 周 10〜15 分 |
| `probe-plugin` | 転がりタグ `probes` の zip（`build=` で新旧を比べる。既存の `vw-probes-update.sh do-install`） | ピッカーの「一覧を順に実行」を外から呼ぶ（同じスプール方式を足す） | 既存の結果本文（`<!-- vw-probes-result … -->`） | 同上。`recovered=yes`（VW ごと落ちた）の拾い直しは本体に残す |
| `mac-app` | `*.app.zip` → `ditto -x -k` → 隔離解除・アドホック署名 → `~/Applications/dev-agent/<name>/` に置く（本番の `/Applications` には触らない） | 同梱 CLI（`photogrammetry-cli` / `mlxchat-cli`）を直接実行する。GUI の確認が要るときだけ `open` ＋ URL スキーム | stdout（`HelperProtocol` の行、`bench` のレポート）・終了コード・画面 | RealityKit / Metal は実機でしか動かない。`bench` は固定幅テキストなので、JSON 出力を足すのは各アプリ側の小さな宿題 |
| `ios-app` | `.ipa`（未署名）→ `scripts/run-ios.sh` 相当で署名・転送・起動（`devicectl`） | 起動確認と URL スキーム。結果の回収経路が無いので、まず「起動して落ちない」まで | `devicectl` のログ・画面 | 証明書とつないだ iPhone が要る。最初は手動の補助に留める |
| `web` | 無し（PR のプレビュー URL をコメントから拾う） | ブラウザで開いて煙試験（Playwright。Clerk の開発インスタンスのログイン状態を保存しておく） | 画面・コンソール・`/api/healthz` | **公開リポジトリ**なので、他人の PR では絶対に動かさない（第 4.7 節） |

アダプタは「入れ替える・起動する・走らせる・回収する」の 4 つの操作に揃え、dev-agent の
本体は種類を知らない（G4）。新しい種類は、アダプタ 1 つとマニフェストの `kind` を足すだけ。

### 4.6 ローカルエージェント（頭脳）の使い分け

| 段 | 頭脳 | 向くもの | 起こし方 |
| --- | --- | --- | --- |
| A | **Claude（Remote Control）** | 対話が要るもの。クラウドの設計と実機の見え方をすり合わせる。画面を見ながらの調整 | Mac で `claude remote-control --spawn worktree` を常駐させ、クラウドセッションが話しかける。dev-agent は関与しない |
| B | **Claude（`claude -p`）** | 定型だが判断の要るもの。実機の失敗ログから原因を当てて直す | dev-agent の AgentBridge が worktree で起こす。`--permission-mode acceptEdits` ＋ `--allowedTools` を `agent.allowed` から組む。`--json-schema` で「直したか・何を・なぜ」を受け取る |
| C | **ローカル LLM（`claude -p` ＋ Ollama）** | 単純で量の多いもの。lint・コンパイルエラーの一次対応・テストの期待値合わせ・マニフェストの `expect` との突き合わせ | B と同じコマンドに `ANTHROPIC_BASE_URL=http://localhost:11434 ANTHROPIC_AUTH_TOKEN=ollama` を付けるだけ。失敗したら `escalate` に従って B へ |

- **同じハーネスで走らせる**ことが要点。`CLAUDE.md`・`.claude/settings.json` の
  permissions・hooks・skills は 3 段すべてで効く。各リポジトリにある「殻を触らない」
  「`sleep` で待たない」といった規約が、ローカル LLM にもそのまま掛かる。
- **ローカル LLM に渡す材料は dev-agent が絞る。** 失敗ログの該当部分・`expect`・
  `allowed`・直してよいファイルだけを渡す。全文脈を読ませない（小さいモデルは文脈で
  迷う）。
- **push 前に手元で確かめる。** `agent.local_build` があれば、ローカルの SDK／Xcode で
  ビルドとテストを通してから push する。CI（30 分のビルド）は最終の門に残す。これが
  クラウドでは出来ず、ローカルだからできる最大の時間短縮になる。
- **予算で止まる。** `budget.rounds` と `budget.minutes` を超えたら `gave-up` を投稿して
  クラウドと人へ返す。無限に直し続けない。

### 4.7 安全弁

1. **対象は本人の PR だけ。** PR の作者が自分（またはクラウドセッションが自分の名義で
   作ったもの）で、fork からでないものに限る。ラベルは write 権限が無いと付けられないので
   二重の門になる。**公開リポジトリ（portal）でもこの規則で守る。**
2. **マニフェストに書けるコマンドは、アダプタの操作とリポジトリ内のスクリプトだけ。**
   任意のシェルは書けない（CI の `build.yml` と同じ信頼水準に揃える）。
3. **エージェントは worktree で動き、`forbidden_paths` に触れない。** 触ったら push せずに
   `gave-up`。
4. **消すコードは増やさない。** 入れ替えで消してよいのは、dev-agent が自分で置いた
   `~/Applications/dev-agent/` 配下と、各アプリのアンインストーラが消すと決めている
   範囲だけ（Vectorworks は既存の `vw-uninstall.sh` の歯止めをそのまま使う）。
5. **止める手段を 3 つ持つ。** ラベル `dev-agent:stop`（PR 単位）、
   `<!-- dev-agent v1 control=stop -->`（PR 単位・既存の流儀）、dev-agent の
   画面／CLI の「すべて止める」（Mac 単位）。
6. **トークンは dev-agent だけが持つ。** macOS キーチェーン（service `dev-agent`）に 1 つ。
   fine-grained PAT で、権限は Pull requests（Read/Write）・Contents（Read。Releases の
   取得）・Metadata だけ。push はエージェントが `gh` の資格情報で行う（dev-agent の
   トークンでは push しない）。**各アプリからトークンの扱いが消える。**
7. **秘密をプロンプトに載せない。** AgentBridge はトークン・鍵をエージェントの環境変数にも
   プロンプトにも渡さない。

### 4.8 dev-agent の作り

- **言語と形**: Swift（SwiftPM）。姉妹リポジトリ（photogrammetry / agent-mlx）と同じ
  「純ロジックの Core ＋ 薄い CLI ＋ 薄い GUI」の三分割にし、同じ CI・リリース・自動
  アップデートの仕組みを移植する（**dev-agent 自身の更新だけは自分で行う**。それが
  このアプリの存在理由なので例外にする）。
  - `Sources/DevAgentCore/` … マニフェストの解釈・Releases の解釈・周の状態機械・
    コメントの組み立て・エージェントへ渡す材料の組み立て。**ネットワークもプロセス起動も
    しない**純ロジックで、`swift test` で押さえる。
  - `Sources/DevAgentAdapters/` … 入れ替え・起動・回収の実装（`Process` / `FileManager` /
    `screencapture`）。判断を置かない。
  - `Sources/dev-agent/` … CLI。`watch`（常駐）／`run --repo --pr`（1 周だけ）／
    `install --repo --build`／`stop`／`status`／`manifest check`。launchd で `watch` を常駐。
  - `Apps/DevAgentApp/` … メニューバーアプリ。一覧（repo × PR × 周）・ログ・停止・
    「このビルドを入れる」。判断を持たない。
- **既存コードの流用**: `UpdateFeed`（Releases → チャンネル）はそのまま Core へ移す。
  Vectorworks の `vw-install.sh` / `vw-uninstall.sh` / `vw-probes-update.sh` は zip の
  直下にあるので、dev-agent はそれを呼ぶだけ（配置の知識を dev-agent に持ち込まない。
  これは Vectorworks 側の「同梱スクリプトに配置を書かない」規約と同じ理屈）。
- **マニフェストの形式**: YAML。SwiftPM の依存を増やさない方針との衝突があるので、
  Yams を**唯一の例外**として入れるか、JSON（`dev-agent.json`）にするかは要決定
  （第 7 節）。
- **状態の置き場**: `~/Library/Application Support/dev-agent/state/<repo>/<pr>.json`
  （周・最後に見たビルド・最後に投稿した時刻・予算の消費）。Vectorworks の
  `feedback.txt` と同じ役目をここへ移す。
- **ポーリング**: 60 秒ごと。`If-None-Match`（ETag）で 304 は上限に数えられないので、
  リポジトリが 5 つでも認証付きの 5,000 回/時に遠く届かない。

### 4.9 各アプリに何が残り、何が消えるか

| リポジトリ | 消える（dev-agent へ） | 残る（外から呼べる入口） | 殻／本体への影響 |
| --- | --- | --- | --- |
| Vectorworks プラグイン | `src/Updater*`・`src/FeedbackLoop*`・`ExtFeedbackPalette`・`scripts/vw-update.*`・`vw-feedback.*`・`vw-token.*`・`core/FeedbackSession` の周の記憶・「アップデータを確認」メニュー | `draw/Feedback` の `runTestRound`（本文の生成・図面の戻し）と、それを外から呼ぶスプールの口（`vw_run_test`）。`vw-install.*` / `vw-uninstall.*` は配布 zip に残る | **殻が小さくなる**＝`VW_SHELL_INPUTS` が減り、再起動を要する変更が減る。ABI は口が減る方向で版を上げる |
| SDK リファレンス（VwSdkProbes） | 殻の更新処理・`vw-probes-update.*`・`vw-probes-feedback.*`・トークン | プローブの実行・結果本文・`recovered=yes` の控え。ピッカーは残す（人が手で走らせる道は残す） | 同上 |
| photogrammetry / agent-mlx | `Sources/*Updater/`・設定画面の更新タブ・`install-update.sh` | CLI と URL スキーム（既にある）。`bench` に `--json` を足す | アプリから Updater ターゲットが消える。`UpdateFeed` は dev-agent 側へ移植 |
| portal | 無し（プレビューは CI が作る） | プレビュー URL のコメント（既にある） | 変更なし。dev-agent は読むだけ |

**順序は「dev-agent が同じことをできるようになってから消す」。** 消すのは M5（第 6 節）で、
それまでは両方が動く期間を置く（dev-agent が入れ替えたビルドを、アプリ側のアップデータが
「古い」と誤認して戻さないよう、移行期間はアプリ側の自動確認を切る）。

## 5. リポジトリごとの移行の要点

### 5.1 Vectorworks プラグイン（最も効果が大きく、最も注意が要る）

いまの往復は**プラグインの中**で閉じている（殻の `FeedbackLoopDriver` が 60 秒ごとに
`loop-control` → `q-dev` → `do-install` → 本体の `vw_payload_run_test` → 投稿）。これを
**dev-agent が外から回す**形に裏返す。

- **本体に残すのは `runTestRound`（絵を作り、本文を作り、図面を戻す）だけ。** 投稿も周の
  記憶も入れ替えも本体から消える。`draw/ImportCommand`（本番の取り込み）には今後も 1 行も
  書かない（既存の規約どおり）。
- **外から呼ぶ口は MCP ブリッジのスプールを再利用する。** 既に「パレットの時計が
  `<id>.req.json` を拾って `<id>.res.json` を返す」仕組みがあり、v1 は読む道具だけ。
  ここに `vw_run_test {ifc, options, out}` を 1 つ足す。dev-agent はリクエストを書き、
  レスポンス（本文のパス・結果・所要）を待つ。**殻に足すのは「時計が本体の 1 関数を取り次ぐ」
  1 行相当**で、以後は本体だけで直せる。
- **Vectorworks の起動・終了は dev-agent が行う**（`open -a "Vectorworks 2026"` /
  AppleScript `quit`）。殻が変わった周の再起動は、今の「切り離したヘルパーから OS の終了
  要求」より素直になる。保存ダイアログで止まらないよう、周の図面は保存しない設定で
  開く（既存の `prepareDrawingForRound` の歯止めはそのまま）。
- **未確認の近道が 1 つ。** 公式文書「Writing automated tests」は
  `Vectorworks -t <TestName> -ab -l out.txt` でシステムテストを CLI から走らせられると
  書いているが、このリポジトリでは実機で確かめた記録が無い。使えれば起動ごと
  dev-agent が握れる。**SDK リファレンスで issue を立てて確かめてから**設計に入れる
  （確かめるまでは上のスプール方式で進める）。
- 結果本文の形式は変えない。クラウド側の読み方の規約を壊さないため。

### 5.2 SDK リファレンス（VwSdkProbes）

- 5.1 と同じ裏返し。ピッカーの「一覧を順に実行」をスプールから呼べるようにし、更新と投稿を
  dev-agent へ移す。**ビルド ID（`build=`）で新旧を比べる規則は dev-agent 側に移植する**
  （コミットではなくビルド ID で比べる理由は `plugin/README.md` のとおり）。
- 「入れ替えずに走らせて古い結果が返る」事故（#171）は、dev-agent が**走らせる前に必ず
  入れ替える**ことで構造的に消える。
- プローブの結果は PR か issue へ返る。宛先の決め方（PR → ブランチから探す → `[issue #N]`）
  はリリース本文の `probes=` から dev-agent が引けるので、本体に残す必要は無い。

### 5.3 photogrammetry / agent-mlx

- 最初の実装対象にする（第 6 節 M1）。理由: 配布形式が単純（`*.app.zip`）、CLI が揃って
  いる、GUI を介さずに「走らせて回収」まで行ける。
- マニフェストの `round.run` は CLI を直接呼ぶ。photogrammetry は固定の写真セットで
  `process`（点群だけなら速い）と `sort`、agent-mlx は小さいモデルで `bench --runs 3`。
- `bench` のレポートは固定幅テキストなので、`--json` を足す（アプリ側の小さな宿題）。
  dev-agent は数値を前の周と比べ、しきい値を超えて遅くなったら `expect-mismatch` にする。
- iOS は「転送して起動して落ちない」まで。結果の回収経路（iPhone → Mac）は無いので、
  当面は人が見る。

### 5.4 portal（Web）

- プレビュー環境は既に CI が作り、URL が PR に載る。dev-agent の仕事は
  「URL を拾い、ブラウザで開き、ログインし、画面と `/api/healthz` を確かめる」だけ。
- Playwright で煙試験を書く（Clerk の開発インスタンスのログイン状態を保存して使う）。
  テストそのものは portal のリポジトリに置き（`dev-agent.yaml` の `round.run` から呼ぶ）、
  dev-agent は走らせて結果を回収するだけにする。
- 公開リポジトリなので、第 4.7 節の「本人の PR だけ」を最初から厳守する。

## 6. 進め方（縦切りで 1 周ずつ）

| 段 | 何をするか | 終わりの印 |
| --- | --- | --- |
| **M0 検証スパイク（作る前に確かめる）** | (a) クラウドセッションから Mac の Remote Control セッションへ依頼が届くか（`send_message` / `ListAgents`）。(b) `claude -p` ＋ Ollama（`qwen3-coder` / `gpt-oss:20b`）で、過去の実際の CI の赤（lint・コンパイル・テスト）を何割直せるか。(c) スプール経由で Vectorworks の本体の関数を外から呼べるか（既存の `vw_call` で確認）。(d) `Vectorworks -t` の実在（SDK リファレンスの issue）。(e) Desktop のローカル・スケジュールタスクで「PR を見に行って結果を返す」を手作業の代わりに 1 周回してみる | 各項目の結果を本書の付録に書く。(b) の成績で `agent.brain` の既定を決める |
| **M1 dev-agent の骨格＋mac-app アダプタ** | Core（マニフェスト・Releases・周の状態・コメント）と CLI の `run --repo --pr`。photogrammetry で「入れ替え → `photogrammetry-cli` → 結果コメント」を 1 周 | photogrammetry の PR に `<!-- dev-agent v1 result … -->` が人手ゼロで載る |
| **M2 Vectorworks アダプタ＋スプールの口** | プラグイン側に `vw_run_test` を足す（本体）。dev-agent 側に vectorworks-plugin アダプタと再起動の処理。既存の往復と**並走**させ、同じ結果が返ることを確かめる | 同じ dev ビルドに対して、プラグイン内の往復と dev-agent の往復が同じ本文を投稿する |
| **M3 AgentBridge** | `claude -p` の起動・worktree・材料の絞り込み・`--json-schema` の受け取り・予算。`brain: claude` から始め、M0(b) の成績に応じて `local` を割り当てる | 実機の失敗から dev-agent が push した修正で CI が緑になる例が 1 つできる |
| **M4 残りのアダプタ** | probe-plugin・web・ios-app。`watch`（常駐）と launchd | 対象の 5 リポジトリすべてが `dev-agent.yaml` を持つ |
| **M5 撤去と GUI** | 各アプリから Updater／往復の駆動を消す（第 4.9 節）。メニューバーアプリ。dev-agent 自身の自動アップデート | Vectorworks の殻の `VW_SHELL_INPUTS` から `Updater*` と `FeedbackLoop*` が消える |

- 各段は「1 変更＝1 周が回る縦切り」で PR にし、Vectorworks の規約と同じく**実機確認が
  要るものは下書き PR で、ユーザーの「確認できた」を待ってからマージ**する。
- M0 は本書の合意後すぐ着手できる。M1 以降は M0 の結果で順序を入れ替える余地を残す
  （たとえば (a) が通れば、M3 より先に Remote Control での対話的な往復を使い始める）。

## 7. 決めてほしいこと（返事を待つ）

1. **プラン。** Remote Control とルーティンは Pro / Max / Team / Enterprise で使える。
   Team / Enterprise なら「セルフホスト環境」（クラウドセッションを Mac で走らせる）が
   最有力になるので、本書の構成を見直す。**個人（Pro / Max）の前提で書いた。**
2. **ローカル LLM の実行基盤。** Ollama を Mac に入れる前提でよいか（agent-mlx をサーバ化
   しない）。
3. **マニフェストの形式。** YAML（Yams を唯一の依存として許す）か、JSON（依存ゼロだが
   手で書きにくい）か。**YAML を推す。**
4. **結果コメントのマーカー。** dev-agent のマーカーを 1 行目に置き、アプリ固有のマーカーを
   2 行目に残す（上記）でよいか。クラウド側の読み方を 1 行目固定にしている箇所があれば、
   そちらを直す。
5. **Windows 版 Vectorworks。** 当面は対象外でよいか（マニフェストは `platforms` で将来
   足せる形にしてある）。
6. **dev-agent の言語。** Swift（姉妹アプリと同じ作法・同じ CI を流用）を推す。Python や
   TypeScript のほうが都合がよい事情（たとえば Windows 対応を早めたい）があれば変える。

## 8. 用語

| 語 | 意味 |
| --- | --- |
| クラウドエージェント | Claude Code のクラウドセッション／ルーティン。設計・実装・PR 作成を担う |
| ローカルエージェント | Mac で動く Claude Code（Remote Control / `claude -p`）または Claude Code＋Ollama。実機で確かめ、小さく直す |
| dev-agent | 本リポジトリで作る Mac アプリ＋CLI。入れ替え・起動・回収・投稿を担い、LLM を持たない |
| 周（round） | 「入れ替え → 走らせる → 結果を投稿」の 1 回。既存の `round=N` と同じ |
| アダプタ | インフラの種類（plugin / app / web）ごとの入れ替え・起動・回収の実装 |
| マニフェスト | リポジトリ直下の `dev-agent.yaml`。期待する動作と直してよい範囲の宣言 |
| 頭脳（brain） | ローカルエージェントが使う LLM。`claude`（Anthropic）か `local`（Ollama） |

## 付録 A. 参照した資料

- Claude Code: [Remote Control](https://code.claude.com/docs/en/remote-control) /
  [Run programmatically（`claude -p`）](https://code.claude.com/docs/en/headless) /
  [Routines](https://code.claude.com/docs/en/routines) /
  [Cloud sessions](https://code.claude.com/docs/en/claude-code-on-the-web) /
  [Self-hosted environments](https://code.claude.com/docs/en/self-hosted-environments) /
  [Desktop scheduled tasks](https://code.claude.com/docs/en/desktop-scheduled-tasks) /
  [Hooks](https://code.claude.com/docs/en/hooks)
- Ollama: [Anthropic compatibility](https://docs.ollama.com/api/anthropic-compatibility)
- OpenCode: [Server](https://opencode.ai/docs/server/)
- 既存リポジトリ: `vectorworks-plugin-import-ifc-homeskz/docs/DEVELOPMENT.md`（自動
  アップデート・実機フィードバックの往復・MCP ブリッジ）、
  `vectorworks-developer-sdk-reference/plugin/README.md`（プローブの仕組み）、
  `photogrammetry` / `agent-mlx` の `Sources/*Updater/`・`build.yml`、
  `portal/.github/workflows/preview.yml`

## 付録 B. M0 の結果（未着手）

M0 の各項目の結果をここに書く。確認水準の印は SDK リファレンスの流儀に合わせる
（無印＝実機確認済み・【推定】・【文書根拠】）。
