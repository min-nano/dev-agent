# dev-agent 設計書 — クラウドエージェントとローカルエージェントの分業

> 状態: **方針合意済み（v1.0、2026-10-01）**。第 7 節の論点はすべて決着し、実装はまだ無い。
> 次は M0（第 6 節）。実装が始まったら、規則と手順は `CLAUDE.md` へ切り出し、この文書は
> 設計の「なぜ」の置き場として保つ。
>
> v1.1 案（2026-10-02）: 設計レビュー 1-1（issue #2）を反映。マニフェストを確かめ方
> （`dev-agent.yaml`。PR の head から読む）と歯止め（`dev-agent.policy.yaml`。既定
> ブランチから読む）の 2 つに分け、ハーネスも既定ブランチから渡す。エージェントが
> 書いてよいパスは許可制（既定は拒否）にし、dev-agent が常に禁じるパスを持つ
> （第 4.4・4.6・4.7 節、第 7 節 #19）。
>
> v1.2 案（2026-10-02）: 設計レビュー 1-2（issue #3）を反映。エージェントには GitHub の
> 資格情報を渡さず、囲いから github.com へも出られないようにする。直しは dev-agent が
> 1 つの commit にまとめ、検査と手元の 1 周を通ったものだけを PR のブランチではなく
> **直しのブランチ**（`dev-agent/pr-<N>/r<round>`）へ App のトークンで push する。
> 取り込むかはクラウドが決める（第 4.2・4.3・4.6・4.7・4.10 節、第 7 節 #20）。
>
> v1.3 案（2026-10-03）: 設計レビュー 1-6（issue #7）を反映。周の始めに PR を 1 回だけ
> API で読み、その応答を**周の固定値**（`RoundPin`。head の sha・作者・ラベル・歯止めの
> sha）として凍らせる。ビルド・check run・直しはすべてその sha だけを使い、ブランチ名は
> 使わない（`git fetch <URL> <sha>` → detached checkout）。決まった区切りで読み直し、
> head が動いていたらその周は `neutral` で閉じて新しい head で回し直す。push の直前は
> 必ず読み直す（第 4.2・4.3・4.4・4.7・4.8 節、第 6 節 M0(m)、第 7 節 #21）。

## 0. 要約

- **クラウドエージェント（Claude Code のクラウドセッション／ルーティン）が土台を書き、
  ローカルエージェントが実機で確かめて直す**。両者をつなぐ唯一の連絡路は **GitHub** にする
  （クラウド → ローカルは PR のラベル、ローカル → クラウドは **PR の check run**。コメント欄に
  機械的な文を並べない）。クラウドから Mac へ入ってくる口は開けない。
- 開発版の配布・入れ替え・実機テストの駆動・結果の投稿を、**各アプリ／プラグインから
  1 つの Mac アプリ `dev-agent` へ引き上げる**。アプリ側に残すのは「外から呼べる入口」だけ。
- **往復のビルドは Mac で行う。** 往復 1 回にかかる時間の大半は CI のビルド（30 分超）なので、
  dev-agent は PR の head を手元の worktree で**差分ビルド**して入れ替える。CI は lint・
  clang-tidy・サニタイザ・配布物の公開という「最終の門」に残し、往復はそれを待たない。
  ビルドするのは**API で確かめた head の sha そのもの**で、ブランチ名では取り寄せない
  （確かめた後に push されたコミットをビルドしない。第 4.2・4.7 節）。
- ローカルエージェントの「頭脳」は新しく作らず、Claude Code のハーネスを使う。
  **既定はローカル LLM（同梱 Ollama）で、Anthropic の Claude は昇格したときだけ**使う
  （クラウドの使用量を守るため）。同じハーネス（`CLAUDE.md`・hooks・permissions）が両方で
  効くので、頭脳を切り替えても作法が変わらない。代用できる範囲は M0 の計測で決め、
  付録 B の表で育てる。
- **リポジトリごとの「期待する動作」は、そのリポジトリの `dev-agent.yaml`（マニフェスト）に
  宣言しておき**、dev-agent が PR ごとの依頼と合わせてローカルエージェントへ渡す。
  直してよい範囲などの歯止めは別ファイル `dev-agent.policy.yaml` に置き、PR からは
  変えられないよう既定ブランチから読む。
- **ローカルの直しは PR のブランチへ直接入れない。** エージェントは GitHub の資格情報を
  持たず、dev-agent が検査と手元の 1 周を通った直しだけを**直しのブランチ**へ push し、
  check run に書く。PR へ取り込むかはクラウドが差分を読んで決める（第 4.2・4.7 節）。
- **環境はアプリが抱える。** ローカル LLM の実行基盤・ビルドの道具（Node・Python・
  CMake・Vectorworks SDK）・Claude Code・モデル・作業用の worktree は、すべて dev-agent が
  同梱するか初回に取り寄せ、**アプリのデータ領域の中だけ**に置く。Mac を入れ替えても
  dev-agent を入れれば環境が揃い、データ領域を消せば元に戻る（第 4.10 節）。外に残る前提は
  macOS・Xcode・Vectorworks 本体・claude.ai のログインだけ。
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
   CI と人の目視が挟まる。**往復 1 回の時間の大半は CI のビルド**（agent-mlx は片側
   30 分以上、Vectorworks プラグインも mac / Windows / tidy を含めて数十分）で、
   手元の差分ビルドなら数分で済むものを毎回待っている。
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
- **往復から CI のビルド時間を外す。** PR の head を Mac で差分ビルドして入れ替える。
  CI の役目は「正しいビルドの保証と配布」に絞り、往復の速さは手元のビルドが決める。
- **リポジトリごとに期待する動作を宣言し、ローカルエージェントへ渡す。** プラグイン・
  独立アプリ・Web で制約も最適な手順も違い、しかも同時並行で動く。だから「何を入れ替え、
  どう起動し、何をもって合格とし、どこまで直してよいか」はリポジトリ側に書いておく。

### 1.3 変えないこと

- **GitHub 中心の流れ**（PR・CI・dev プレリリース・PR コメントによる機械可読な往復）は
  そのまま使う。ただし**ローカルからの結果の戻し方はコメントから check run へ変える**
  （第 4.3 節）。既存の `<!-- … -->` マーカー付きコメントは、dev-agent へ移るまでの移行期間
  だけ残る。
- **CI が最終の門であること。** 手元のビルドは速いが、clang-tidy・サニタイザ付きの
  テスト・両 OS のビルド・配布物の公開は CI にしか無い。往復は手元のビルドで回し、
  **マージの可否は従来どおり CI の緑で決める**。dev プレリリースも引き続き公開する
  （Windows 機や別の Mac に入れるため）。
- **人が最終判断する。** ローカルエージェントが直せるのは、マニフェストで許した範囲の
  小さな直し（ビルド・テスト・lint の失敗、数値のずれ）だけで、設計判断と「確認できた」の
  宣言は人が行う。`draw/` を含む PR をユーザーの実機確認なしにマージしない、という既存の
  規約は変えない。

## 2. 目標と非目標

### 目標

| # | 目標 | 測り方 |
| --- | --- | --- |
| G1 | PR に push されてから実機の結果が PR に載るまで、**人の操作ゼロ** | push → check run の完了までに人の操作が 0 回であること（時間の目標は G6） |
| G2 | **ビルド・テスト・lint の失敗の一次対応をローカルで閉じる** | クラウドエージェントが CI の赤に対応する回数が減る |
| G3 | 各アプリから**アップデータと往復の駆動を撤去**し、アプリを軽くする | Vectorworks の殻の `VW_SHELL_INPUTS` が減る。Swift アプリから `*Updater` ターゲットが消える |
| G4 | **リポジトリの種類が増えても dev-agent の本体を変えない** | 新しい種類は「アダプタ 1 つ＋マニフェスト」で足せる |
| G5 | **止められる・絞れる・見える** | 停止の合図 1 つで全リポジトリの往復が止まる。何を入れ替え何を走らせたかが PR と GUI の両方で読める |
| G6 | **往復が CI のビルドを待たない** | push → 手元の差分ビルド → 入れ替え → check run 完了までの時間（目標: Swift アプリ 5 分以内、Vectorworks プラグイン 10 分以内。初回のフルビルドを除く） |
| G7 | **クラウド（Anthropic）の使用量を往復で増やさない** | 往復 1 周あたりの Anthropic 側の呼び出し回数（目標: 結果の読み取りと昇格時だけ。周ごとの定型処理は 0 回）。ローカル LLM が片付けた件数の割合を付録 B で追う |

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
| **Claude Code Remote Control**（`claude remote-control`） | Mac 上で動く Claude Code のセッションを claude.ai／スマホから操作できる。実行とファイルアクセスは Mac 側。Pro / Max で使える。外向き HTTPS だけで、Mac に口を開けない | **人の道具として採用。エージェント間の経路には使わない** | **人が**外出先から Mac の Claude Code を使うための窓で、dev-agent の CLI はその中から道具として呼べる。クラウドエージェントがローカルへ依頼する経路には**しない**（第 4.6 節「Remote Control の位置付け」）。dev-agent は常駐させず、人が要るときに `dev-agent rc` で起こす |
| **Claude Code 非対話モード**（`claude -p --bare`） | スクリプトから 1 回分の仕事を投げられる。`--allowedTools` / `--permission-mode` / `--json-schema` で縛れる。終了コードと JSON で結果が取れる | **採用（定型の直しの実行器）** | dev-agent が「失敗ログ＋マニフェストの期待＋許した範囲」を渡して走らせる。頭脳は環境変数で切り替える（次項） |
| **Ollama の Anthropic 互換 API**（`/v1/messages`） | `ANTHROPIC_BASE_URL=http://localhost:11434` で **Claude Code をそのままローカル LLM で動かせる**。tool use 対応。推奨モデルは `qwen3-coder` / `gpt-oss:20b`。Apple Silicon で MLX 最適化あり。MIT ライセンスで、CLI 版（`ollama-darwin.tgz`）を**アプリに同梱して再配布できる** | **採用（ローカルの「安い頭脳」。dev-agent に同梱）** | 同じハーネス（`CLAUDE.md`・hooks・permissions・skills）が効くので、頭脳を変えても作法が変わらない。利用者に別途入れてもらわず、dev-agent の `.app` の中に CLI 版を同梱し、`OLLAMA_MODELS` と `OLLAMA_HOST` をアプリのデータ領域と専用ポートへ向けて dev-agent が起動・停止する。LM Studio（OpenAI 互換）や oMLX も候補だが、Anthropic 互換を Ollama 側が保守している点が決め手。`tool_choice` 強制・プロンプトキャッシュは無い |
| **Claude Code ルーティン**（クラウド。API／GitHub イベント／スケジュール） | GitHub の PR イベントや HTTP POST でクラウドセッションを起こせる。SDK リファレンスの `issue-webhook` が既に使っている | **採用（クラウド側の起点。既に使用中）** | 「dev-agent の結果が PR に載った → クラウドが読む」は、PR 購読（`subscribe_pr_activity`）とルーティンで賄える。ローカルでは走らない |
| **Claude Code Desktop のローカル・スケジュールタスク** | Mac 上で、Desktop アプリが開いている間、最短 1 分間隔で Claude のセッションを起こせる。ローカルのファイルと道具に届く | **補助（M0 の足場）** | 何も作らずに「ローカルで Claude が PR を見に行く」を試せる。ただし毎回 LLM が起きるので**待ち受けのポーリングには向かない**（使用量を食う）。常用の監視は dev-agent が LLM 無しで行う |
| **Claude Code セルフホスト環境** | クラウドセッションを自前のランナーで走らせる | **使えない（確定）** | Team / Enterprise 限定（公開ベータ）。本件は個人の Max プランなので対象外 |
| **GitHub Actions のセルフホストランナー**（Mac） | PR のワークフローを Mac で走らせる。外向きのロングポーリングで job を受けるので数秒で反応し、job がそのまま check になる | **不採用（全リポジトリが公開のため）** | GitHub 自身が「公開リポジトリでは使うな」としている（誰でも PR を出せ、ワークフローの `if` や承認設定の誤りがそのまま Mac での任意実行につながる）。一度は採用したが、利用者の指摘で取り下げた。リポジトリが私有になることがあれば再検討 |
| **OpenCode / Goose / Aider / Codex CLI**（ローカル LLM 対応のコーディングエージェント） | いずれも Ollama 等で動く。OpenCode は `opencode serve` で HTTP API から駆動できる | **採用しない（当面）** | Claude Code＋Ollama で同じことができ、しかもハーネスを 1 つに保てる。Claude Code をローカル LLM で動かした成績が悪ければ、OpenCode の serve モードを第 2 候補として試す |
| **agent-mlx（自作の MLX チャットアプリ）／mlx-swift-lm** | MLX でローカル推論。mlx-swift-lm は tool calling（`ToolCallProcessor`。qwen / json / xmlFunction 形式）を持つ | **当面は使わない（将来の置き換え候補）** | dev-agent の中に Anthropic 互換の `/v1/messages` サーバを Swift で書けば、外部ランタイム無しで「アプリで完結」できる。ただし Claude Code が使う API の細部（streaming のイベント形・thinking・`cache_control`）を自前で追い続けることになり、同梱した Ollama より保守が重い。**同梱 Ollama で始め、Ollama の同梱が重荷になったときに agent-mlx のエンジンを土台に置き換える**。agent-mlx は「dev-agent が実機で確かめる対象」の 1 つとして扱う |
| **Apple Containerization**（macOS 26。`container` CLI と Swift パッケージ。1.0 は 2026-06） | Linux コンテナを 1 つずつ軽量 VM で動かす。Docker Desktop 不要。Swift パッケージとしてアプリに組み込める | **採用（Linux 向けの道具のサンドボックス）** | Node・Python・Rust のように Linux でも同じに動く道具は、dev-agent が組み込んだ Containerization の VM の中で動かし、worktree だけをマウントする。イメージと VM の中身はアプリのデータ領域に置く。**macOS 26 と Apple Silicon が要る**（M0 で組み込みの可否を確かめる） |
| **Docker Desktop**（導入済み） | Linux コンテナ | **使わない（代替は上記）** | 別途入れて保守する物が 1 つ増える。Apple Containerization が組み込めない事情が出たときの退避先として残す |
| **Claude Code（CLI 本体）** | 頭脳の実行器。native 版は `~/.local/bin` 固定で置き場所を変えられない | **採用（npm 版をデータ領域へ入れる）** | 同梱した Node で `npm install -g --prefix <データ領域>` し、`CLAUDE_CONFIG_DIR` もデータ領域へ向ける。版は dev-agent 側で固定し（`DISABLE_AUTOUPDATER=1`）、更新は dev-agent のリリースで行う。ログイン（claude.ai の OAuth）だけはブラウザで 1 度人が行う |
| **GitHub App（自作・個人所有）** | (1) **webhook**: PR・ラベル・コメント・push のイベントを、App に設定した 1 つの URL へ署名付き（HMAC-SHA256）で送る。(2) **Checks API**: check run（PR の Checks タブと状態欄に出る。`output.text` に長文、annotations）を作れるのは **GitHub App だけ**（公式: "To create a check run, you must use a GitHub App"） | **採用（気付き・結果・直しのブランチの push）** | 利用者が自分の GitHub App「DevAgent」を 1 度作り、5 リポジトリに入れる。秘密鍵と webhook secret は dev-agent のキーチェーンへ。dev-agent は JWT → installation token を自分で発行する（Security フレームワークの RSA 署名。依存は増えない）。投稿者が `devagent[bot]` になるので、人の発言と機械の出力が見分けられる。クラウドセッションの PR 購読は check suite の失敗と成功のまとめを配信するので、**結果が出た瞬間にクラウドが起きる**。webhook の届け先は次項 |
| **webhook の中継（smee.io）** | GitHub（probot）が運用する webhook の中継。送られた webhook を、ランダムな URL のチャネルに**外向きの SSE で購読している側へ**そのまま（ヘッダごと）流す。Mac に口を開けない。無料・設定ゼロ | **採用（リアルタイムに気付く経路。真実は API で読み直す）** | App の webhook URL を smee のチャネルにし、dev-agent が SSE で待ち受ける。**中継が運ぶのは「合図」だけ**で、dev-agent は合図を受けたら GitHub API で PR・ラベル・head を読み直してから動く（中継が偽物を流しても、無駄な API 呼び出しが 1 回起きるだけ）。加えて webhook の署名を App の secret で検証する。届かない・途切れることはあるので、60 秒の ETag 付きポーリングを**取りこぼしの保険**として残す。中継を自前にしたくなったら Cloudflare Workers 等に置き換えられる（dev-agent 側は URL を変えるだけ） |
| **Tailscale / Cloudflare Tunnel / ngrok** | クラウドから Mac へ届く口を作る | **使わない** | クラウドセッションの egress は許可リスト制で、しかも Mac に口を開ける必要が無い。連絡は GitHub（ラベル・コメント・リリース）と Remote Control の外向き接続で足りる |

