# MCP / AI Tooling

AI ツール（MCP クライアント）から各種 MCP サーバーへ接続する手順や設定差分を整理した学習ノートです。

## 含まれるファイル

| ファイル | 内容 |
|----------|------|
| [`adobe-analytics-mcp-gemini-antigravity.html`](adobe-analytics-mcp-gemini-antigravity.html) | Antigravity CLI から Adobe Analytics MCP に接続する実践ガイド。対話OAuthが使えない Antigravity 向けに、24時間で失効する IMS トークンをローカル stdio プロキシでオンデマンド自動更新する、実際に動いた構成（プロキシ全文・動作確認・トラブルシュート付き）。Gemini CLI は Claude / ChatGPT 同様に MCP サーバー登録だけで繋がるため本ガイドは不要。 |

## 免責事項

本資料は個人による学習用コンテンツであり、Adobe Inc. および Google LLC の公式見解・サポート文書ではありません。記載内容の正確性・完全性・将来にわたる妥当性を保証するものではありません。エンドポイント・設定スキーマ・認証仕様は各社の更新で変わり得ます。実装時は必ず各社の公式ドキュメントで最新仕様をご確認ください。
