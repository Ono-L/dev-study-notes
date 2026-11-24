# dev-study-notes（個人開発プロジェクト技術解説・復習ノート）

これまでに構築した 3 つの個人開発プロジェクト（[ai-travel-planner](https://github.com/Ono-L/ai-travel-planner) / [gbf-beginner-bot](https://github.com/Ono-L/gbf-beginner-bot) / [epaper-dashboard](https://github.com/Ono-L/epaper-dashboard)）で使った技術・設計・セキュリティ・テストの仕組みを、**「なぜそう設計したか」「どう動いているか」** まで噛み砕いてまとめた学習・復習用リポジトリです。

スマホや PC から GitHub 上でそのまま見返して、日々の復習や技術整理に使えます。

---

## 目次（テーマ別解説ノート）

| No. | ファイル | 主なトピック | 対応プロジェクト |
| :---: | :--- | :--- | :--- |
| 01 | [01_cloudflare_edge_and_hono.md](./01_cloudflare_edge_and_hono.md) | Cloudflare Workers / D1 / Static Assets / Hono / Zod / `waitUntil()` | `ai-travel-planner`, `gbf-beginner-bot` |
| 02 | [02_rag_and_embeddings.md](./02_rag_and_embeddings.md) | RAG 全体フロー / Embedding 768次元＋L2正規化 / Vectorize / ハルシネーション対策 | `gbf-beginner-bot` |
| 03 | [03_security_and_auth.md](./03_security_and_auth.md) | Ed25519 署名検証 / HttpOnly Cookie セッション / タイミング攻撃対策 / OWASP LLM01 | 全 3 プロジェクト |
| 04 | [04_python_fastapi_and_iot.md](./04_python_fastapi_and_iot.md) | Python 3.12 型ヒント / FastAPI / Pydantic v2 / ドライバ抽象化 / Pillow 1bit 描画 | `epaper-dashboard` |
| 05 | [05_sdd_adr_mcp_and_testing.md](./05_sdd_adr_mcp_and_testing.md) | 仕様駆動開発（SDD）/ ADR / Remote MCP（JSON-RPC 2.0）/ 自動テスト / Git 運用 | 全 3 プロジェクト |
| 06 | [06_architecture_qa_and_summary.md](./06_architecture_qa_and_summary.md) | 全プロジェクト設計要点まとめ & 技術 Q&A 10 選 | 全 3 プロジェクト |

---

## おすすめの復習ルート

1. **まずは全体像をつかむ**: [06_architecture_qa_and_summary.md](./06_architecture_qa_and_summary.md) の要点表をざっと読む（3 分）
2. **得意領域を深く理解する**:
   - Web・クラウド寄り → `01` → `03` → `05`
   - AI・RAG 寄り → `02` → `03` → `05`
   - Python・バックエンド／IoT 寄り → `04` → `03`
3. **実コードと照らし合わせる**: 各ノート内のリンクから実リポジトリの該当ファイルを開き、「コードの何行目がその役割か」を確認する
