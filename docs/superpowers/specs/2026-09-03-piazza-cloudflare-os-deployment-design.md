# PIAZZA 向け Cloudflare OS 本番デプロイ設計

作成日: 2026-09-03
状態: 設計承認済み（実装計画は未作成）

## 1. 目的

`piazza-life/cloudflare-os`（upstream `cloudflare/cloudflare-os` のフォーク）を piazza-life の
Cloudflare アカウントにデプロイし、社員が Google Workspace アカウントでログインして、
本人の権限で Google Workspace / GitHub / Slack / Notion / BigQuery / poi に接続しながら
業務改善のアプリやワークフローを作れる状態にする。

成功条件:

- `@piazza-life.com` 以外のアカウントは公開 URL にアクセスできない。
- 管理者が `/admin` に入り、サインアップ可否・ブランディング・使える connector を制御できる。
- 各社員が Google（BigQuery を含む）/ GitHub / Slack / Notion を自分の OAuth 認可で接続できる。
- 各社員が poi-mcp を URL 貼付だけで接続できる（poi 側の対応完了後）。
- 社員は API キーを持たずに Anthropic のモデルでチャットできる。
- upstream の更新を submodule のピン更新だけで取り込める。

## 2. 調査で確定した前提

- フォークは upstream `main` と同一コミット（差分なし）。
- upstream は単体では本番デプロイ手段を持たない。公式の自己管理デプロイ経路は
  `cloudflare/cloudflare-os-starter`（upstream を submodule にし、`deployment.jsonc` と
  `scripts/deploy.ts` で 6 Worker を生成・デプロイする）。ただし starter が配る Gatekeeper は
  context / scheduler / custom の 3 つのみで、Google 等の vendor Gatekeeper は含まれない。
- 認証は Cloudflare Access モードが最も堅い。backend が `CF_ACCESS_ISS` の JWKS で
  `cf-access-jwt-assertion` を検証し、`CF_ACCESS_AUD` が設定されるとパスワード認証は閉じる。
  フロントは `VITE_CF_ACCESS_MODE=true` でのビルドが必要（ビルド時定数）。
- upstream にはメールドメインの許可リスト機能はない。ドメイン制限は Access ポリシーで行う。
- `piazza-life.com` の DNS は AWS Route53 にあり、Cloudflare にゾーンがない。Workers の
  Custom Domain・MCP Server Portal・Email Gatekeeper はゾーン必須のため今回は使えない。
  Subdomain setup（サブドメインだけの委任）は Enterprise 限定。
- Access は workers.dev ホスト名も保護できる。
- LLM は deployment 側 AI Gateway モードが使える。Gateway が同一アカウントにあれば
  `WORKERS_AI` binding 経由で API トークン不要（`google` provider を除く）。
- `gatekeeper-google` は BigQuery を標準サポート（`bigquery` スコープ、読み取り専用 SQL の強制、
  リソース単位の権限、専用 configurator UI）。
- piazza-data-analysis の「BigQuery MCP」は自作サーバーではなく、Google 提供の
  `https://bigquery.googleapis.com/mcp` を手動登録の Google OAuth クライアントで使う運用
  （`piazza-data-analysis/mcp/DECISION.md`）。したがって Cloudflare OS では MCP ではなく
  Google Gatekeeper の BigQuery リソースを使う。
- poi-mcp は `https://poi-mcp.piazza-life.com/mcp`（Cloud Run、公開、Streamable HTTP、`POST /mcp`）。
  RFC 9728 の protected-resource metadata は返すが、認可サーバーが Google 本体で
  Dynamic Client Registration（RFC 7591）非対応。`gatekeeper-mcp` は DCR 前提のため
  現状では接続できない。
- 現在のローカル `wrangler` と Cloudflare MCP コネクタは個人アカウント
  （hiromitsu.sasaki@promitsu.jp）にログインしている。piazza-life アカウントでの認証が別途必要。

## 3. 決定事項