### 3.2 既製で賄えないもの＝dev-agent が担うもの

上の表から、残るのは次の 3 つだけで、**dev-agent はこの 3 つに絞って薄く作る**。

1. **アプリ固有のビルド・入れ替え・起動・観測（アダプタ）。** 「PR の head を SDK 付きで
   差分ビルドし、Plug-Ins に置き、Vectorworks を起動し、取り込みを走らせ、本文を回収する」は
   どの既製品も知らない。しかも種類ごとに違う（第 4.5 節）。
2. **リポジトリごとの期待する動作（マニフェスト）。** 何を入れ替え、どう起動し、何をもって
   合格とし、どこまで直してよいかの宣言と、それをローカルエージェントへ渡す仕組み。
3. **往復の進行と結果の投稿。** 合図の受け取り・周の管理・停止・check run の投稿・
   資格情報の保管。今は 4 つのリポジトリが別々に持っているものを 1 か所にする。

4. **環境の抱え込み。** 道具・SDK・モデル・Claude Code をアプリのデータ領域に揃え、
   消せば戻る形に保つ（第 4.10 節）。既製のパッケージマネージャ（Homebrew 等）は
   「Mac 全体を変える」ので使わない。

逆に、**「考える」部分は一切作らない**。頭脳は Claude Code（Remote Control／`claude -p`）
と Ollama で、dev-agent は材料を揃えて渡し、結果を回収するだけにする。

## 4. 全体像

### 4.1 役割分担

```
        ┌──────────────────── GitHub（唯一の連絡路）────────────────────┐
        │ PR（ラベル・push）  check run（実機の結果）  レビュー  CI（ビルド・配布）│
        └──▲────────────┬───────────────▲───────────────────▲───────────┘
           │依頼        │webhook（合図）  │結果               │push（直しのブランチ）
           │（ラベル）   │ ↓ 中継（smee）  │（check run）      │
           │            ▼ SSE で受ける    │                   │
   ┌───────┴──────────────┐      ┌───────┴───────────────────┴───────────────────┐
   │ クラウドエージェント   │      │ Mac                                            │
   │ Claude Code クラウド   │      │  dev-agent（常駐。判断する LLM を持たない）     │
   │ セッション／ルーティン │      │   ├ Watcher    : 合図を受け、API で読み直す     │
   │  - 設計・実装・PR      │      │   │              （保険に 60 秒のポーリング）     │
   │  - 依頼を書く          │      │   ├ Manifest   : repo の dev-agent.yaml を読む  │
   │  - 結果を読んで次へ    │      │   ├ Builder    : worktree で差分ビルド           │
   └──────────────────────┘      │   ├ Adapters   : vw-plugin / mac-app / web …    │
                                  │   ├ Rounds     : ビルド→入れ替え→走らせる→回収  │
   ┌──────────────────────┐      │   ├ Reporter   : check run（GitHub App）         │
   │ 人（スマホ・ブラウザ） │      │   └ AgentBridge: 下の 2 つへ材料を渡す           │
   │  Remote Control で     ├──────┼─▶ Claude Code（人の対話用。常駐しない）         │
   │  Mac の Claude を使う  │      │ ├▶ Claude Code + 同梱 Ollama（ローカル LLM。既定）│
   └──────────────────────┘      │ └▶ Claude Code（Anthropic。昇格したときだけ）     │
                                  │                                                │
                                  │ 道具: VW SDK / Xcode / CMake / Node / Python   │
                                  │ 対象: Vectorworks 2026 / Photogrammetry.app /  │
                                  │       MLXChat.app / iPhone / ブラウザ          │
                                  └────────────────────────────────────────────────┘
```

- **連絡路は GitHub だけ。** クラウド→ローカルの「依頼」は PR のラベル（と人が読む
  コメント）、ローカル→クラウドの「結果」は head の check run（自作 GitHub App）。
  気付きは App の webhook を中継（smee.io）経由で SSE で受けるので**数秒**で、Mac に口は
  開けない。中継が運ぶのは合図だけで、真実は必ず GitHub API で読み直す。クラウド
  セッションは PR を購読しているので、check run が結論に達すれば起きる。
- **Remote Control は人の窓。** 人が外出先から Mac の Claude Code を使うためのもので、
  エージェント同士の経路にはしない（第 4.6 節）。
- **dev-agent は LLM を持たない。** 見張る・ビルドする・入れ替える・走らせる・回収する・
  投稿する・直しを push するは全部決定的な処理で、テストできる。考える必要があるときだけ AgentBridge が Claude Code を
  起こす。

### 4.2 1 周の流れ（往復の単位）

```
 1. クラウドエージェントが PR を作る／push する（CI は従来どおり並行して走る）
 2. クラウドエージェントが PR にラベル `dev-agent:verify` を付ける（特に見てほしいことが
      あれば `@dev-agent` で始まるコメントを人が読む文章で書く）
 3. ラベル付け／push／`@dev-agent` コメントの webhook が中継を通って数秒で届く。dev-agent は
      合図を受けて GitHub API で PR・ラベル・head を読み直す（届かなくても 60 秒のポーリングが拾う）。
      **その 1 回の応答を周の固定値（RoundPin）として凍らせる**（head の sha・作者・ラベル・
      既定ブランチの sha。下の「周の固定値」）。この周の中ではブランチ名を使わない
 4. dev-agent が PR 用の worktree を**固定した sha に detached で合わせ**（`git fetch <URL> <sha>`）、
      その sha の dev-agent.yaml と既定ブランチの固定した sha の dev-agent.policy.yaml を
      `git show` で読み（第 4.4 節）、
      差分ビルド（Builder）→ 入れ替え →（要れば再起動）→ 起動 → 走らせる → 本文・ログ・画面を回収
      ビルドが失敗したら、そこで 1 周を終えてその失敗を結果にする
      決まった区切りで PR を読み直し、head が動いていたらその周を打ち切る（下の「周の途中で
      head が動いたら」）
 5. 結果を固定した sha の check run に書く（名前 `DevAgent / <kind> (macOS)`。本文はアプリが作ったもの）
      ビルド開始で in_progress、終わりで success / failure / neutral
 6. 失敗していて、マニフェストが直しを許していれば AgentBridge が Claude Code を起こす
      - 同じ worktree で、失敗ログ＋期待＋許した範囲を渡す。GitHub の資格情報は渡さず、
        囲いから github.com へも出られない（第 4.7 節）。エージェントは worktree を編集するだけ
      - エージェントが終わったら、dev-agent が変更を周の始めの sha の上の 1 つの commit に
        まとめて push 用のリポジトリへ取り込み、そこで変えたパスがすべて書いてよいパスに
        収まっているかを `git diff` で決定的に確かめる（第 4.7 節）。外れていたら変更を捨てて
        failure で止まる
      - 直したら push せずにまず 4 を手元で回し直す（ビルド → 入れ替え → 走らせる）
      - 手元で通ったものだけを、**push の直前に PR を読み直して head が固定した sha のままで
        あることを確かめてから**、PR のブランチではなく**直しのブランチ**
        `dev-agent/pr-<N>/r<round>` へ push し、head の check run に「直しを用意した」と書く
        （周数と時間に上限）
      - 直せなければ、試したことを check run の summary に書いて failure で止まる
 7. クラウドエージェントが check run の失敗（または成功のまとめ）で起き、`get_check_run` で
      結果を読んで次の手を決める（直しのブランチを取り込む・設計変更・人へ質問・マージ待ち）。
      直しを取り込めば PR の head が動き、CI（最終の門）と次の周が走る。周ごとに起こさず、
      **周が結論に達したときだけ**起きる
 8. 人は PR と dev-agent の画面で経過を見て、「確認できた」を宣言する／止める
```

**2 の依頼が無くても、head が動けば 4〜5 は回る**（今の Vectorworks の往復は「dev
プレリリースが出たら」だったが、これを「push されたら」に早める）。依頼コメントは
「今回は特にここを見て」「ここまでは直してよい」を足すためのもの。CI の dev プレリリースは
往復の起点ではなくなり、**別の機械へ配るための成果物**になる（`artifact.source:
github-release` を選べば従来どおり CI の成果物で回すこともできる）。

#### 周の固定値（確かめた head と、ビルドする head を揃える）

PR・ラベル・作者を API で読んだ時点と、worktree でビルドする時点の間に push が入りうる。
ブランチ名で取り寄せると、**確かめていないコミットをビルドし、その結果を確かめた sha の
check run に書く**ことになる（直しの起点・`compare`・ビルドを飛ばす判定もずれる）。
そこで周の始めに 1 度だけ読み、それを周の中の唯一の出どころにする。

| 項目 | 決めごと |
| --- | --- |
| 読み方 | `GET /repos/{o}/{r}/pulls/{N}` の **1 回の応答**から `head.sha`・`user.login`・`head.repo.full_name`・`labels` を取る（1 つの応答なので互いに食い違わない）。既定ブランチの sha（歯止めの出どころ）も同じ周の始めに固定する。依頼コメントは別の呼び出しになるので、使ったコメントの id を記録する |
| 記録 | `state/<repo>/<pr>.json` に周ごとに残し、check run の `summary` に `head=<sha>`・`policy=main@<sha7>` を書く |
| 取り寄せ | 明示した URL で sha を取る: `git fetch --no-tags https://github.com/<o>/<r>.git <head_sha>` → `git cat-file -e <head_sha>^{commit}` → `git checkout --detach <head_sha>`。`origin` は使わない（worktree の `.git/config` は信用しない。第 4.7 節 7 項と同じ理屈）。worktree にローカルのブランチを持たせない |
| 予備の道 | sha で取れないときは `refs/pull/<N>/head` を dev-agent 専用の ref（`refs/dev-agent/pr-<N>`）へ取り、`rev-parse` が `head_sha` と一致することを確かめてから使う。一致しなければ head が動いたものとして API を読み直す。どちらを主にするかは M0(m) で決める（既定は sha で直接） |
| 取れなかったら | API を読み直す。head が変わっていれば新しい周へ移る。変わっていないのに取れなければ `neutral`（取り寄せ失敗）で閉じる |
| 確かめ方の読み方 | `dev-agent.yaml` は `git show <head_sha>:dev-agent.yaml` で読む。worktree のファイルは前の周のエージェントが書き換えている可能性があるので読まない |
| 使う先 | check run の作成・PATCH・`annotations` の `head_sha`、直しの commit の親、直しのブランチの `base=`・`compare=`・trailer `Dev-Agent-Base:`。**すべて固定値から組み立て、ブランチ名から sha を引き直さない** |

#### 周の途中で head が動いたら

周の中の**決まった区切り**で PR を API で読み直し、固定値と比べる。合図（`synchronize`）や
ポーリングで動いたと知ったときも、次の区切りで止める（走っているビルドやアプリを途中で
殺さない）。

| 区切り | 読み直す |
| --- | --- |
| ビルドの前 | ○ |
| 入れ替え・再起動の前 | ○ |
| 走らせた後、結果を書く前 | ○ |
| エージェントを起こす前 | ○ |
| エージェントが終わった後 | ○ |
| **直しのブランチへ push する直前** | **必ず**（合図が無くても API で読む） |

| 読み直した結果 | 振る舞い |
| --- | --- |
| `head.sha` が固定値と違う | 固定した sha の check run を **`neutral`** で閉じる（title `round N: head が <sha7> へ進んだので打ち切り`、`summary` に `superseded_by=<sha>`）。エージェントの変更と直しの commit は捨て、**push しない**（古い起点の直しは取り込みの判断材料として古く、「同じ head への直しは 1 本まで」とも合わない）。新しい head で周を回し直す（周の番号は進める） |
| `dev-agent:verify` / `dev-agent:fix` が外れた、または `dev-agent:stop` が付いた | `dev-agent:stop` と同じ扱い（`neutral` で結果だけ書いて止まる） |
| `dev-agent:fix` だけが外れた | 確かめる段は続け、直しの段には入らない |
| 変わっていない | そのまま続ける |

