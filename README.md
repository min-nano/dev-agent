# dev-agent

クラウドの AI エージェント（Claude Code のクラウドセッション／ルーティン）が書いたコードを、
**ローカルの Mac で入れ替え・走らせ・確かめ・小さく直して結果を PR へ返す**ための実行基盤です。
各アプリ／プラグインが個別に抱えていた「開発版の自動アップデート」と「実機フィードバックの
往復」をここへ引き上げ、アプリ側を軽くします。往復のビルドは CI を待たず、
Mac 上の worktree で差分ビルドして回します（CI は最終の門と配布に残します）。
ローカル LLM・ビルドの道具・Claude Code・モデルは dev-agent が同梱または初回に取り寄せて
アプリのデータ領域にだけ置き、Mac を入れ替えても入れ直せば揃い、消せば元に戻るようにします。

- 設計書（方針合意済み）: [docs/design.md](docs/design.md) — 背景・既存サービスの評価・全体像・
  マニフェスト・移行計画・決めてほしいこと
- 対象リポジトリ: `vectorworks-plugin-import-ifc-homeskz` / `vectorworks-developer-sdk-reference` /
  `photogrammetry` / `agent-mlx` / `portal`

実装はまだありません。設計書の第 7 節の論点は決着したので、次は M0（検証スパイク）です。