| 論点 | 決定 | 理由 |
|---|---|---|
| ログイン | Cloudflare Access + Google ログイン、ポリシーは email domain = piazza-life.com | 未認証リクエストが Worker に届かない。upstream 側にドメイン制限機能がない |
| 公開 URL | workers.dev で開始 | ゾーンがない。DNS 移管は会社全体の判断なので切り離す |
| デプロイの器 | starter 方式の新リポジトリ `piazza-life/cloudflare-os-deploy` | upstream 追従が submodule 更新だけで済む。フォークは無改変で温存 |
| vendor Gatekeeper | deploy.ts を拡張して Google / GitHub / Slack / Notion / MCP を同時デプロイ | starter の対象外だが、同じ生成パターンで足せる |
| BigQuery | Google Gatekeeper の BigQuery リソース | 既に upstream にあり、本人権限・読み取り専用が担保される |
| poi | poi-mcp に DCR 対応の OAuth ファサードを追加（poi リポジトリ側の別スペック） | フォークを改変せず、Claude Code 等の client_id 配布問題も解消 |
| LLM | 同一アカウントの AI Gateway に Anthropic キーを BYOK 登録、providers = anthropic, cloudflare | 社員はキー不要。コストとログを Gateway で一元化。トークン不要 |
| 管理者 | `ADMINS = ["hiromitsu.sasaki@piazza-life.com"]` | 初期管理者は依頼者本人 |
| デプロイ実行 | 当面ローカルから | CI 化は動作確認後 |

## 4. 全体構成

### 4.1 Worker 一覧（piazza-life アカウント）

| Worker 名 | 役割 | 由来 | 公開ルート |
|---|---|---|---|
| `piazza-os` | router。フロント配信、`/api` と `/blueprint-screenshot` を workshop へ、`/gatekeeper/<name>/*` を binding 名で該当 Gatekeeper へ転送 | upstream `packages/router` | あり（workers.dev） |
| `piazza-os-workshop` | backend。全ユーザーデータは Durable Objects | upstream `packages/workshop-backend` | なし |
| `piazza-os-context` | Context Gatekeeper（ambient、PREINSTALL） | upstream | なし |
| `piazza-os-scheduler` | Scheduler Gatekeeper（ambient、PREINSTALL） | upstream | なし |
| `piazza-os-gk-google` | Google（Gmail / Docs / Drive / Sheets / Calendar / BigQuery） | upstream `packages/gatekeeper-google` | なし |
| `piazza-os-gk-github` | GitHub | upstream | なし |
| `piazza-os-gk-slack` | Slack（読み取り専用） | upstream | なし |
| `piazza-os-gk-notion` | Notion | upstream | なし |
| `piazza-os-gk-mcp` | 任意リモート MCP（poi 用） | upstream `packages/gatekeeper-mcp` | なし |
| `piazza-os-custom` | starter の例示 Gatekeeper。PIAZZA の社内ガイダンスを agent に提示する用途 | starter `packages/custom-gatekeeper` | なし |
| `piazza-os-error-reporter` | 構造化エラーログ | starter `packages/error-reporter` | なし |

router 以外は workers.dev と Preview URL を無効化し、service binding 経由でのみ到達可能にする。

### 4.2 リクエストの流れ

1. ブラウザが `https://piazza-os.<subdomain>.workers.dev` にアクセスする。
2. Cloudflare Access が Google ログインを要求し、`@piazza-life.com` のみ通す。
3. router がフロントを配信し、`/api` を workshop へ転送する。workshop は Access JWT を検証し、
   メールをキーに `UserDurableObject` を解決する。
4. ユーザーが connector を接続すると、`/gatekeeper/<name>/oauth` が該当 Gatekeeper Worker に届き、
   Gatekeeper が本人の OAuth 認可を行いトークンを自分の DO に保存する。
5. agent や Gadget は workshop から `GatekeeperVendor` entrypoint 経由で Gatekeeper を呼ぶ。

### 4.3 バインディング

- workshop → 各 Gatekeeper: `GATEKEEPER_<NAME>`、`entrypoint: "GatekeeperVendor"`。
  context のみ `props: { sharingDomain: <origin> }`。
- router → workshop: `WORKSHOP_BACKEND`。router → 各 Gatekeeper: `GATEKEEPER_<NAME>`（entrypoint なし）。
  router は `GATEKEEPER_` 接頭辞を小文字化して `/gatekeeper/<name>` に対応付ける
  （`packages/router/src/index.ts`）。