古い sha の結果は「古い sha について」は正しいが、待つ価値が無い（Vectorworks の 1 周は
10〜15 分）ので、打ち切って新しい head を早く確かめるほうを選ぶ。立て続けの push を
まとめる規則とリポジトリ間の順番は issue #19 で決める。この判定（固定値と読み直した PR を
比べて「続ける／打ち切る（理由）／確かめるだけ」を返す）は Core の純ロジックにし、
`swift test` で押さえる。

### 4.3 連絡の書式（GitHub 上の約束）

**コメント欄は人（とクラウドエージェント）のもの、check run は機械のもの**、と分ける。
dev-agent は PR にコメントを書かない。

#### クラウド → ローカル: ラベル（＋任意の依頼コメント）

付けられるのは write 権限のある人＝本人かその代理のクラウドセッションだけなので、
これが安全弁を兼ねる。

| ラベル | 意味 |
| --- | --- |
| `dev-agent:verify` | この PR の head を実機で確かめて結果を返す（head が動くたび） |
| `dev-agent:fix` | `verify` に加えて、マニフェストで許した範囲の直しを試みてよい |
| `dev-agent:stop` | この PR の往復を止める（走っている周は安全な区切りで打ち切り、`neutral` で結果だけ書く） |

「今回は特にここを見て」「ここまでは直してよい」は、クラウドエージェントが**人が読む
文章として** PR にコメントする（機械可読な印は要らない。dev-agent はラベルの付いた
PR の最新のコメントのうち `@dev-agent` で始まるものを依頼として読む）。**公開
リポジトリでは誰でもコメントできる**ので、読むのは**作者が本人（クラウドセッションが
本人名義で書いたものを含む）のコメントだけ**で、他人の `@dev-agent` は無視する
（ラベルは write 権限が無いと付けられないので門になっているが、コメントにはその門が
無い）。緊急停止もラベルで行う（コメントの `control=stop` は廃止）。

#### 気付き方: webhook → 中継 → SSE（保険にポーリング）

| 経路 | 役目 | 決めごと |
| --- | --- | --- |
| GitHub App の webhook → smee.io のチャネル → dev-agent が SSE で購読 | **数秒で気付く** | 購読するイベントは `pull_request`（labeled / unlabeled / synchronize / reopened / closed）・`issue_comment`（created）・`release`（published。CI の成果物で回すとき）。dev-agent は署名（`X-Hub-Signature-256`）を App の secret で検証し、**本文は使わず、合図として PR 番号だけ取り出して API で読み直す**。中継の URL はキーチェーン |
| 60 秒ごとの `If-None-Match`（ETag）付きポーリング | **取りこぼしの保険** | 304 は上限に数えられない。5 リポジトリでも認証付きの 5,000 回/時に遠く届かない。中継が落ちていても往復は止まらず、遅くなるだけ |

中継は「公開リポジトリの公開イベントを、公開の中継サービスで Mac へ流す」だけなので、
秘密は通らない。偽の合図が来ても API で読み直すので動作は変わらず、署名の検証で
そもそも捨てる。**「中継を信頼しない」とは、中継に判断の材料を置かないという意味**で、
指示（ラベル・依頼コメント・head・作者）は必ず API から読む。だから中継をセルフホストに
しても設計は変わらない（公開 URL を持つ以上、偽の POST は来るので署名の検証が本体。
webhook 自体が順不同・重複・取りこぼしを起こすので、読み直しも残す）。セルフホストで
得られるのは盗み見の排除と可用性の自前管理だが、公開リポジトリでは前者の価値が無く、
後者はサーバの保守と引き換えになるため、smee.io で始める。

#### ローカル → クラウド: check run（GitHub App として。自前で投稿する）

head の sha ごとに、アダプタ 1 つにつき check run を 1 つ作る。sha は周の固定値（第 4.2 節）の
`head_sha` で、作成・更新・`annotations` の追記のどれも、ブランチ名から引き直さない。

| 項目 | 値 |
| --- | --- |
| `name` | **`DevAgent / <kind> (macOS)`**。例: `DevAgent / vectorworks-plugin (macOS)` |
| `status` / `conclusion` | ビルド開始で `in_progress`、終わりで `success`（期待どおり）／`failure`（ビルド失敗・期待と不一致・落ちた）／`neutral`（道具が無い・停止で中断・head が進んで古くなった・sha を取り寄せられなかった）／`skipped`（本人の PR でない等、対象外） |
| `output.title` | 1 行の結論。例: `round 3: 柱 120/120・耐力壁 36/36・注意 0`、`ビルド失敗（clang: 2 errors）` |
| `output.summary` | 周の表（周・build・source=local／ci・所要・ビルド・入れ替え・実行・判定）と、`expect` の各項目の○×。周の固定値の `head=<sha>`・`policy=main@<sha7>`。打ち切ったときは `superseded_by=<sha>` |
| `output.text` | **アプリが作った本文をそのまま**（Vectorworks の往復なら今の `## 実機フィードバック …` 以下、プローブなら `## 実機プローブ …` 以下）。診断ログの全文を含む。上限（65,535 文字）に収まるよう古いほうから削る |
| `annotations` | ビルドエラー・lint をファイルと行に付ける（1 回 50 件まで、超えたら PATCH で追記） |
| `details_url` | dev-agent の GUI でその周を開く URL スキーム（`dev-agent://round/<repo>/<pr>/<N>`） |

- **同じ head で周を重ねる**（手元の直しを試しているとき）は、同じ check run を PATCH で
  更新し、`output.text` の先頭に最新の周、以下に前の周を残す。
- **直しを用意したら**、head の check run は `failure` のまま、`title` を
  「round N: 失敗 → 直しを `dev-agent/pr-<N>/r<round>` に用意（手元の 1 周は通過）」にし、
  `summary` に下の「直しのブランチ」の決まった行を書く。
- **直せずに止まった**ときは `failure` にし、`summary` に試したこと・残っている失敗・
  人かクラウドに頼みたいことを書く。これが従来の `gave-up` コメントの代わり。
- **画面**（スクリーンショット）は check run の `images` に載せられるが、公開 URL が要る。
  私有リポジトリでは載せられないので、画面は dev-agent のデータ領域に保存して GUI で
  見せ、check run には枚数とパスだけ書く（公開リポジトリでも載せない。他人に見える）。
- **プローブの結果が issue 宛て**（PR の無い main のプローブ）のときだけ、check run を
  main の head に作ったうえで、issue にも 1 行（check run への link）をコメントする。
  これが唯一のコメントで、理由は「issue からは check run が見えない」ため。

- **required checks には入れない**（決定。Mac が落ちていると全 PR が止まるため）。
  `draw/` を含む PR をユーザーの「確認できた」まで待つ規約は、人の運用のまま。
- **セルフホストランナーは使わない。** 全リポジトリが公開で、GitHub 自身が公開
  リポジトリでのセルフホストランナーを勧めていない（PR を出せる誰もが Mac で任意の
  コードを走らせる口になりうる）。上の経路は、GitHub から Mac へ**入るものが「合図」
  だけ**で、実行の判断（本人の PR か・ラベルがあるか・head は何か）を dev-agent が API で
  読み直して行う点が違う。

#### ローカル → クラウド: 直しのブランチ（取り込むかはクラウドが決める）

ローカルの直しは **PR のブランチへ直接入れず**、dev-agent が別のブランチへ push する。
理由は 3 つ。(1) エージェントに push の資格情報を持たせずに済む（第 4.7 節）。(2) PR の
ブランチへ push するのはクラウド（と人）だけになり、ローカルとクラウドが同じブランチへ
同時に push して衝突することが構造的に無くなる。(3) 7〜9B 級のローカル LLM が書いた
直しは、クラウド（か人）が差分を読んでから PR に入る。要らなければ放っておけばよい。

| 項目 | 決めごと |
| --- | --- |
| 名前 | **`dev-agent/pr-<N>/r<round>`**。`round` はその PR の中で通した周の番号で、名前が重ならない。`dev-agent/pr-<N>` という名前そのものは作らない（git の ref が衝突するため） |
| 起点 | 周の始めに固定した PR の head の sha（周の固定値。第 4.2 節）。直しのブランチは必ずその**直上に 1 commit だけ**置く（第 4.7 節）。push の直前に PR を読み直し、head がこの sha から動いていたら push しない |
| 1 本まで | 同じ head への直しは 1 本まで。クラウドが取り込むか head が動くまで、次の直しは作らない（予算の単位は issue #15 で決める） |
| head の check run | `failure` のまま（head そのものは失敗しているので）。`summary` に機械でも読める決まった行を書く: `fix_branch=dev-agent/pr-12/r4` / `base=<sha>` / `fix=<sha>` / `compare=https://github.com/<o>/<r>/compare/<base>...<fix>` / `brain=local\|haiku\|claude`。compare はブランチ名ではなく **sha 同士**で張る（PR のブランチが先へ進んでも、直しの差分だけが見える） |
| 直しの commit の check run | 直しの sha にも `DevAgent / <kind> (macOS)` を `success`（`source=local`）で付ける。クラウドが fast-forward で取り込めば PR の head がこの sha になり、**結果が既にあるので dev-agent はその head をビルドし直さない**（周が 1 つ減る）。cherry-pick で取り込めば sha が変わるので、普通に 1 周回る。ビルドを飛ばすのは、API で読んだ PR の `head.sha` が直しの sha と**完全に一致**し、その sha に **DevAgent App（app id で確かめる）が付けた** `source=local` の `success` があるときだけ（同じ名前の check run を他の App が付けても使わない） |
| クラウドの取り込み方 | 各リポジトリの `CLAUDE.md` に書く。PR の head が `base=` と同じなら `git merge --ff-only <fix>`、違えば cherry-pick（衝突したら直すか、取り込まない）。差分を読んで要らなければ何もしない。直しのブランチを main へ merge しない、そこから PR を作らない |
| 人の取り込み方 | 同じ。compare の URL で差分を見て、手元で取り込む。積み上げの PR（直しのブランチ → PR のブランチ）は作らない（PR が増えるため。要るようになったら足す） |
| CI | **各リポジトリの `build.yml` の push のトリガーから `dev-agent/**` を除く**。App のトークンでの push は workflow を起こすので、除かないと 30 分のビルドが二重に走り、`dev-<slug>` のプレリリースも直しの周ごとに増える。直しは PR へ取り込まれた時点で CI を通る。この変更は `.github/**` なのでエージェントには書けず、そのリポジトリを dev-agent へ移す PR で人かクラウドが入れる |
| 消すとき | 次のどれかに当たったら dev-agent が消す: **取り込まれた**（直しの commit が PR の head の祖先になった、または `git cherry` で patch-id が一致した）／**同じ PR の新しい直しのブランチに置き換えられた**／**PR が閉じた**（merge を含む）。どれにも当たらない間は、**何日経っても消さない**（PR が開いている間は判断待ちで、期限で消すと後で使えたはずの直しを失う）。イベントの取りこぼし（Mac が止まっている間に PR が閉じた等）は、起動時と 1 時間ごとに `dev-agent/**` を一覧して上の条件を API で判定し直して拾う。クラウドが手直しして取り込んだため patch-id が一致しないものは、PR が閉じたときに消える |

#### クラウド側の読み方

クラウドセッションは `get_check_run` で `output` を読む。各リポジトリの
`docs/DEVELOPMENT.md`「届いたコメントの読み方」は、dev-agent へ移るときに「check run の
読み方」へ書き換える（本文の形式は変えないので、節の中身はほぼそのまま）。PR 購読は
check suite の失敗と成功のまとめを配信するので、**周の途中では起きず、結論が出たときに
起きる**。これが G7（使用量）にも効く。`summary` に `fix_branch=` があれば、上の
「直しのブランチ」の取り込み方に従って差分を読み、取り込むかを決める（取り込みの判断で
起きるのは、この結論の 1 回に含まれる）。

### 4.4 マニフェスト（リポジトリごとの「期待する動作」）

リポジトリの直下に置き、**コードと一緒に版管理する**。**読む場所が違うので、2 つの
ファイルに分ける。**「確かめ方」は PR が変えてよいので head から、「歯止め」は PR が
変えてはならないので既定ブランチから読む。

| ファイル | 中身 | 読む場所 | 理由 |
| --- | --- | --- | --- |
| **`dev-agent.yaml`**（確かめ方） | `version` / `build` / `round` / `expect` / `artifact` / `context` | **PR の head** | PR がビルド手順や試験を変えたら、その PR で効かないと困る。CI が走らせる `build.yml` と同じ信頼水準（本人の PR だけ。第 4.7 節）で、ビルドと実行は囲いの中（第 4.10 節） |
| **`dev-agent.policy.yaml`**（歯止め） | `version` / `name` / `kind` / `platforms` / `install` / `agent` | **既定ブランチ** | `agent`（`brain`・`escalate`・`allowed`・`paths`・`budget`）を head から読むと、エージェントが直しの中で書き換え、次の周から自分の制約を外せる。`install` の置き場所（`plugins_dir` など）は**囲いの外への書き込み**で、Seatbelt の例外もここから作るので、head から読むと `~/Library/LaunchAgents` のような場所へ置かせられる。`kind` 等はどのアダプタを動かすかの選択 |

1 つのファイルを節ごとに読み分ける形も考えたが、分けるほうを採る。

- **「どこから読まれるか」がファイル名で分かる。** PR の diff で `dev-agent.policy.yaml`
  が出てきたら、それは「merge するまで効かない、人が見るべき変更」だと一目で分かる。
  1 ファイルだと、同じ diff の中に「すぐ効く行」と「merge 後に効く行」が混ざる。
- **規則が「ファイル単位」で済む。** dev-agent の読み方は「`dev-agent.yaml` は head、
  `dev-agent.policy.yaml` は既定ブランチ」の 2 行で、節の合成が要らない。常に書き換えを
  禁じるパス（第 4.7 節）もファイル名で書ける。
- **取り違えをスキーマで弾ける。** `dev-agent.yaml` に `agent` や `install` を書いたら
  （歯止めを head に置こうとしたら）、黙って無視せずスキーマのエラーにする。逆も同じ。

決めごと（読み分け）:

- **どちらも周の固定値の sha から `git show <sha>:<path>` で読む**（第 4.2 節）。
  `dev-agent.yaml` は固定した head の sha から、`dev-agent.policy.yaml` は固定した既定
  ブランチの sha から。worktree のファイルもブランチ名も使わない（worktree は前の周の
  エージェントが書き換えている可能性があり、ブランチ名は読んだ後に動きうる）。

- **「既定ブランチ」は PR の `base.ref` ではなく、リポジトリの `default_branch`** を
  GitHub API で読んだもの（`main`）。PR を積み重ねると base は別の作業ブランチになり、
  そこはクラウドが自由に push できるので歯止めの根拠にならない。周の始めにその sha を
  固定して読み、check run の `summary` に「歯止めの出どころ: `main@<sha7>`」と書く。
- **歯止めの信頼の根は「既定ブランチへは人が見て merge したものしか入らない」**こと。
  ルールセットで既定ブランチへの直接 push を禁じ、PR 経由に限る（M1 で 5 リポジトリに
  設定する）。`dev-agent.policy.yaml` や `.claude/` を緩める変更は、PR の diff として
  人の目を通る。
- PR が `dev-agent.policy.yaml` を変えていても、**効くのは merge した後**。dev-agent は
  head と既定ブランチで中身が違えば、`summary` に「この PR の `dev-agent.policy.yaml` の
  変更は merge 後に効く」と 1 行書く（気付かずに「設定したのに効かない」と迷わないため）。
  head 側の `dev-agent.policy.yaml` も**スキーマの検証だけはする**（typo は PR のうちに
  知らせる）。
- **既定ブランチにまだ `dev-agent.policy.yaml` が無い**（導入する PR そのもの）ときは、
  最も狭い既定値で補う: `agent.brain: none`（直さない）、`install` はアダプタが
  dev-agent の管理下に決めた置き場（`~/Applications/dev-agent/<name>/` など）に限り、
  外の置き場が要るアダプタ（`vectorworks-plugin`）は `neutral` で「merge 後に回る」と返す。
  `kind` も無いので、head の `dev-agent.policy.yaml` の `kind` を**アダプタの選択にだけ**
  使う（置き場所と直しの範囲には使わない）。

`dev-agent.yaml`（確かめ方。PR の head から読む）:

```yaml
version: 1

artifact:
  source: local-build                       # 往復は手元のビルドで回す（既定）
  fallback: github-release                  # 手元でビルドできないとき（道具が無い等）は CI の成果物
  release:                                  # github-release のときの探し方（CI の公開形式に合わせる）
    dev_tag: "dev-{slug}"                   # slug は build.yml と同じ変換（tr '/:@ ' '----' …）
    stable_tag: stable
    notes: key-value                        # channel= / branch= / commit= / built=
    assets:
      macos-arm64: "min-nano_structureDev.vwlibrary.zip"

build:                                      # 手元の差分ビルド（CI の build.yml と同じ入口を使う）
  requires:                                 # 無ければ dev-agent doctor が指摘し、fallback へ
    - env: VW_SDK_DIR                       # SDKLib を含むフォルダ
    - tool: cmake
  configure: >-                             # 初回と CMakeLists が変わったときだけ
    cmake -S . -B {build_dir} -DVW_SDK_DIR=$VW_SDK_DIR
    -DVW_BUILD_CHANNEL=dev -DVW_BUILD_BRANCH={branch} -DVW_BUILD_VERSION={sha7}
  command: cmake --build {build_dir} --config Release --parallel
  output: "{build_dir}/min-nano_structureDev.vwlibrary"   # install へ渡す成果物
  cache: per-pr                             # build_dir を PR ごとに保つ（差分ビルドのため）
  timeout: 30m

round:
  trigger: [new-head, request]              # PR の head が動いた／依頼コメントが来たら 1 周
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
  report: check-run                         # 名前は dev-agent / <kind> (macOS)

expect:                                     # 人とエージェントが読む「合格の定義」
  - 全フィクスチャで取り込みが例外なく終わり、`結果:` が成功になる
  - 要素の内訳で「描けた」が「命令」と一致する
  - 「注意（描画側の異常）」の節が空である
  - 前の周と比べて要素数が減っていない（減るのは依頼コメントが予告したときだけ）

context:                                    # エージェントへ渡す追加の読み物
  - CLAUDE.md
  - docs/DEV-NOTES.md#実機確認の作法
```

`dev-agent.policy.yaml`（歯止め。既定ブランチから読む）:

```yaml
version: 1
name: min-nano_structure                    # 表示名。check run の見出しに使う
kind: vectorworks-plugin                    # アダプタの種類（第 4.5 節）
platforms: [macos-arm64]                    # 今はこれだけ。windows は将来

install:                                    # 囲いの外へ置く場所（Seatbelt の例外もここから作る）
  adapter: vectorworks-plugin
  app: "Vectorworks 2026"
  plugins_dir: "~/Library/Application Support/Vectorworks/2026/Plug-Ins"
  name: min-nano_structureDev
  installer: vw-install.sh                  # zip 直下。--machine --from <dir> --name <name> --plugins-dir <dir>
  restart_when: shell-id-changed            # installed-shell= と Info.plist の VWShellId を比べる

agent:                                      # 直してよい範囲
  brain: local                              # local（同梱 Ollama。既定）| claude（Anthropic）| none
  local_model: auto                         # auto = doctor がメモリから選ぶ。名指しもできる
  escalate: local -> claude:haiku -> claude # 手元の 1 周が通らなければ次へ（周数の範囲で）
  allowed: [build-error, test-failure, lint, expect-mismatch]
  paths:                                    # 書いてよいパス。既定は拒否（allow に無いものは全部だめ）
    allow:
      - "src/**"
      - "tests/**"
    deny:                                   # allow の中から外すもの（殻・境界）
      - "src/Extensions/**"                 # 殻（再起動を強いる）
      - "src/Payload*"
  budget: { rounds: 3, minutes: 60 }
  # push 先は選べない。直しは常に dev-agent/pr-<N>/r<round> へ（第 4.3 節）
```

決めごと:

- **`expect` は自然言語でよい。** 機械で判定できるものはアダプタが `result=` に畳み、
  判定できないもの（絵が崩れていないか）は人が見る。`expect` はエージェントと人の両方が
  読む「合格の定義」で、**依頼コメントの個別の指示より弱い**（依頼が勝つ）。
- **`agent.allowed` に無い失敗は直さない。** 設計判断・SDK の未知の挙動・殻の変更は
  クラウドと人に返す。
- **書いてよいパスは許可制で、既定は拒否。** `agent.paths.allow` に当たるパスだけを
  書いてよく、書いていないものは全部だめ。`paths.deny` は allow の中から殻・境界を外す
  ためのもので、各リポジトリの既存の歯止めを写す。優先は「dev-agent が常に禁じるパス
  （第 4.7 節）＞ `deny` ＞ `allow` ＞ 既定の拒否」で、`allow: ["**"]` と書いても常時禁止は
  開かない。否定のパターン `!…` はスキーマで拒む。禁止制にしないのは、新しく足した
  ファイルや書き漏らしたパスが黙って「直してよい」側に入るのを避けるため（インストーラ
  の `scripts/vw-*` は、禁止に書かなくても allow に無いので触れない）。
- **`paths.allow` が無い・空なら直しは行わない**（`brain: none` と同じ。確かめるだけ
  回し、`summary` に「書いてよいパスが無いので直さなかった」と書く）。
- **`brain` の既定は `local`。** ただし `allowed` の種類ごとに M0 の成績で「最初から
  `claude`」にできる（`agent.brain_by_kind: {expect-mismatch: claude}` のように）。
  ローカルで通らなければ `escalate` の順に昇格する。
- **`build` は CI と同じ入口を呼ぶ。** `configure` / `command` に書くのは、各リポジトリの
  `build.yml` が呼んでいるものと同じ（`cmake` / `scripts/xcode-build.sh` /
  `swift build` ＋ `package-app.sh`）。dev-agent 専用のビルド手順を作ると「手元では
  通るが CI で落ちる」が増えるので作らない。刻印（コミット・ブランチ・チャンネル）も
  CI と同じ変数で渡し、`channel` は `dev` にする（アプリ側のアップデータが残っている間は
  それが「別のビルド」と誤認しないよう、移行期間は自動確認を切る。第 4.9 節）。
- **スキーマの検証は dev-agent が無 SDK・無ネットワークで行い**、エラーは check run に
  載せる（マニフェストの typo で黙って何もしない、を避ける）。

### 4.5 アダプタ（インフラの種類ごとの違い）

| kind | 手元のビルド | 入れ替え | 起動と実行 | 回収 | 固有の制約 |
| --- | --- | --- | --- | --- | --- |
| `vectorworks-plugin` | `cmake`（`VW_SDK_DIR`）。殻・本体・`.vwr` を 1 回で。差分ビルドなら本体だけの変更は数分 | ビルド出力を `vw-install.sh --machine --from`（既存）で配置。殻 ID（`VWShellId`）が変わったら Vectorworks を終了・再起動（AppleScript `quit` → `open -a`）。**再起動を外から行えるので、今の「切り離したヘルパーから終了要求」は要らなくなる** | 本体の `runTestRound` を**外から**呼ぶ。経路は既存の MCP ブリッジのスプール（`$TMPDIR/min-nano_structure-mcp/<id>.req.json`）に `vw_run_test` を足す（第 5.1 節）。公式文書の `Vectorworks -t <Test> -ab -l out.txt`（自動テストの CLI）は未確認なので、SDK リファレンスで issue を立てて確かめる | 本体が作る結果本文（今と同じ）・診断ログ・`screencapture` | GUI アプリ。ログイン中のユーザーセッションが要る。未署名プラグインの警告を 1 度許可しておく。1 周 10〜15 分 |
| `probe-plugin` | `cmake -S plugin`（同じ SDK）。PR のプローブは `gather-probes.sh --prs <N>` で集めてから（ローカルビルドの ID は `local` になる） | ビルド出力を置く。CI の成果物で回すときは転がりタグ `probes` の zip（`build=` で新旧を比べる。既存の `vw-probes-update.sh do-install`） | ピッカーの「一覧を順に実行」を外から呼ぶ（同じスプール方式を足す） | 既存の結果本文（`<!-- vw-probes-result … -->`） | 同上。`recovered=yes`（VW ごと落ちた）の拾い直しは本体に残す |
| `mac-app` | photogrammetry: `swift build -c release --arch arm64` → `package-app.sh`。agent-mlx: `generate-xcodeproj.sh` → `xcode-build.sh MLXChat-macOS …`（MLX の初回は 30 分超、以後は `DERIVED_DATA` を PR ごとに保って差分） | `.app` を隔離解除・アドホック署名して `~/Applications/dev-agent/<name>/` に置く（本番の `/Applications` には触らない） | 同梱 CLI（`photogrammetry-cli` / `mlxchat-cli`）を直接実行する。GUI の確認が要るときだけ `open` ＋ URL スキーム | stdout（`HelperProtocol` の行、`bench` のレポート）・終了コード・画面 | RealityKit / Metal は実機でしか動かない。`bench` は固定幅テキストなので、JSON 出力を足すのは各アプリ側の小さな宿題 |
| `ios-app` | `run-ios.sh --build-only`（署名付き。チームとデバイスは手元の設定） | `run-ios.sh` 相当で転送・起動（`devicectl`） | 起動確認と URL スキーム。結果の回収経路が無いので、まず「起動して落ちない」まで | `devicectl` のログ・画面 | 証明書とつないだ iPhone が要る。最初は手動の補助に留める |
| `web` | `core/build.sh`（wasm）→ `npm run build`。API は手元の uvicorn（`static-channels/development`） | 無し（手元の dev サーバを立てる。CI のプレビュー URL で回すこともできる） | ブラウザで開いて煙試験（Playwright。Clerk の開発インスタンスのログイン状態を保存しておく） | 画面・コンソール・`/api/healthz` | **公開リポジトリ**なので、他人の PR では絶対に動かさない（第 4.7 節） |

アダプタは「ビルドする・入れ替える・起動する・走らせる・回収する」の 5 つの操作に揃え、
dev-agent の本体は種類を知らない（G4）。**ビルドの道具は dev-agent が揃える**（Node・
CMake・uv/Python・Rust・Vectorworks SDK。第 4.10 節）。Xcode と Metal ツールチェーンだけは
外にあるものを使い、無ければ `dev-agent doctor` が名指しして `artifact.fallback` へ倒す。新しい種類は、アダプタ 1 つとマニフェストの `kind` を足すだけ。

### 4.6 ローカルエージェント（頭脳）の使い分け

**既定はローカル LLM で、Anthropic の Claude は昇格したときだけ。** Max プランの使用量は
クラウドセッションと `claude -p` と Remote Control で共有されるので、往復の定型処理に
Claude を使うと、肝心の設計・実装（クラウド）の分が減る。

| 段 | 頭脳 | 向くもの | 起こし方 | 使用量 |
| --- | --- | --- | --- | --- |
| local（既定） | **ローカル LLM（`claude -p` ＋ 同梱 Ollama）** | 定型で量の多いもの（下の表） | `ANTHROPIC_BASE_URL=http://127.0.0.1:<専用ポート> ANTHROPIC_AUTH_TOKEN=ollama`。Ollama は使うときだけ起動し、終わったら降ろす（メモリのため） | 0 |
| claude | **Claude（`claude -p`）** | local が外した・local に向かないもの。実機の失敗ログから原因を当てて直す | AgentBridge が `escalate` に従って起こす。まず `--model haiku`、それでも駄目なら既定のモデル | 消費する |

#### ローカル LLM に任せる仕事（候補。M0 で測って付録 B に確定する）

| 仕事 | 入力 | 出力 | 見込み |
| --- | --- | --- | --- |
| 失敗の分類 | ビルド／テスト／実機のログ | `build-error` / `test-failure` / `lint` / `expect-mismatch` / `crash` / `unknown` | 高い（分類は小さいモデルでも安定する） |
| 結果の要約 | アプリが作った本文・診断ログ | check run の `title` と `summary`（人とクラウドが最初に読む 10 行） | 高い。**クラウドが読むトークンを減らす**効果が大きい |
| lint・書式の直し | lint の出力 | 修正 commit | 高い |
| 単純なコンパイルエラー | clang / swiftc のエラー 1〜3 件 | 修正 commit | 中。型の取り違え・include 漏れ・引数の数など |
| テストの期待値合わせ | 失敗したアサーションと差分 | 修正 commit | 中。**値を合わせるだけの変更は危険**なので、`expect` に「期待値の変更は依頼があるときだけ」と書けるようにする |
| `expect` との突き合わせ | 本文と `expect` の各項目 | ○× と根拠の行 | 中〜高 |
| 実機の失敗の原因当て | 診断ログ・差分 | 修正 commit | 低い。B へ昇格する前提 |

