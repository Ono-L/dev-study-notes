# 05. 仕様駆動開発（SDD）・ADR・Remote MCP・テスト・Git 運用解説

> 関連リポジトリ: [ai-travel-planner](https://github.com/Ono-L/ai-travel-planner) / [gbf-beginner-bot](https://github.com/Ono-L/gbf-beginner-bot) / [epaper-dashboard](https://github.com/Ono-L/epaper-dashboard)

---

## 1. 仕様駆動開発（SDD: Spec-Driven Development）

### 一言でいうと
コードから書き始めるのではなく、**「要件・設計・制約・未決定事項」をまず Markdown 仕様書として言語化し、それを唯一の正本（Single Source of Truth）として人間と AI が実装を進める開発手法**です。

### なぜ AI 時代の開発で効くのか？
- AI エージェントに曖昧な指示で書かせると、勝手に不要なライブラリを入れたり、既存設計と矛盾するコードを生んだりしがちです。
- そこで全プロジェクトに `AGENTS.md` / `CLAUDE.md` を配置し、「仕様書を読む」「`package.json` / `pyproject.toml` 外のライブラリを勝手に足さない」「未確定事項（TBD）を独断で実装しない」「変更後は必ず型チェックとテストを通す」をルール化しています。
- `ai-travel-planner` ではさらに、**Vite カスタムプラグイン** で `docs/` 配下の YAML フロントマターを自動監視・同期し、仕様書と進捗ボードがズレない仕組みまで実装しています。

---

## 2. ADR（Architecture Decision Record / 技術的意思決定記録）

### 一言でいうと
**「何を作ったか」だけでなく「なぜその技術を選び、別の選択肢を捨てたのか（トレードオフ）」を記録するドキュメント**です。

### 実例：Trip Splitter の ADR-001
- **背景**: Phase 1 で Expo（React Native）+ Hono RPC のモバイルプロトタイプを実機検証まで完了した。
- **意思決定**: モバイルアプリストア公開を見送り、Cloudflare Workers Static Assets による Web 公開路線へピボット。
- **理由**:
  1. Apple Developer Program（年額 \$99）等のランニングコストを排し、無料枠（\$0）で継続運用するため。
  2. 旅行同行者にアプリをインストールさせる手間を省き、URL 共有だけで即参加できる UX を優先するため。
- **保全**: 既存のモバイル実装も消さず `archive/trip-splitter-v1` ブランチとして保全。

> **ポイント**: 「最初はこう考えて試したが、コストと UX を評価してこう方針転換した」という判断の背景を残しておくことで、あとから見返したときも設計意図がぶれにくくなります。

---

## 3. Remote MCP（Model Context Protocol）の仕組み

### 一言でいうと
Claude Desktop や VS Code などの **AI クライアントと、外部のツール・データベースを標準化されたやり方でつなぐためのプロトコル（USB-C のような共通規格）** です。

### `ai-travel-planner` の実装（`POST /api/sdd/mcp`）
HTTP 上で **JSON-RPC 2.0** メッセージをやり取りします：

1. **`tools/list`**: AI クライアントが「このサーバーにはどんなツールがある？」と問い合わせ → `list_sdd_tasks`（SDD タスク一覧取得ツール）の定義と入力スキーマを返却。
2. **`tools/call`**: AI クライアントがツールを実行 → Cloudflare D1 から最新タスク一覧を取得し、テキスト形式で返却。

ローカル専用（stdio）の MCP ではなく **Cloudflare Workers 上のリモート HTTP エンドポイント** として実装したことで、端末を問わず外部 AI から仕様進捗へアクセスできる構成になっています。

---

## 4. 型安全（TypeScript `strict: true`）と自動テストの考え方

### TypeScript `strict: true` と `any` 全面禁止
- `any` を使うと、その変数に触れた箇所すべての型チェックが無効化され、実行時エラー（`Cannot read properties of undefined` 等）の温床になります。
- 外部からの JSON 入力など型が分からないデータは **`unknown`** として受け取り、**Zod スキーマ** や **型ガード関数** で検証してから使うのが安全な設計です。

### 自動テストで何を守っているか？
- **`gbf-beginner-bot`（Vitest / 5 ファイル・27 テスト）**:
  - `chunker.test.ts`: Markdown の見出し分割・文字数制御
  - `embedder.test.ts`: 768 次元出力と L2 正規化（長さ 1 になること）
  - `retriever.test.ts`: コサイン類似度順のソート・閾値除外
  - `answer.test.ts`: 根拠 0 件時の LLM 非呼び出し、`[[NO_EVIDENCE]]` 検知、モデル自動フォールバック、`<user_question>` プロンプトインジェクション隔離
  - `interactions.test.ts`: Ed25519 不正署名 401 拒否、PING→PONG、`/ask` の Deferred + `waitUntil()` Webhook 更新、Ephemeral / public 切替
- **`epaper-dashboard`（pytest / 3 ファイル）**:
  - API 認証（未指定・不正・サーバー未設定の 401）・CRUD・プレビュー画像
  - 400×300 / 1bit 画像生成・不正サイズ拒否・実機スタブ例外
  - Open-Meteo パース・ネットワーク障害フォールバック

---

## 5. Git 運用 & 開発環境管理の基礎

### なぜ開発フォルダを Google Drive 同期フォルダに置いてはいけないか？
1. `.git/` がクラウド同期の競合コピーで破損しやすい。
2. `pnpm` の `node_modules` は大量のシンボリックリンク・ハードリンクを使うため、仮想ファイルシステム（Google Drive）上で壊れる。
3. Python の `.venv` は絶対パスが埋め込まれているため移動すると壊れる。
4. SQLite DB（`notifications.db` や `.wrangler` 状態）は書き込み中に同期が走ると破損する。
5. `.env` などの機密ファイルが意図せずクラウドへ上がるリスクがある。

→ **コードの同期・バックアップは GitHub、環境の再現性は Docker / DevContainer** に任せるのが鉄則です。

### Conventional Commits（日本語本文）
`feat:`（新機能）、`fix:`（バグ修正）、`docs:`（ドキュメント）、`refactor:`（リファクタ）、`test:`（テスト）、`chore:`（設定・雑務）のように接頭辞を揃えることで、後から `git log` を見ただけで変更の意図が一瞬で分かります。

---

## 復習チェック質問（自分の言葉で説明してみよう）
1. なぜ TypeScript で `any` ではなく `unknown` を使うべきなのか？
2. ADR を残すことで、将来の自分やチームにどんなメリットがあるか？
3. MCP の `tools/list` と `tools/call` はそれぞれ何をするメソッドか？
4. 開発プロジェクトを Google Drive ではなく Git + GitHub で管理すべき理由を 3 つ挙げよ。