- workshop: `BLUEPRINTS` / `AVATARS`（KV）、`BLUEPRINT_CONTENT`（R2）、`LOADER`（worker_loaders）、
  `BROWSER`、`WORKERS_AI`。KV / R2 は Wrangler の自動プロビジョニングに任せる。
- context: `CONTEXT_COLLECTIONS`（KV）。Artifacts は無効。

### 4.4 環境変数と secret

| Worker | vars | secrets |
|---|---|---|
| workshop | `ADMINS`, `CF_ACCESS_ISS`, `CF_ACCESS_AUD`, `PUBLIC_BASE_URL`, `CF_AI_GATEWAY`, `CF_AI_GATEWAY_ACCOUNT_ID`, `CF_AI_GATEWAY_PROVIDERS` | なし（binding 転送のためトークン不要） |
| gk-google / gk-github / gk-slack / gk-notion | `BASE_URL = <origin>/gatekeeper/<name>` | `CLIENT_ID`, `CLIENT_SECRET` |
| gk-mcp | `BASE_URL`, `MCP_CLIENT_NAME = "PIAZZA OS"` | なし |
| custom | `CUSTOM_NAME`, `CUSTOM_MESSAGE` | なし |

`AUTH_GATEKEEPERS` と `DISABLE_PASSWORD_AUTH` は設定しない（Access モードで不要）。

## 5. 認証・権限

### 5.1 Cloudflare Access

- Zero Trust チーム `https://<team>.cloudflareaccess.com` を用意する。
- IdP は Zero Trust の Google ログイン（Google Cloud の OAuth クライアントを 1 つ登録、
  リダイレクト URI は `https://<team>.cloudflareaccess.com/cdn-cgi/access/callback`）。
  グループ単位の制御が必要になったら Google Workspace IdP に格上げする。
- self-hosted アプリケーションを router の workers.dev ホスト名に対して作成し、
  Allow ポリシーは「Emails ending in `@piazza-life.com`」のみとする。
- アプリの AUD タグを `deployment.jsonc` の `access.audience` に、チームのオリジンを
  `access.issuer` に書く。

### 5.2 アプリ内の権限

- `ADMINS` は `/admin` に入れるメールの平坦なリスト。役割階層はない。
- `/admin` で制御するもの: 新規サインアップ可否、サイト名・ロゴ・アクセントカラー、
  お知らせ、agent への追加指示、無効化する connector / resource、ambient Gatekeeper の
  disabled / optional / enabled。
- 外部サービスへのアクセスは Gatekeeper ごとにユーザー本人の OAuth 認可で、agent と Gadget には
  リソース単位で「紹介」した範囲だけが渡る（upstream の capability 方式）。副作用のある操作は
  ユーザー承認待ちになる。

## 6. 外部サービス側の登録（ユーザー作業）

エージェントは手順書とリダイレクト URI 一覧を用意し、登録とキー投入はユーザーが行う。

| サービス | 登録物 | リダイレクト URI（origin = 公開 URL） | 備考 |
|---|---|---|---|
| Cloudflare Zero Trust | チーム、Google IdP、self-hosted app | IdP 用: `https://<team>.cloudflareaccess.com/cdn-cgi/access/callback` | AUD タグを控える |
| Google Cloud（`piazza-1174` または新規プロジェクト） | OAuth クライアント（Web application）、同意画面は Internal | `<origin>/gatekeeper/google/oauth` | 有効化する API: Gmail, Docs, Drive, Sheets, Calendar, BigQuery。BigQuery 実行には `piazza-bq-analysts@piazza-life.com` 相当の IAM が本人に必要 |
| GitHub（piazza-life org） | OAuth App（GitHub App ではない） | `<origin>/gatekeeper/github/oauth` | org の OAuth App 承認ポリシーに注意 |
| Slack | App。User Token Scopes、Token Rotation 有効 | `<origin>/gatekeeper/slack/oauth` | 読み取り専用 |
| Notion | Public integration | `<origin>/gatekeeper/notion/oauth` | ページはユーザーが Notion 側で共有したものだけ見える |
| Cloudflare AI Gateway | Gateway `piazza-os`、Anthropic キーを BYOK 登録 | なし | 同一アカウント。Workers AI も同じ Gateway を通る |
| Anthropic | 組織の API キー | なし | Gateway に保存し、リポジトリには置かない |