- **同じハーネスで走らせる**ことが要点。`CLAUDE.md`・`.claude/settings.json` の
  permissions・hooks・skills は両方で効く。各リポジトリにある「殻を触らない」
  「`sleep` で待たない」といった規約が、ローカル LLM にもそのまま掛かる。
- **ただしハーネスは既定ブランチのものを使い、worktree（head）のものを読ませない。**
  `.claude/settings.json` の hooks は Mac 上で任意のコマンドを走らせ、permissions と
  sandbox の設定は Claude Code 自身の囲いを外せる。head から読むと、PR（やエージェント
  自身の前の周の直し）が `claude -p` を起こした時点で効いてしまう。そこで AgentBridge は
  1. 既定ブランチの sha から `git show` で `.claude/settings.json`・`CLAUDE.md` を取り出し、
  2. dev-agent の固定の方針（sandbox 有効・`bypassPermissions` 禁止・`Edit` / `Write` は
     `paths.allow` にだけ allow、`paths.deny` と常時禁止は deny）を**上に重ねて**データ領域に
     1 つの設定ファイルを作り（Bash 経由の書き込みまでは縛れないので、これは手前の
     歯止めで、最終の判定は第 4.7 節の `git diff`）、
  3. `claude -p` には worktree の設定を自動で読ませず（`--setting-sources` から `project`
     と `local` を外す）、作った設定を `--settings` で、`CLAUDE.md` を
     `--append-system-prompt` で渡す。MCP は `--strict-mcp-config` で dev-agent が
     渡したものだけにし、worktree の `.mcp.json` を読ませない。
  これらのフラグが head の設定・hooks・MCP・skills を本当に読まないかは **M0(k) で
  確かめる**。確かめられなかった種類のファイルについては、「PR がそれを変えていたら、
  その PR ではエージェントを起こさない（確かめるだけ回す）」に倒す。`CLAUDE.md` は
  読み物で歯止めではない（歯止めは dev-agent の決定的な検査が担う）ので、下位の
  ディレクトリの `CLAUDE.md` が自動で読まれても穴にはならない。人が起こす `dev-agent rc`
  も同じ組み立てで起こし、head のハーネスを使いたいときは人が明示する。
- **ローカル LLM に渡す材料は dev-agent が絞る。** 失敗ログの該当部分・`expect`・
  `allowed`・直してよいファイルだけを渡す。全文脈を読ませない（小さいモデルは文脈で
  迷う）。`--bare` で `CLAUDE.md` 以外の自動読み込みを切り、`--append-system-prompt` で
  要るものだけ足す。
- **昇格の規則は 1 か所**（`agent.escalate`）。local が「直せた」と言っても、手元の 1 周
  （ビルド → 入れ替え → 走らせる）が通らなければ直せていない。通らなかったら claude へ。
  claude でも通らなければ失敗で止まる。
- **push 前に手元で 1 周回す。** 通ったものだけ、直しのブランチへ push する（第 4.3 節）。
  取り込むかはクラウドが決め、CI（30 分のビルド）は取り込んだ後の最終の門に残る。
- **予算で止まる。** `budget.rounds` と `budget.minutes` を超えたら `failure` にして
  クラウドと人へ返す。無限に直し続けない。

#### Remote Control の位置付け（エージェント間の経路にはしない）

Remote Control は「Mac で動く Claude Code のセッションを、claude.ai やスマホから人が
操作する窓」である。技術的にはクラウドセッションからも話しかけられるが、**この設計では
クラウドエージェントがローカルへ依頼する経路には使わない**。理由は 3 つ。

1. **記録が PR に残らない。** 往復の記録は check run に集めると決めた。
   Remote Control 経由の依頼はセッションの中だけで完結し、人もクラウドも後から追えない。
2. **マニフェストの歯止めが効かない。** `allowed` / `paths` / `budget` は
   dev-agent の AgentBridge が掛けるもので、Remote Control のセッションは素の Claude Code
   として何でもできる。クラウドが「速いから」とこちらを選ぶと、歯止めが抜ける。
3. **使用量を消費する。** Remote Control は常に Anthropic の Claude で、ローカル LLM に
   逃がせない（`ANTHROPIC_BASE_URL` を向け替えると Remote Control が使えない）。

そこで次を決めごとにする。

| 誰が | Remote Control をどう使うか |
| --- | --- |
| **人** | 外出先から Mac の Claude Code を使う窓。`dev-agent rc`（人が明示的に起こす）で、PR の worktree を開いたセッションを立てる。その中で Claude は `dev-agent run` / `dev-agent build` を**道具として**呼べる。「絵を見ながら直す」はここで人が行う |
| **クラウドエージェント** | **使わない。** ローカルへの依頼はラベルだけ。各リポジトリの `CLAUDE.md` に「ローカルへの依頼は `dev-agent:*` のラベルで行い、他のセッションへ `send_message` しない」と書く（規約で縛る。クラウドセッションが選べる道具を機械的に封じる手段は無いので、**dev-agent が Remote Control のサーバを常駐させない**＝話しかける相手を置かない、で実効性を持たせる） |
| **dev-agent** | 起こさない。`dev-agent rc` のときだけ、人の求めで `claude remote-control --spawn worktree` を擬似端末で起動し、人が閉じれば終わる |

#### 16 GB の Mac での現実

- 使える予算は agent-mlx の `DeviceProfile` と同じ規則で **物理メモリ × 0.70 ≒ 11 GB**。
  `qwen3-coder`（30B-A3B、4bit で約 19 GB）と `gpt-oss:20b`（約 14 GB）は**入らない**。
  現実的なのは **7〜9B 級の 4bit**（`qwen2.5-coder:7b` 約 4.7 GB、`qwen3:8b` 約 5.2 GB。
  いずれも【推定】）で、12B 級（`gemma3:12b` 約 8 GB）は KV キャッシュと合わせて限界。
  Claude Code のシステムプロンプトと `CLAUDE.md` だけで数万トークンになるので、
  `num_ctx` は 32k 以上が要り、KV キャッシュは `OLLAMA_KV_CACHE_TYPE=q8_0` で半分にする。
- **同時に動かさない。** ビルド（Xcode は数 GB）・Vectorworks・agent-mlx の実機確認・
  ローカル LLM は、dev-agent が**一度に 1 つ**だけ動かす。LLM は呼び出しごとに読み込み、
  終わったら降ろす（`keep_alive=0`。7B 級なら読み込みは数秒）。Containerization の VM は
  必要なときだけ起こし、メモリは 2 GB に抑える。
- **だから代用できる範囲は狭めに見積もる。** 上の表の「高い」から始め、M0 の計測で
  「中」を 1 つずつ確かめて付録 B に書く。モデルの候補と成績は `toolchains.yaml` と
  付録 B で版管理し、メモリを増やした Mac に入れ替えたら `doctor` が自動で大きい
  モデルを選ぶ。

### 4.7 安全弁

1. **対象は本人の PR だけ。** PR の作者が自分（またはクラウドセッションが自分の名義で
   作ったもの）で、fork からでないものに限る。**全リポジトリが公開**なので、この判定は
   webhook の本文を信じず、dev-agent が GitHub API で読み直した PR の `user` と
   `head.repo` で行う。ラベルは write 権限が無いと付けられないので二重の門になる。
   他人の PR は、ラベルが付いていても（付けられないはずだが）動かさない。
2. **マニフェストに書けるコマンドは、アダプタの操作とリポジトリ内のスクリプトだけ。**
   任意のシェルは書けない（CI の `build.yml` と同じ信頼水準に揃える）。
3. **歯止めは PR から変えられない。** `dev-agent.policy.yaml`（`agent` / `install` 等）と
   エージェントのハーネス（`.claude/`・`CLAUDE.md`）は、PR の head ではなく既定ブランチ
   から読む（第 4.4・4.6 節）。PR の head は「確かめる対象」であって、「確かめ方の
   歯止め」を決める側ではない。
4. **エージェントは書いてよいパスにしか触れない。それを dev-agent が決定的に確かめる。**
   書いてよいパスは**許可制で、既定は拒否**。既定ブランチの `agent.paths.allow` に当たり、
   `agent.paths.deny` にも、dev-agent が常に禁じるパス（下の表）にも当たらないものだけ。
   常時禁止はマニフェストで開けられない（`allow: ["**"]` でも開かない）。

   | 常に禁じるパス | 理由 |
   | --- | --- |
   | `dev-agent.yaml`・`dev-agent.policy.yaml` | 確かめ方（`build` / `round`）を自分で変えて「通った」ことにさせない。歯止めは既定ブランチから読むので書き換えても効かないが、PR に混ぜて merge させる道も塞ぐ |
   | `.claude/**`・`**/.claude/**`・`CLAUDE.md`・`**/CLAUDE.md`・`CLAUDE.local.md`・`.mcp.json` | ハーネス。次に起こす Claude Code（昇格先を含む）の作法と権限を変えさせない |
   | `.github/**` | CI と配布。直しの push で workflow を書き換えさせない |
   | `.gitmodules` | 取り寄せ先を差し替えさせない |

   判定は LLM に頼らず、エージェントが終わるたびに（手元の 1 周を回し直す前に）
   dev-agent が行う。
   - **エージェントは commit しない（しても使わない）。** エージェントが終わったら、
     dev-agent が worktree の中身（commit 済み・未 commit・未追跡のすべて）から tree を作り、
     周の始めの sha（PR の head）を親にした **1 つの commit** として、エージェントの囲いの
     外にある push 用のリポジトリ（第 7 項）へ取り込む。author と committer は
     `DevAgent[bot]`、trailer に `Dev-Agent-Round:`・`Dev-Agent-Base:`・`Dev-Agent-Brain:`
     （`local` / `haiku` / `claude`）を付ける。エージェントの commit をそのまま使わないのは、
     author を人の名前に偽れること、merge commit や複数の commit が混ざることを避けるため。
   - 判定は**その push 用のリポジトリの中で**、周の始めの sha と上の commit を比べて行う。
     判定したものと push するものが同じ commit になり、判定と push の間に worktree を
     すり替えられても効かない。`git diff --name-only
     --no-renames -z` を使う（改名検出があると、`.github/x.yml` を `x.yml` へ移した
     ときに新しい名前しか出ない。`-z` は改行を含む名前のため）。変更・追加・削除の
     **すべてのパスが**書いてよいパスに収まっていなければならない（改名は削除＋追加と
     して両方を見る。新しいファイルも allow に当たらなければだめ）。
   - パターンの照合は**大文字小文字を区別しない**（APFS は既定で区別しないので、
     `.CLAUDE/settings.json` は次の checkout で `.claude/settings.json` として読まれる）。
     Unicode も NFC に揃えてから比べる。照合は Core の純ロジックにし、`swift test` で
     こうした抜け道を押さえる。
   - **シンボリックリンクとサブモジュールの追加・変更も止める**（`git diff --raw` で
     mode `120000` と `160000`）。リンク越しに worktree の外や禁止のパスへ書かせないため、
     また `.gitmodules` を禁じても gitlink だけで別の commit を指させないため。
   - 外れていたら、その周の変更を捨てて（worktree を周の始めの sha に戻す）`failure` に
     し、`summary` に外れたパスを書く。**昇格はしない**（直せなかったのではなく、
     範囲を破ったので）。
   - dev-agent 自身が worktree で `git` を呼ぶときは `-c core.hooksPath=/dev/null
     -c core.fsmonitor=false` を付け、`GIT_CONFIG_NOSYSTEM=1` にする。エージェントが
     `.git/config` や `.git/hooks` を書き換えて、囲いの外の dev-agent に実行させる道を
     塞ぐ（あわせて第 3 層の囲いでエージェントの書き込みからそれらを外す。第 4.10 節）。
5. **消すコードは増やさない。** 入れ替えで消してよいのは、dev-agent が自分で置いた
   `~/Applications/dev-agent/` 配下と、各アプリのアンインストーラが消すと決めている
   範囲だけ（Vectorworks は既存の `vw-uninstall.sh` の歯止めをそのまま使う）。
6. **止める手段を 2 つ持つ。** ラベル `dev-agent:stop`（PR 単位）と、dev-agent の
   画面／CLI の「すべて止める」（Mac 単位）。
7. **GitHub の資格情報は dev-agent だけが持ち、push も dev-agent が行う。** 自作の
   GitHub App「DevAgent」の秘密鍵と webhook secret を macOS キーチェーン（service
   `dev-agent`）に。権限は Checks（Read/Write）・Pull requests（Read。ラベルとコメントを
   読む）・Issues（Write。プローブの issue 宛ての 1 行だけ）・**Contents（Read/Write。
   Releases の取得と、直しのブランチの push・削除）**・Metadata。Workflows の権限は
   付けないので、`.github/workflows/` を変える push は GitHub 側でも拒まれる（常時禁止との
   二重の門）。installation token は短命で、dev-agent が都度発行する。**各アプリから
   トークンの扱いが消える。**
   - **push はデータ領域の push 用のリポジトリ**（`state/push/<repo>.git`。bare。エージェントの
     囲いからは見えも書けもしない）から行う。worktree から `git push origin` はしない。
     worktree の `.git/config` はエージェントが手を入れた可能性があり、`remote.origin.url`
     や `url.<x>.insteadOf` を書き換えられていると、トークンが別のホストへ送られるため。
   - push は**エージェントのプロセスが残っていないとき**だけ、明示した URL と refspec で
     行う: `git push https://github.com/<o>/<r>.git <fix>:refs/heads/dev-agent/pr-<N>/r<round>`。
     トークンは `GIT_ASKPASS` で渡し、ディスクにも引数にも残さない。
   - **書いてよいブランチを GitHub 側では絞らない**（決定。App の Contents: write は
     全ブランチに効き、ルールセットで絞ると人・クラウド・CI の push まで例外の一覧で
     管理することになるため）。既定ブランチだけは第 7 節 #19 のルールセットで守られ、DevAgent は
     その例外に入れない。それ以外のブランチは dev-agent のコードで守る: 書き先の
     refspec は `refs/heads/dev-agent/` で始まるものしか組み立てない、**`--force` を一切
     使わない**（GitHub は早送りにならない push を拒むので、不具合が起きても他のブランチの
     commit は消えず、余計な commit が載るまでに留まる）、ブランチの削除は `dev-agent/`
     で始まる名前にだけ行う。これらは Core の純ロジックにして `swift test` で押さえる。
   - Contents: write は Releases と tags も書ける。トークンは dev-agent の外へ出ず、
     dev-agent はそれらを書くコードを持たないので、残りのリスクとして受け入れる。
