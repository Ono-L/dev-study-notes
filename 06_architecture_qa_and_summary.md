# 06. 全プロジェクト設計要点まとめ & 技術 Q&A

> 3 つのプロジェクトの設計ポイントと「なぜこの構成にしたか」を 3 分で見返せる要約ノートです。

---

## 1. 制作物 30 秒サマリー表

| プロジェクト | 何を解決したか | 主な設計ポイント | セキュリティ & 品質 |
| :--- | :--- | :--- | :--- |
| **[SDD Progress Tracker & Portfolio](https://github.com/Ono-L/ai-travel-planner)** | 仕様書と実装の乖離解消＆Web ポートフォリオの無料公開 | Vite 自動同期 + ハイブリッド LLM + Remote MCP を **1 つの Cloudflare Worker（Static Assets）無料枠** に統合 | Web Crypto API で HMAC-SHA256 署名付き `HttpOnly` Cookie 認証・定数時間比較・OWASP LLM01 対策・`strict: true` |
| **[gbf-beginner-bot](https://github.com/Ono-L/gbf-beginner-bot)** | 複雑なゲーム仕様を初心者が Discord で即座に質問・解決 | 16 本の自前ナレッジを 768 次元 + L2 正規化で Vectorize/D1 へ格納、`waitUntil()` で Discord 3 秒制約を解決、Ephemeral デフォルト UX | Ed25519 署名検証・スクレイピング禁止・3 重ハルシネーション抑制・自動モデルフォールバック・`<user_question>` 隔離・**Vitest 27 テスト** |
| **[epaper-dashboard](https://github.com/Ono-L/epaper-dashboard)** | 天気と iPhone 通知をデスク常設の低消費電力電子ペーパーで視認 | `DisplayDriver` 抽象化（`mock` / `waveshare`）＋ Docker ブラウザプレビューにより **実機なしで開発・テスト完結** | Python 3.12 型ヒント + Pydantic v2・`hmac.compare_digest` 認証・401 フェイルクローズ・Open-Meteo 障害フォールバック・**pytest 3 スイート** |
| **[Trip Splitter (Archive)](https://github.com/Ono-L/ai-travel-planner/tree/archive/trip-splitter-v1)** | グループ旅行の立替管理と最小送金精算 | DevContainer + pnpm/Turborepo + Expo + Hono RPC 実機検証後、**ADR-001** でコスト(\$0)＆URL 参加 UX のため Web 路線へピボット | `packages/shared` の Zod 単一ソース化・Git ブランチ＆ADR で意思決定履歴を保全 |

---

## 2. 設計意図・技術 Q&A 10 選

### Q1. なぜ Trip Splitter をモバイルから Web へピボットしたのか？
- Phase 1（Expo + Hono RPC の実機トンネル検証）まで終えた段階で、①ストア公開の固定費（Apple 年額 \$99 等）をゼロにして継続公開したい、②旅行同行者がアプリを入れず共有 URL だけで参加できる UX の方が体験が良い、と判断したため（ADR-001）。既存実装も `archive/trip-splitter-v1` に保全している。

### Q2. なぜ Cloudflare Workers Static Assets にしたのか？
- React フロントと Hono API を 1 つの Worker で同一オリジン配信できるため、CORS の煩雑さと複数サービスの管理コストを無くしつつ無料枠で公開できるため。

### Q3. Workers で認証をどう実装したのか？
- エッジ環境は Node.js `crypto` が使えないため、標準の Web Crypto API（`crypto.subtle`）で HMAC-SHA256 署名付きの `HttpOnly` / `SameSite=Lax` Cookie セッションを実装した。パスワードと Bearer トークン比較には SHA-256 ダイジェストの定数時間比較を使い、タイミング攻撃を防いでいる。

### Q4. なぜ Embedding を 768 次元＋L2 正規化したのか？
- `gemini-embedding-001` の 3072 次元は Vectorize の上限を超えるため 768 次元へ縮小した。3072 次元未満の出力は単位ベクトルにならないため、`normalizeL2` で長さを 1 に揃えてコサイン類似度検索の精度を保っている。さらに登録時は `RETRIEVAL_DOCUMENT`、検索時は `RETRIEVAL_QUERY` を使い分けている。

### Q5. Discord Bot の 3 秒タイムアウト制約はどう解決したのか？
- `/ask` は即座に Deferred（`type: 5`）を返し、Cloudflare Workers の `ctx.waitUntil()` で非同期に RAG 検索〜Gemini 回答生成を実行し、完了後に Discord Webhook へ `PATCH` して回答を表示している。

### Q6. AI のハルシネーションやプロンプトインジェクション対策は？
- ハルシネーションは「①低スコア除外 → ②0 件時は LLM を呼ばず定型文 → ③根拠不足時は `[[NO_EVIDENCE]]` マーカーでブロック＋出典はコード側で自動付与」の 3 重ガード。
- プロンプトインジェクションは `<spec_input>` / `<user_question>` タグでの構造的隔離と、入力内の閉じタグ事前サニタイズ、さらに `systemInstruction` との分離で緩和している。

### Q7. 外部 API が落ちたらどうなる設計か？
- `gbf-beginner-bot` は Gemini API の `404/429/5xx` や 15 秒タイムアウト時にフォールバックモデルへ自動切替する。
- `epaper-dashboard` は Open-Meteo 通信障害時に例外で止めず `is_fallback=True` のフォールバック天気情報を返し、時刻・通知表示を継続する（`httpx.MockTransport` でテスト済み）。

### Q8. 実機（Raspberry Pi）なしで電子ペーパー開発を進めた工夫は？
- Pillow の 400×300 / 1bit 描画エンジンと出力ドライバを `DisplayDriver` 抽象基底クラス＋Factory で分離した。環境変数 `DISPLAY_MODE=mock` で PNG 出力＆ブラウザプレビュー（`/preview`）を確認でき、実機移行時は描画コード無修正で `waveshare` へ切り替えられる。

### Q9. セキュリティで全プロジェクト共通に意識していることは？
- ①シークレットは Git に入れない（`.env` / `.dev.vars` 除外・本番は `wrangler secret put`、API キーは URL クエリではなくヘッダー送信）、②フェイルクローズ（キー未設定時は必ず `401/503` 拒否）、③タイミング攻撃対策（定数時間比較）、④SQL パラメータバインド、⑤外部スクレイピング禁止。

### Q10. 仕様駆動開発（SDD）を実践してよかった点は？
- 実装前に要件・DB スキーマ・規約・未確定事項（TBD）を整理するため、手戻りや AI 生成時の不要ライブラリ混入・設計ブレが激減した。また ADR に判断根拠が残るため、後から見返しても他者に説明しても意図が明確になる。