secret は gitignore 済みの `secrets.local.jsonc` に Worker ごとに書き、`pnpm secrets` が
`wrangler secret bulk` で投入する。値はコマンドラインに出さない。

## 7. deploy リポジトリ `piazza-life/cloudflare-os-deploy`

### 7.1 構成

- `cloudflare/cloudflare-os-starter` の内容を土台にし、submodule を
  `piazza-life/cloudflare-os` に向けて現在のコミットでピン止めする。
- `deployment.jsonc` を PIAZZA 向けに埋める（アカウント ID、Worker 名、`publicBaseUrl`、
  `route: { workersDev: true }`、Access、AI Gateway、custom Gatekeeper の文言）。
- 追加するもの: `gatekeepers` セクション、`scripts/secrets.ts`、`secrets.local.example.jsonc`、
  `docs/setup-runbook.md`（外部サービス登録手順）。

### 7.2 `deployment.jsonc` の拡張

```jsonc
"gatekeepers": {
  "google": { "name": "piazza-os-gk-google" },
  "github": { "name": "piazza-os-gk-github" },
  "slack":  { "name": "piazza-os-gk-slack" },
  "notion": { "name": "piazza-os-gk-notion" },
  "mcp":    { "name": "piazza-os-gk-mcp", "clientName": "PIAZZA OS" }
}
```

キーを省略した Gatekeeper はデプロイしない。キーは upstream の `packages/gatekeeper-<key>` と
binding 名 `GATEKEEPER_<KEY大文字>` に対応する。

### 7.3 `scripts/deploy.ts` の拡張

- `packageDirs` に vendor Gatekeeper を動的に追加し、各 `wrangler.jsonc` を読み込んで
  `wrangler.prod.jsonc` を派生する。共通処理（`account_id`、名前、workers.dev / preview 無効、
  observability）は既存の `setCommon` を使う。
- 派生 config に `vars.BASE_URL = <origin>/gatekeeper/<key>` を設定する。upstream の
  wrangler.jsonc には `BASE_URL` がないため、deploy 側で必ず与える。
- workshop の `services` に `GATEKEEPER_<KEY>`（`entrypoint: "GatekeeperVendor"`）、router の
  `services` に `GATEKEEPER_<KEY>`（entrypoint なし）を追加する。
- ビルドは各 Gatekeeper の `deploy` スクリプトと同じ手順を再現する:
  `vp run --no-cache build:configurator` の後、`wrangler deploy` が wrangler.jsonc の
  `build.command`（`capnweb-validate build`）を実行する。
- デプロイ順: error-reporter → 全 Gatekeeper（並列可） → workshop → router。
- `pnpm check`（`--check`）は既存どおり dry-run と `deploy.test.ts` を走らせ、加えて
  登録すべきリダイレクト URI の一覧と、secret が未投入の Worker を表示する。
- 名前の一意性チェックに Gatekeeper 名を含める。

### 7.4 `scripts/secrets.ts`

- `secrets.local.jsonc` を読み、Worker 名ごとに `wrangler secret bulk --name <worker>` へ
  標準入力で渡す。`CLOUDFLARE_ACCOUNT_ID` を `deployment.jsonc` の値で固定する。
- 値をログに出さない。ファイルが無い場合は example のコピーを促して終了する。

### 7.5 テスト

- `deploy.test.ts` に以下を追加する: Gatekeeper 有無で `services` が正しく生成される、
  `BASE_URL` が origin から導出される、名前の衝突を検出する、`gatekeepers` 省略時は
  starter と同一の出力になる。
- `pnpm check` で全 Worker の dry-run が通ること。

## 8. poi-mcp の DCR 対応（poi リポジトリ、別スペック）

本設計の依存先。poi リポジトリで別途 spec と plan を作る。ここでは境界だけ定める。