8. **秘密をエージェントに渡さず、手も届かせない。** AgentBridge はトークン・鍵を
   エージェントの環境変数にもプロンプトにも渡さない。それだけでは同じユーザーで動く
   Bash から人の資格情報（Xcode の git が既定で使う `credential.helper=osxkeychain`、
   `~/.config/gh`、`~/.ssh` と `SSH_AUTH_SOCK`、`~/.netrc`、`~/.git-credentials`）に届くので、
   **エージェントの囲いから github.com・api.github.com へは出られない**ようにする
   （Claude Code のサンドボックスのネットワーク許可を、同梱 Ollama の 127.0.0.1 と
   api.anthropic.com だけにする）。あわせて Seatbelt で上のファイルとキーチェーン
   （`com.apple.SecurityServer`）を拒み、`HOME` をデータ領域へ向け、`SSH_AUTH_SOCK` を外し、
   `GIT_CONFIG_NOSYSTEM=1`・`GIT_CONFIG_GLOBAL=/dev/null` で起こす（第 4.10 節）。
   資格情報を見つけられても、push する道が無い。
9. **ビルドと実行は囲いの中で。** Linux で動く道具は Containerization の VM（worktree だけ
   マウント）、Mac でしか動かないもの（Xcode・Vectorworks SDK・Vectorworks 本体）は
   Seatbelt（`sandbox-exec`）で「worktree とデータ領域以外へ書けない」プロファイルの下で
   動かす。エージェントの Bash は Claude Code 自身のサンドボックス設定を有効にして走らせる
   （第 4.10 節）。
10. **Mac 全体を変えない。** `/usr/local`・Homebrew・`~/.local`・シェルの rc には触れない。
    dev-agent が置くものはデータ領域と launchd の plist 1 つだけで、`dev-agent uninstall` が
    全部消す。
11. **確かめた sha だけをビルドし、その sha に結果を付ける。** 1 項（本人の PR か）と
    ラベルの判定は、周の始めの 1 回の API の応答で行い、その応答の `head.sha` を周の固定値に
    する（第 4.2 節）。取り寄せは明示した URL と sha で行い、ブランチ名を使わない。
    worktree は detached で、確かめ方も歯止めも固定した sha から `git show` で読む。
    決まった区切りで読み直し、head やラベルが変わっていれば打ち切って `neutral` にする。
    **直しの push の直前は必ず読み直し**、head が動いていれば push しない。Builder は
    sha しか受け取らない形にし、ブランチ名で checkout する道を作らない。

### 4.8 dev-agent の作り

- **言語と形**: Swift（SwiftPM）。姉妹リポジトリ（photogrammetry / agent-mlx）と同じ
  「純ロジックの Core ＋ 薄い CLI ＋ 薄い GUI」の三分割にし、同じ CI・リリース・自動
  アップデートの仕組みを移植する（**dev-agent 自身の更新だけは自分で行う**。それが
  このアプリの存在理由なので例外にする。第 4.11 節）。
  - `Sources/DevAgentCore/` … マニフェストの解釈・Releases の解釈・周の状態機械・
    コメントの組み立て・エージェントへ渡す材料の組み立て・マニフェストの読み分け（head の
    `dev-agent.yaml` と既定ブランチの `dev-agent.policy.yaml`）・書いてよいパスの照合（第 4.7 節）・
    周の固定値（`RoundPin`）と、読み直した PR と比べて「続ける／打ち切る（理由）／確かめるだけ」を
    返す判定（`HeadGuard`。第 4.2 節）。**ネットワークもプロセス起動も
    しない**純ロジックで、`swift test` で押さえる。
  - `Sources/DevAgentAdapters/` … ビルド・入れ替え・起動・回収の実装（`Process` /
    `FileManager` / `screencapture`）。判断を置かない。
  - `Sources/DevAgentBuilder/` … worktree の管理（`git worktree add` / 明示した URL と sha での
    fetch / detached checkout。**引数に sha しか受け取らない**。第 4.7 節 11 項）と
    `build` 節の実行。PR ごとに `~/Library/Caches/dev-agent/<repo>/pr-<N>/{src,build}` を
    保ち、差分ビルドを効かせる。PR が閉じたら消す。エージェントの後の worktree から直しの
    commit を作り、push 用のリポジトリへ取り込む（第 4.7 節）。
  - `Sources/DevAgentToolchains/` … 道具の保管庫（第 4.10 節）。`toolchains.yaml`（版・URL・
    SHA-256・同梱か取り寄せか）を読み、同梱物の展開・取り寄せ・検証・環境変数の組み立て。
  - `Sources/DevAgentSandbox/` … Containerization の VM の起動とマウント、Seatbelt の
    プロファイル生成。判断を置かない。
  - `Sources/DevAgentGitHub/` … GitHub App の JWT（RS256。Security フレームワーク）と
    installation token、check run の作成と更新、PR・ラベル・コメント・Releases の読み取り
    （ETag 付き）、webhook の署名検証、直しのブランチの push と削除。判断を置かない。
    check run の本文の組み立て、push の refspec の組み立て（`dev-agent/` 以外を作らない）、
    直しのブランチを消すかの判定（第 4.3 節）は Core（純ロジック）。
  - `Sources/DevAgentWatch/` … 中継（smee.io）への SSE 購読と、保険のポーリングの時計。
    受けた合図を「この PR を読み直せ」に落とすだけで、判断を置かない。
  - `Sources/dev-agent/` … CLI。`setup`（道具とモデルを揃え、App の鍵と中継を設定する）／
    `doctor`（足りない物の名指し）／`watch`（常駐。合図を受けて `run` を回す）／
    `run --repo --pr [--sha]`（1 周だけ。`--sha` を渡すと、API の head と違えば拒む）／`build --repo --pr`／`install --repo --build`／
    `rc`（人のための Remote Control）／`stop`（Mac 全体）／`status`／`manifest check`／
    `uninstall`（全部消す）。launchd で `watch` を常駐。
  - `Apps/DevAgentApp/` … メニューバーアプリ。一覧（repo × PR × 周）・ログ・停止・
    「このビルドを入れる」。判断を持たない。
- **既存コードの流用**: `UpdateFeed`（Releases → チャンネル）はそのまま Core へ移す。
  Vectorworks の `vw-install.sh` / `vw-uninstall.sh` / `vw-probes-update.sh` は zip の
  直下にあるので、dev-agent はそれを呼ぶだけ（配置の知識を dev-agent に持ち込まない。
  これは Vectorworks 側の「同梱スクリプトに配置を書かない」規約と同じ理屈）。
- **マニフェストの形式**: YAML（決定）。SwiftPM の依存は **Yams と Apple の
  Containerization の 2 つだけ**を許す。姉妹アプリの「外部依存ゼロ」はそのままで、
  dev-agent だけが持つ。
- **状態の置き場**: `~/Library/Application Support/dev-agent/state/<repo>/<pr>.json`
  （周・周ごとの固定値（head の sha・作者・ラベル・歯止めの sha・使った依頼コメントの id）・
  最後に見た head・最後に投稿した時刻・予算の消費・直しのブランチの一覧）。
  push 用のリポジトリは `state/push/<repo>.git`。ビルドの中間物は
  `~/Library/Caches/dev-agent/`（消えても作り直せるもの）。Vectorworks の
  `feedback.txt` と同じ役目をここへ移す。
- **気付き方**: App の webhook → smee.io → SSE（数秒）＋ 60 秒の ETag 付きポーリング
  （保険）。第 4.3 節。

### 4.9 各アプリに何が残り、何が消えるか

| リポジトリ | 消える（dev-agent へ） | 残る（外から呼べる入口） | 殻／本体への影響 |
| --- | --- | --- | --- |
| Vectorworks プラグイン | `src/Updater*`・`src/FeedbackLoop*`・`ExtFeedbackPalette`・`scripts/vw-update.*`・`vw-feedback.*`・`vw-token.*`・`core/FeedbackSession` の周の記憶・「アップデータを確認」メニュー | `draw/Feedback` の `runTestRound`（本文の生成・図面の戻し）と、それを外から呼ぶスプールの口（`vw_run_test`）。`vw-install.*` / `vw-uninstall.*` は配布 zip に残る | **殻が小さくなる**＝`VW_SHELL_INPUTS` が減り、再起動を要する変更が減る。ABI は口が減る方向で版を上げる |
| SDK リファレンス（VwSdkProbes） | 殻の更新処理・`vw-probes-update.*`・`vw-probes-feedback.*`・トークン | プローブの実行・結果本文・`recovered=yes` の控え。ピッカーは残す（人が手で走らせる道は残す） | 同上 |
| photogrammetry / agent-mlx | `Sources/*Updater/`・設定画面の更新タブ・`install-update.sh` | CLI と URL スキーム（既にある）。`bench` に `--json` を足す | アプリから Updater ターゲットが消える。`UpdateFeed` は dev-agent 側へ移植 |
| portal | 無し（プレビューは CI が作る） | プレビュー URL のコメント（既にある）。手元の dev サーバの起動手順は README のまま | 変更なし |

**CI からは何も消えない。** `build.yml` のビルド・tidy・テスト・dev プレリリースはそのまま。
変わるのは「往復がそれを待たない」ことだけ。**クラウド側の読み方の規約**（各リポジトリの
`docs/DEVELOPMENT.md`「届いたコメントの読み方」と `CLAUDE.md` の該当節）は、そのリポジトリが
dev-agent へ移る PR で「DevAgent の check run の読み方」に書き換える。

**順序は「dev-agent が同じことをできるようになってから消す」。** 消すのは M5（第 6 節）で、
それまでは両方が動く期間を置く（dev-agent が入れ替えたビルドを、アプリ側のアップデータが
「古い」と誤認して戻さないよう、移行期間はアプリ側の自動確認を切る）。

### 4.10 環境の抱え込み（同梱・取り寄せ・サンドボックス・片付け）

狙いは 2 つ。**Mac を入れ替えても dev-agent を入れれば環境が揃う**こと、**データ領域を
消せば元に戻る**こと。そのために「何が外にあり、何をアプリが抱えるか」を先に決める。

#### 外にあるもの（dev-agent は触らない）

| もの | なぜ外か | dev-agent がすること |
| --- | --- | --- |
| macOS 26 以降・Apple Silicon（現状: **macOS 27・16 GB**） | Containerization の前提 | `doctor` が版を見て、26 未満なら Linux 向けの道具をホストで動かす（囲いは Seatbelt だけ）。メモリは第 4.6 節の予算に使う |
| Xcode（Metal ツールチェーン込み） | 10 GB 超で再配布できない | `doctor` が `xcode-select` と Metal ツールチェーンの有無を見る。足りなければ `xcodebuild -downloadComponent MetalToolchain` を提案する |
| Vectorworks 2026 本体 | 確かめる対象そのもの | 起動・終了だけ行う |
| claude.ai のログイン | OAuth はブラウザで人が行う | 初回に `claude auth login` を開く。資格情報はキーチェーン |
| GitHub App「DevAgent」 | 人が 1 度作り、5 リポジトリに入れる（Checks と Contents の write、webhook が要る） | 初回に App ID・秘密鍵（.pem）・webhook secret を受け取りキーチェーンへ。webhook URL は `setup` が作った smee のチャネル。M5 で GitHub の manifest flow（1 クリックで App を作る）に置き換える |
| iPhone の署名証明書（任意） | Apple Developer のもの | `run-ios.sh` がキーチェーンから引く |

#### アプリが抱えるもの（データ領域）

```
~/Library/Application Support/dev-agent/
  toolchains/         node/ cmake/ uv/ python/ rustup/ cargo/ vw-sdk/<版>/ claude/（npm の prefix）
  models/             同梱 Ollama のモデル（OLLAMA_MODELS）
  containers/         Containerization のイメージと VM の rootfs
  claude-config/      CLAUDE_CONFIG_DIR（settings・セッション・記憶）
  state/              <repo>/<pr>.json（周・周の固定値・予算・直しのブランチ）
                      push/<repo>.git（直しを検査して push する bare リポジトリ。エージェントの囲いの外）
  toolchains.lock     実際に展開した版と SHA-256
~/Library/Caches/dev-agent/
  <repo>/pr-<N>/{src,build}   worktree と差分ビルドの中間物
  spm/ derived-data/ npm/ cargo-target/
~/Library/Logs/dev-agent/
~/Applications/dev-agent/<name>/      確かめるために入れた .app
~/Library/LaunchAgents/jp.min-nano.dev-agent.plist
キーチェーン: dev-agent（GitHub トークン）、Claude Code の資格情報
```

**dev-agent が環境変数で全部をここへ向ける**: `PATH`（toolchains を先頭に）・
`CARGO_HOME` / `RUSTUP_HOME`・`UV_PYTHON_INSTALL_DIR` / `UV_CACHE_DIR`・`npm_config_prefix` /
`npm_config_cache`・`CLAUDE_CONFIG_DIR`・`OLLAMA_MODELS` / `OLLAMA_HOST`・`VW_SDK_DIR`・
`DERIVED_DATA` / `SPM_DIR`（姉妹アプリの `xcode-build.sh` が読む）。各リポジトリのビルド
スクリプトは変えない——**環境変数で置き場所を外から決められるようにしてあるものだけを
使う**（無いものはそのリポジトリ側の小さな宿題）。

#### 同梱か取り寄せか（`toolchains.yaml`）

dev-agent の中に 1 つの表 `toolchains.yaml` を持ち、**各道具の版・URL・SHA-256・入れ方**を
固定する。CI は dev-agent をビルドするときにこの表を読んで同梱物を `.app` に入れ、
`setup` は同じ表を読んで取り寄せ物をデータ領域へ入れる。**表が 1 つなので、同梱と
取り寄せの境目を後から動かしても手順は変わらない。**

