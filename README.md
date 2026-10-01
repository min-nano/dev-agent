# dev-agent

クラウドの AI エージェント（Claude Code のクラウドセッション／ルーティン）が書いたコードを、
**ローカルの Mac で入れ替え・走らせ・確かめ・小さく直して結果を PR へ返す**ための実行基盤です。
各アプリ／プラグインが個別に抱えていた「開発版の自動アップデート」と「実機フィードバックの
往復」をここへ引き上げ、アプリ側を軽くします。

- 設計書（草案）: [docs/design.md](docs/design.md) — 背景・既存サービスの評価・全体像・
  マニフェスト・移行計画・決めてほしいこと
- 対象リポジトリ: `vectorworks-plugin-import-ifc-homeskz` / `vectorworks-developer-sdk-reference` /
  `photogrammetry` / `agent-mlx` / `portal`

実装はまだありません。設計書の合意後、M0（検証スパイク）から着手します。