- poi-mcp 自身が認可サーバーとして振る舞う薄いファサードを足す。
  - `GET /.well-known/oauth-authorization-server`: issuer = poi-mcp のオリジン、
    `registration_endpoint`、PKCE S256、`authorization_code` と `refresh_token`。
  - `POST /register`: 公開クライアントとして受け付け、`redirect_uris` 等を署名付きの
    `client_id` に内包してステートレスに保持する（Cloud Run に永続ストアを足さない）。
  - `GET /authorize`: `client_id` と `redirect_uri` を検証し、poi の静的 Google クライアントで
    Google へ転送する。クライアントの `state` と PKCE challenge は署名付き `state` に載せる。
  - `GET /callback`: Google の code をトークンに交換し、暗号化した poi 認可コードを
    クライアントの `redirect_uri` に返す。
  - `POST /token`: PKCE を検証し、Google の access token と refresh token をそのまま返す。
    `refresh_token` グラントは Google に中継する。
- 既存の `Authorization: Bearer <Google access token>` 検証（tokeninfo、`aud`、
  `email_verified`、`@piazza-life.com`）は変更しない。
- protected-resource metadata の `authorization_servers` を poi-mcp 自身のオリジンに変える。
- 完了後、Cloudflare OS 側では `gatekeeper-mcp` の接続フォームに
  `https://poi-mcp.piazza-life.com/mcp` を貼るだけで接続できる。

## 9. 検証

1. `pnpm check` が通り、dry-run で 11 Worker の config が生成される。
2. `pnpm deploy` 後、公開 URL に個人 Google アカウントでアクセスすると Access で拒否される。
   `@piazza-life.com` では入れる。
3. `/admin` に管理者で入れ、非管理者では入れない。
4. Google を接続し、BigQuery リソースで `piazza-1174` に読み取りクエリを実行できる。
   Gmail / Docs / Drive / Sheets / Calendar のいずれかでも接続できる。
5. Slack / GitHub / Notion を接続し、agent から読み取りができる。
6. Anthropic のモデルがモデル一覧に出て、キーを入力せずにチャットできる。AI Gateway の
   ログに記録される。
7. agent に数分後のスケジュールを頼み、Scheduler が動く。
8. poi-mcp の DCR 対応後、`gatekeeper-mcp` から poi に接続してページを一覧できる。
9. 各 Worker の Workers Logs にエラーが出ていない。

## 10. 運用

- 更新: フォークに upstream を取り込む → deploy リポジトリの submodule を進める →
  `pnpm-workspace.yaml` の `catalog:` を再同期 → `pnpm check` → `pnpm deploy` → 検証。
- 戻し: `wrangler rollback`（binding 互換の範囲）または submodule を前のコミットに戻して再デプロイ。
- 監視: Error Reporter の構造化ログと各 Worker の Workers Logs。
- secret のローテーションは Cloudflare / 各サービスのコンソールでユーザーが行う。

## 11. スコープ外（後続候補）

- カスタムドメイン（`piazza-life.com` を Cloudflare に移管した後）。移行時は Access アプリと
  各 OAuth アプリのリダイレクト URI を登録し直す。
- MCP Server Portal、Email Gatekeeper（いずれもゾーン必須）。
- Linear / Confluence / Supabase など他の upstream Gatekeeper。`gatekeepers` にキーを足すだけで
  追加できる設計にする。
- GitHub Actions からのデプロイ。
- 個人アカウント（promitsu.jp）に残っている 2026-08-06 の `cloudflare-os-*` Worker の整理。

## 12. 前提としてユーザーに用意してもらうもの

- piazza-life Cloudflare アカウントの ID と、そのアカウントでの `wrangler login`
  （または Workers / KV / R2 / AI 権限付き API トークン）。
- Zero Trust チーム名（未作成なら作成）。
- Google Cloud で OAuth クライアントを作成し API を有効化できる権限。
- GitHub org、Slack、Notion でアプリを作成できる権限。
- Anthropic の組織 API キー。
- Workers Paid など、Durable Objects / Worker Loader / Browser Rendering / Workers AI /
  AI Gateway が使えるプラン。初回 `pnpm check` と最初のデプロイで確認する。