| 道具 | 入れ方 | 理由 |
| --- | --- | --- |
| Ollama（CLI 版） | **同梱** | MIT。Anthropic 互換 API をこれが担う。版の固定が要る |
| Node（LTS） | **同梱** | MIT。portal・Playwright・Claude Code（npm 版）が要る |
| CMake | **同梱** | BSD-3。Vectorworks プラグインと VwSdkProbes が要る |
| uv | **同梱** | MIT/Apache-2。Python 本体と venv をデータ領域の中に作る |
| Python | 取り寄せ（uv が入れる） | python-build-standalone。macOS 既定の Python は触らない |
| Rust（rustup・`wasm32-unknown-unknown`） | 取り寄せ | 大きい。`RUSTUP_HOME` / `CARGO_HOME` をデータ領域へ |
| Vectorworks SDK | 取り寄せ | 再配布できない。URL は "latest" で中身が黙って変わるので ETag を `toolchains.lock` に記録し、CI（`vw-sdk-cache-key.sh`）と同じ版を使う |
| Claude Code | 取り寄せ（同梱 Node の npm で prefix 指定） | native 版は置き場所を変えられない。版は表で固定し、自動更新を切る |
| ローカル LLM のモデル | 取り寄せ（同梱 Ollama が pull） | 数十 GB。Mac のメモリに合わせて `doctor` が選ぶ |
| Linux のコンテナイメージ | 取り寄せ（Containerization が pull） | Node / Python / Rust の Linux 版。worktree をマウントして使う |
| Playwright のブラウザ | 取り寄せ（Node 側） | `PLAYWRIGHT_BROWSERS_PATH` をデータ領域へ |

**版の更新は dev-agent のリリースで行う。** 週 1 回のワークフローが表の版と SHA-256 を
上げる PR を立てる（SDK リファレンスの `sdk-index` と同じ流儀）。利用者側は dev-agent の
自動アップデートを受けるだけで、道具の版が揃って上がる。

#### 囲い（サンドボックス）の 3 層

| 層 | 何を | どう囲うか |
| --- | --- | --- |
| 1. 置き場所 | 全部 | 上の環境変数で、書き込み先をデータ領域と worktree に限る。これが土台で、残り 2 層が無くても「消せば戻る」は成り立つ |
| 2. Linux 向けの道具 | Node / Python / Rust のビルド・テスト・Playwright | dev-agent が組み込んだ Containerization の VM。worktree と必要なキャッシュだけマウントし、ネットワークは npm / PyPI / crates.io に限る。ホストの `$HOME` は見えない |
| 3. Mac でしか動かないもの | Xcode のビルド・Vectorworks SDK のビルド・Vectorworks 本体・エージェントの `claude -p` | Seatbelt（`sandbox-exec`）のプロファイルで、書き込みを worktree・データ領域・一時ディレクトリに限る。Vectorworks の Plug-Ins フォルダだけ例外で許す（例外のパスは既定ブランチの `dev-agent.policy.yaml` の `install` から作る。第 4.4 節）。`claude -p` は Claude Code 自身のサンドボックス設定（Bash の囲い）も有効にし、そのプロファイルでは worktree の git ディレクトリの `config` と `hooks/` への書き込みを外し、データ領域の `state/`（push 用のリポジトリを含む）は読み書きとも拒む。さらに人の資格情報（`~/.config/gh`・`~/.ssh`・`~/.netrc`・`~/.git-credentials`・キーチェーン）を拒み、ネットワークは同梱 Ollama と api.anthropic.com だけにする（github.com へは出られない。第 4.7 節） |

第 2 層は **macOS 26 でしか使えない**ので、`doctor` が版を見て、26 未満なら第 3 層だけで
動かす（機能は落ちない。囲いが弱くなるだけ）。Docker Desktop はどちらの層にも使わない。

#### 片付け

- `dev-agent uninstall` が、上の「アプリが抱えるもの」を**上から順に全部消す**（データ
  領域・キャッシュ・ログ・`~/Applications/dev-agent/`・launchd の plist・キーチェーンの
  2 項目）。Vectorworks の Plug-Ins に入れた dev ビルドは、各リポジトリのアンインストーラ
  （`vw-uninstall.sh`。フォルダ名が一致し中に殻があるときだけ消す）に任せる。
- 消してよいのは**自分が置いたものだけ**。Xcode・Vectorworks・`~/.ollama`（利用者が別に
  入れていたもの）・`~/.claude`（別の Claude Code）には触れない。
- データ領域を手で丸ごと消しても壊れない（次の起動で `setup` が揃え直す）。

### 4.11 dev-agent 自身の更新（起動時と UI から、自動で）

dev-agent は他のアプリから更新の仕組みを引き上げる当人なので、**自分の更新は自分で、
人の手を介さずに**行う。姉妹アプリの `UpdateFeed` / `UpdaterService` を土台にし、
Vectorworks プラグインの「起動時には確認しない」は踏襲しない（あれは Vectorworks の起動に
乗るのを避けるためで、dev-agent 自身の起動には乗せてよい）。

| 項目 | 決めごと |
| --- | --- |
| 配布 | 姉妹アプリと同じ。CI が main への push で `stable`、PR ごとに `dev-<slug>` のプレリリースを公開する。notes は `channel=` / `branch=` / `commit=` / `built=`。アセットは `DevAgent.app.zip`（CLI と同梱物を内包） |
| 確認のきっかけ | **起動時**（GUI の起動、`watch` の常駐開始）と、**UI の「更新を確認」**（メニューバーのメニューと `dev-agent update`）。加えて常駐中は 6 時間ごとに確認する（ずっと起動したままの Mac で取り残されないため）。確認は Releases API を ETag 付きで読むだけで軽い |
| 新旧の比べ方 | Info.plist の `GitCommit` / `GitBranch` / `BuildChannel` と、選んでいるチャンネル（stable か特定の dev ブランチ）のリリースの `commit=` を比べる。異なれば更新 |
| チャンネル | 既定は `stable`。UI で dev ブランチを選べる（dev-agent 自身の PR を実機で確かめるための道。姉妹アプリの「開発版を選んで更新」と同じ） |
| 入れ替えの手順 | ダウンロード → `ditto` で展開 → 隔離解除とアドホック署名 → **往復が走っていなければ**自分を終了し、同梱の `install-update.sh` が `.app` を置き換えて再起動する。`watch` の常駐（launchd agent）は新しい `.app` で立ち上がる |
| 走っている周との兼ね合い | 周の最中には入れ替えない。更新を見つけたら「次に手が空いたとき」に印を付け、周が終わった直後に入れ替える。UI からの明示の指示でも、周が走っていれば「周の終わりに入れ替える」と答える（途中で止めると check run が `in_progress` のまま残る） |
| 同梱物の更新 | `.app` に同梱した道具（Ollama・Node・CMake・uv）は `.app` ごと置き換わる。取り寄せ物（Claude Code・SDK・モデル）は、新しい `toolchains.yaml` と `toolchains.lock` の差分を次の `setup` が埋める（起動後に自動で走る） |
| 自動で入れるか | **入れる**（確認と入れ替えまで自動）。止めたいときは UI で「自動更新を止める」。止めている間も確認はして、「新しい版がある」とだけ知らせる |
| 失敗したとき | 展開や署名に失敗したら元の `.app` を触らず、次回の確認で再試行する。入れ替え後の起動に失敗したら、`install-update.sh` が直前の `.app` を戻す（1 世代だけ退避しておく） |

M1 から入れる。理由は、dev-agent 自身の開発が「PR → dev ビルド → Mac で確かめる」の往復で
進むので、最初から自動で入れ替わるほうが速いこと（姉妹アプリで同じ仕組みが動いている
ので移植で済む）。

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
- **手元のビルド**は `cmake -S . -B build -DVW_SDK_DIR=… -DVW_BUILD_CHANNEL=dev` の
  既存の手順（`docs/DEVELOPMENT.md`「ローカルビルド」）そのまま。本体だけの変更なら
  差分ビルド＋入れ替えで再起動も要らず、**push から結果まで数分**になる。clang-tidy は
  手元では掛けないので、`source=local` の結果で絵が良くても、マージは CI の緑を待つ。
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

- 最初の実装対象にする（第 6 節 M1）。理由: ビルドの入口が 1 つ（`swift build` ＋
  `package-app.sh` / `xcode-build.sh`）、CLI が揃っている、GUI を介さずに「走らせて回収」
  まで行ける。**photogrammetry から始める**（MLX を引かないので手元のビルドが数十秒で、
  仕組みの往復を速く回せる）。agent-mlx は初回 30 分超のビルドを PR ごとの
  `DERIVED_DATA` で温存し、以後は差分で回す。
- マニフェストの `round.run` は CLI を直接呼ぶ。photogrammetry は固定の写真セットで
  `process`（点群だけなら速い）と `sort`、agent-mlx は小さいモデルで `bench --runs 3`。
- `bench` のレポートは固定幅テキストなので、`--json` を足す（アプリ側の小さな宿題）。
  dev-agent は数値を前の周と比べ、しきい値を超えて遅くなったら `expect-mismatch` にする。
- iOS は「転送して起動して落ちない」まで。結果の回収経路（iPhone → Mac）は無いので、
  当面は人が見る。

### 5.4 portal（Web）

- 手元では `core/build.sh` → uvicorn（:8080）→ `npm run dev`（:5173）を dev-agent が
  立てて煙試験を当てる（README の手順どおり。`.env` と ADC は Mac に置いてあるものを使う）。
  CI のプレビュー URL でも同じ試験を当てられるようにし、本番相当の確認はそちらで行う。
- Playwright で煙試験を書く（Clerk の開発インスタンスのログイン状態を保存して使う）。
  テストそのものは portal のリポジトリに置き（`dev-agent.yaml` の `round.run` から呼ぶ）、
  dev-agent は走らせて結果を回収するだけにする。
- 公開リポジトリなので、第 4.7 節の「本人の PR だけ」を最初から厳守する。

## 6. 進め方（縦切りで 1 周ずつ）

| 段 | 何をするか | 終わりの印 |
| --- | --- | --- |
| **M0 検証スパイク（作る前に確かめる）** | (a) GitHub App の webhook → smee.io → SSE で、ラベル付けから dev-agent が気付くまでの秒数と、1 日の取りこぼし率（ポーリングが拾った件数）を測る。launchd agent から `open -a "Vectorworks 2026"` と `xcodebuild` が**ログイン中のユーザーの画面で**動くか（GUI と Metal が使えるか）も確かめる。(b) **ローカル LLM の代用範囲の計測**: 5 リポジトリの過去の PR から「CI の赤 → 直した commit」の組を 30〜50 件集め、`dev-agent bench-brain` で `claude -p` ＋ Ollama（16 GB に入る 7〜9B 級を 3 つ）に同じ失敗を直させ、第 4.6 節の仕事ごとに成功率・所要・メモリを測る（判定は「手元の 1 周が通るか」）。同じ組を `--model haiku` でも測り、昇格先の目安にする。(c) スプール経由で Vectorworks の本体の関数を外から呼べるか（既存の `vw_call` で確認）。(d) `Vectorworks -t` の実在（SDK リファレンスの issue）。(e) Desktop のローカル・スケジュールタスクで「PR を見に行って結果を返す」を手作業の代わりに 1 周回してみる。(f) 各リポジトリの手元ビルドの所要時間（初回・差分）を実測し、G6 の目標を現実の数字にする。(g) Containerization を SwiftPM のアプリに組み込み、worktree をマウントした VM で `npm run build` が通るか（署名と entitlement の条件も）。(h) npm 版の Claude Code を prefix 指定でデータ領域へ入れ、`CLAUDE_CONFIG_DIR` の下で `claude -p` と Remote Control（擬似端末で起動）が動くか。(i) 同梱した Ollama を `OLLAMA_MODELS` / `OLLAMA_HOST` 指定で起動し、Claude Code から使えるか。(j) 自作 GitHub App の installation token で check run を作り、`output.text` に 60 KB の本文を載せ、クラウドセッションの PR 購読がその失敗で起きるか・`get_check_run` で読めるか。(k) **head のハーネスを読ませない起こし方**: worktree に「起動で印のファイルを作る hook」「`bypassPermissions`」「`.mcp.json` のサーバ」「skill」を仕込み、第 4.6 節の組み立て（`--setting-sources`・`--settings`・`--strict-mcp-config`・`--append-system-prompt`）で `claude -p` を起こして、どれも効かないこと、既定ブランチから渡した hooks と deny は効くことを確かめる。(l) **エージェントが push できないことと、dev-agent の push の道**: 第 4.7 節の囲い（ネットワーク許可・Seatbelt・環境）の下の `claude -p` の Bash から、`git push`（osxkeychain・SSH・`~/.config/gh` のいずれを使っても）・`gh`・`security find-internet-password` が**すべて失敗する**こと。App の installation token（Contents: write、Workflows なし）で `dev-agent/pr-<N>/r<round>` への push と削除ができ、既定ブランチへは拒まれ、`.github/workflows/` を含む push も拒まれること。直しの sha に付けた check run が、fast-forward で取り込んだ後の PR の head でそのまま見え、dev-agent がビルドを飛ばせること。(m) **sha で取り寄せる道**: GitHub に対して `git fetch --no-tags https://github.com/<o>/<r>.git <head_sha>` で PR の head（fork でないもの）が取れるか、force-push で到達できなくなった sha がどう振る舞うか（取れる・取れない・いつまで）、予備の `refs/pull/<N>/head` を取って `rev-parse` で一致を確かめる道が動くか。あわせて、head が進んで `neutral` で閉じた古い sha の check run が、クラウドセッションの PR 購読を起こさないか | 各項目の結果を本書の付録に書く。(k) で効いてしまう種類があれば、第 4.6 節の「PR がそれを変えていたらエージェントを起こさない」をその種類に適用する。(l) でエージェントから push できる道が 1 つでも残れば、塞げるまで M3（直し）に進まない。(b) の成績で、第 4.6 節の表の「見込み」を実測に置き換え、`agent.allowed` ごとの既定（local で始めるか、最初から claude か）を決める。(g) が駄目なら第 2 層を外し、(h) が駄目なら native 版を `~/.local` に置く妥協を第 4.10 節に書く。(m) で sha の直接の取り寄せが通らなければ、`refs/pull/<N>/head` と一致の確認を主の道にする（第 4.2 節）。`neutral` がクラウドを起こすなら、打ち切りの check run の書き方を見直す |
| **M1 dev-agent の骨格＋道具の保管庫＋Builder＋mac-app アダプタ** | Core（マニフェスト・周の状態・コメント）、Toolchains（`toolchains.yaml`・`setup`・`doctor`・環境変数。囲いは第 1 層だけ）、Builder（worktree と差分ビルド）、CLI の `run --repo --pr`。GitHub（App の token・check run・webhook と中継・保険のポーリング）。**dev-agent 自身の自動アップデート**（第 4.11 節。姉妹アプリから移植）。photogrammetry で「push → 合図 → 手元ビルド → 入れ替え → `photogrammetry-cli` → check run」を 1 周 | photogrammetry の PR に `DevAgent / mac-app (macOS)` の check run が人手ゼロで付く。push から結果まで 5 分以内 |
| **M2 Vectorworks アダプタ＋スプールの口** | プラグイン側に `vw_run_test` を足す（本体）。dev-agent 側に vectorworks-plugin のビルド（`cmake` ＋ `VW_SDK_DIR`）・配置・再起動。既存の往復と**並走**させ、同じ結果が返ることを確かめる | 同じ head に対して、プラグイン内の往復（CI のビルド。コメント）と dev-agent の往復（手元のビルド。check run）が同じ本文を出す。push から結果まで 10 分以内 |
| **M3 AgentBridge＋同梱 Ollama** | Claude Code（npm 版）をデータ領域へ。`claude -p` の起動・worktree・材料の絞り込み・`--json-schema` の受け取り・予算。同梱 Ollama の起動・停止とモデルの取り寄せ。M0(b) の成績で「高い」と出た仕事から `local` を既定にし、昇格の規則を入れる | 実機の失敗から dev-agent が直しのブランチへ push した修正を、クラウドが取り込んで CI が緑になる例が 1 つできる。そのうち Anthropic を呼ばずに済んだ割合を付録 B に書く |
| **M4 残りのアダプタ＋囲いの第 2・3 層** | probe-plugin・web・ios-app。Containerization の VM（Node / Python / Rust）と Seatbelt のプロファイル。`watch`（常駐）と launchd | 対象の 5 リポジトリすべてが `dev-agent.yaml` と `dev-agent.policy.yaml` を持つ。portal のビルドとテストがホストに Node を入れずに通る |
| **M5 撤去と GUI と片付け** | 各アプリから Updater／往復の駆動を消す（第 4.9 節）。メニューバーアプリ（更新の確認・チャンネル選択・自動更新の停止を含む）。週 1 回の道具の版上げ PR。`uninstall` | Vectorworks の殻の `VW_SHELL_INPUTS` から `Updater*` と `FeedbackLoop*` が消える。クリーンな Mac に dev-agent を入れて `setup` だけで往復が回り、`uninstall` で残り物が無い |

- 各段は「1 変更＝1 周が回る縦切り」で PR にし、Vectorworks の規約と同じく**実機確認が
  要るものは下書き PR で、ユーザーの「確認できた」を待ってからマージ**する。
- M0 は本書の合意後すぐ着手できる。M1 以降は M0 の結果で順序を入れ替える余地を残す
  （たとえば (a) が通れば、M3 より先に Remote Control での対話的な往復を使い始める）。

## 7. 決まったこと・決めてほしいこと

### 決まったこと（2026-10-01）

| # | 項目 | 決定 |
| --- | --- | --- |
| 1 | プラン | 個人の **Max**。Remote Control・ルーティンは使える。セルフホスト環境は対象外 |
| 3 | マニフェストの形式 | **YAML**（Yams を唯一の依存として許す） |
| 5 | Windows 版 Vectorworks | **当面対象外**（`platforms` で将来足せる形は保つ） |
| 6 | dev-agent の言語 | **Swift**（姉妹アプリと同じ作法・CI を流用） |
| 7 | ビルドの場所 | **往復は手元の差分ビルドで回す**。CI は最終の門と配布に残す（第 1.3・4.2・4.4 節） |
| 2 | ローカル LLM の実行基盤 | **別途インストールせず、アプリで完結させる**。手段は同梱 Ollama（第 3.1・4.10 節）。将来 agent-mlx のエンジンで置き換える余地は残す |
| 8 | 手元の道具 | Xcode はある。Vectorworks SDK・Node は無く、Python は既定のみ。Docker Desktop はあるが使わない。**道具は dev-agent が同梱／取り寄せでデータ領域に揃え、できる範囲で囲う**（第 4.10 節） |
| 9 | 片付け | **データ領域を消せば元に戻る**ことを設計の要件にする（`uninstall`。第 4.10 節） |
| 10 | macOS の版・メモリ | **macOS 27・16 GB**。Containerization は使える。ローカル LLM は 7〜9B 級に絞り、同時に 1 つしか動かさない（第 4.6 節） |
| 4 | 結果の戻し方 | **コメントではなく check run**（自作 GitHub App）。コメント欄に機械的な文を並べない（第 4.3 節） |
| 12 | Claude Code の置き方 | **npm 版をデータ領域へ**（第 3.1・4.10 節） |
| 13 | GitHub App | **作る**（名前 `DevAgent`。check run と webhook の両方に使う） |
| 14 | check の名前 | **`DevAgent / <kind> (macOS)`**。required checks には**入れない** |
| 15 | 気付き方 | セルフホストランナーは**全リポジトリが公開なので使わない**。App の webhook を smee.io で中継して SSE で受け（数秒）、60 秒のポーリングを保険にする。中継は合図だけを運び、真実は API で読み直す（第 4.3 節） |
| 16 | Remote Control の位置付け | **人の窓であって、エージェント間の経路にはしない**。クラウドからローカルへの依頼はラベルだけ。dev-agent は Remote Control のサーバを常駐させず、人が `dev-agent rc` で起こす（第 4.6 節） |
| 17 | webhook の中継 | **smee.io** を使う。自前の中継へは URL を変えるだけで移れる（第 4.3 節） |
| 18 | dev-agent 自身の更新 | **起動時と UI から確認し、自動で入れ替える**（周の最中は周の終わりまで待つ）。M1 から入れる（第 4.11 節） |
| 11 | 頭脳の既定 | **ローカル LLM を既定にし、Claude は昇格したときだけ**。代用できる範囲は M0 の計測で詰め、付録 B で育てる（第 4.6 節・G7） |
| 19 | 歯止めの出どころ（issue #2） | **マニフェストを 2 つに分ける。確かめ方（`build` / `round` / `expect` / `artifact` / `context`）は `dev-agent.yaml` に置いて PR の head から読み、歯止め（`agent` / `install` / `kind` 等）は `dev-agent.policy.yaml` に置いてハーネス（`.claude/`・`CLAUDE.md`）とともに既定ブランチ（`default_branch`。PR の base ではない）から読む**。取り違えた節はスキーマのエラーにする。エージェントが書いてよいパスは**許可制で既定は拒否**（`agent.paths.allow` から `paths.deny` を除いたもの）。両マニフェスト・ハーネス・`.github/**`・`.gitmodules` は dev-agent が常に禁じ、マニフェストでは開けない。判定は `git diff --no-renames` と大文字小文字を区別しない照合で決定的に行う。既定ブランチへの直接 push はルールセットで禁じる（第 4.4・4.6・4.7 節）。push の資格情報と push 先は #20 |
| 20 | 直しの push（issue #3） | **エージェントには GitHub の資格情報を渡さず、囲いから github.com へも出さない。直しは dev-agent が周の始めの sha の上の 1 つの commit（`DevAgent[bot]`）にまとめ、囲いの外の push 用のリポジトリで書いてよいパスを検査し、手元の 1 周が通ったものだけを App の installation token（Contents: write）で直しのブランチ `dev-agent/pr-<N>/r<round>` へ push する**。PR のブランチへは push しない。head の check run は `failure` のまま `fix_branch=` / `base=` / `fix=` / `compare=` を書き、直しの sha にも `success` の check run を付ける。**取り込むかはクラウドが決める**（fast-forward か cherry-pick。人も同じ手順で取り込める）。積み上げの PR は作らない。各リポジトリの `build.yml` は `dev-agent/**` への push で CI を走らせない。直しのブランチは「取り込まれた／新しい直しに置き換えられた／PR が閉じた」ときだけ消し、期限では消さない。書けるブランチを GitHub のルールセットで絞ることはせず、dev-agent のコード（`dev-agent/` 以外の refspec を作らない・`--force` を使わない・`dev-agent/` 以外を消さない）で守る（第 4.2・4.3・4.6・4.7・4.10 節）。予算の単位は issue #15 で決める |
| 21 | 確かめた head とビルドする head（issue #7） | **周の始めに PR を 1 回だけ API で読み、その応答を周の固定値（`RoundPin`。head の sha・作者・ラベル・既定ブランチの sha・使った依頼コメントの id）として凍らせる。ビルド・check run・直しの起点・`compare`・ビルドを飛ばす判定は、すべて固定値の sha を使い、ブランチ名は使わない**。取り寄せは明示した URL で `git fetch <URL> <sha>` → detached checkout（予備に `refs/pull/<N>/head` を取って一致を確かめる道。主と予備は M0(m) で確定）。`dev-agent.yaml` は固定した sha から `git show` で読む。周の中の決まった区切り（ビルド・入れ替え・結果を書く・エージェントを起こす・エージェントの後・**push の直前は必ず**）で読み直し、head が動いていたら固定した sha の check run を `neutral`（`superseded_by=`）で閉じ、直しは push せずに捨て、新しい head で回し直す。ラベルが外れた／`stop` が付いたら `stop` と同じ扱い。判定は Core の純ロジック（第 4.2・4.3・4.4・4.7・4.8 節）。立て続けの push をまとめる規則とリポジトリ間の順番は issue #19 |

### 決めてほしいこと

いまは無い。新しい論点が出たらここに番号を振って足し、決まったら上の表へ移す。

## 8. 用語

| 語 | 意味 |
| --- | --- |
| クラウドエージェント | Claude Code のクラウドセッション／ルーティン。設計・実装・PR 作成を担う |
| ローカルエージェント | Mac で動く Claude Code（Remote Control / `claude -p`）または Claude Code＋Ollama。実機で確かめ、小さく直す |
| dev-agent | 本リポジトリで作る Mac アプリ＋CLI。ビルド・入れ替え・起動・回収・check run を担い、判断する LLM を持たない（同梱 Ollama は道具として起動するだけ） |
| 周（round） | 「ビルド → 入れ替え → 走らせる → check run」の 1 回。既存の `round=N` と同じ |
| アダプタ | インフラの種類（plugin / app / web）ごとのビルド・入れ替え・起動・回収の実装 |
| Builder | PR ごとの worktree を保ち、マニフェストの `build` 節で差分ビルドする dev-agent の部品 |
| マニフェスト | リポジトリ直下の `dev-agent.yaml`（期待する動作。PR の head から読む）と `dev-agent.policy.yaml`（直してよい範囲・置き場所。既定ブランチから読む）の 2 つ |
| 頭脳（brain） | ローカルエージェントが使う LLM。`local`（同梱 Ollama。既定）か `claude`（Anthropic。昇格時） |
| 合図 | App の webhook が中継（smee.io）を通って dev-agent に届くイベント。本文は信じず、PR 番号だけ取り出して API で読み直す |
| check run | dev-agent が GitHub App として PR の head に作る実機の結果。名前 `DevAgent / <kind> (macOS)` |
| 周の固定値（RoundPin） | 周の始めに 1 回だけ API で読んだ PR の head の sha・作者・ラベルと既定ブランチの sha。その周のビルド・check run・直しはすべてこれを使い、ブランチ名を使わない（第 4.2 節） |
| 直しのブランチ | ローカルの直しを載せて dev-agent が push するブランチ `dev-agent/pr-<N>/r<round>`。PR の head の直上に 1 commit だけを置き、取り込むかはクラウドが決める |
| push 用のリポジトリ | データ領域の `state/push/<repo>.git`。dev-agent が直しの commit を作り、検査し、push する bare リポジトリで、エージェントの囲いからは見えない |

## 付録 A. 参照した資料

- Claude Code: [Remote Control](https://code.claude.com/docs/en/remote-control) /
  [Run programmatically（`claude -p`）](https://code.claude.com/docs/en/headless) /
  [Routines](https://code.claude.com/docs/en/routines) /
  [Cloud sessions](https://code.claude.com/docs/en/claude-code-on-the-web) /
  [Self-hosted environments](https://code.claude.com/docs/en/self-hosted-environments) /
  [Desktop scheduled tasks](https://code.claude.com/docs/en/desktop-scheduled-tasks) /
  [Hooks](https://code.claude.com/docs/en/hooks)
- Ollama: [Anthropic compatibility](https://docs.ollama.com/api/anthropic-compatibility) /
  [リポジトリ（MIT）](https://github.com/ollama/ollama)
- Apple: [Containerization](https://opensource.apple.com/projects/containerization/) /
  [apple/container](https://github.com/apple/container)（1.0、macOS 26）
- mlx-swift-lm: [リリース](https://github.com/ml-explore/mlx-swift-lm/releases)（tool calling）
- OpenCode: [Server](https://opencode.ai/docs/server/)
- 既存リポジトリ: `vectorworks-plugin-import-ifc-homeskz/docs/DEVELOPMENT.md`（自動
  アップデート・実機フィードバックの往復・MCP ブリッジ）、
  `vectorworks-developer-sdk-reference/plugin/README.md`（プローブの仕組み）、
  `photogrammetry` / `agent-mlx` の `Sources/*Updater/`・`build.yml`、
  `portal/.github/workflows/preview.yml`

## 付録 B. M0 の結果（未着手）

M0 の各項目の結果をここに書く。確認水準の印は SDK リファレンスの流儀に合わせる
（無印＝実機確認済み・【推定】・【文書根拠】）。

### B.1 ローカル LLM の代用範囲（M0(b)。育てていく表）

| 仕事（第 4.6 節） | モデル | 件数 | 通った | 所要（中央値） | ピークメモリ | 既定にするか |
| --- | --- | --- | --- | --- | --- | --- |
| （未計測） | | | | | | |

### B.2 手元ビルドの所要時間（M0(f)）

| リポジトリ | 初回 | 差分（本体 1 ファイル） | 備考 |
| --- | --- | --- | --- |
| （未計測） | | | |
