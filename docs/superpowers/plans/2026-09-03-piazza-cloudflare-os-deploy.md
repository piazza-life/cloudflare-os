# PIAZZA Cloudflare OS Deploy Repository Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `piazza-life/cloudflare-os-deploy` を新設し、Cloudflare Access（Google ログイン、@piazza-life.com 限定）付きの Cloudflare OS を、Google / GitHub / Slack / Notion / MCP Gatekeeper 込みで piazza-life アカウントの workers.dev にデプロイできる状態にする。

**Architecture:** `cloudflare/cloudflare-os-starter` を clone して土台にし、submodule を `piazza-life/cloudflare-os` の現行コミットに向ける。`scripts/deploy.ts` に「`deployment.jsonc` の `gatekeepers` セクションに書かれた upstream Gatekeeper を追加で生成・ビルド・デプロイし、workshop と router に service binding を足す」拡張を入れる。secret 投入用の `scripts/secrets.ts` と、外部サービス登録の手順書を追加する。

**Tech Stack:** Node 24（TypeScript をそのまま実行）、pnpm 11.17、vite-plus（`vp run`）、wrangler 4、`node:test`、jsonc-parser。

**Spec:** `docs/superpowers/specs/2026-09-03-piazza-cloudflare-os-deployment-design.md`（`piazza-life/cloudflare-os` の `docs/piazza-deployment-spec` ブランチ）

## Global Constraints

- 新リポジトリの場所: `~/src/github.com/piazza-life/cloudflare-os-deploy`。GitHub 側の作成は `gh repo create piazza-life/cloudflare-os-deploy --private`（push とリポジトリ作成はユーザーの承認後に行う）。
- submodule `cloudflare-os` は `ssh://git@github.com/piazza-life/cloudflare-os.git` の `edfe520c7d544989fff3926576ffec840f80939c` に固定する。
- Worker 名: `piazza-os`, `piazza-os-workshop`, `piazza-os-context`, `piazza-os-scheduler`, `piazza-os-custom`, `piazza-os-error-reporter`, `piazza-os-gk-google`, `piazza-os-gk-github`, `piazza-os-gk-slack`, `piazza-os-gk-notion`, `piazza-os-gk-mcp`。
- 公開 URL は `https://piazza-os.<piazza-life の workers.dev サブドメイン>.workers.dev`。
- Gatekeeper の binding 名は `GATEKEEPER_<KEY を大文字化>`。workshop 側は `entrypoint: "GatekeeperVendor"`、router 側は entrypoint なし。
- 各 vendor Gatekeeper の `vars.BASE_URL` は `<origin>/gatekeeper/<key>`。MCP のみ `MCP_CLIENT_NAME`（既定 `"PIAZZA OS"`）を追加する。
- secret は `CLIENT_ID` / `CLIENT_SECRET`（google / github / slack / notion）。MCP は secret なし。値をコマンドラインやログに出さない。
- `ADMINS` は `["hiromitsu.sasaki@piazza-life.com"]`。
- AI Gateway: 名前 `piazza-os`、同一アカウント、`providers: ["anthropic", "cloudflare"]`。
- 全ステップで `--no-cache` の `vp run` を使う（starter のルール）。
- コミットメッセージ末尾に `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>` を付ける。`git push` はユーザーの明示的な指示があるときのみ。

---

## ファイル構成

deploy リポジトリ（新設）:

- `scripts/gatekeepers.ts` — 新規。Gatekeeper キー → パッケージディレクトリ・pnpm パッケージ名・secret 名・OAuth リダイレクトパスの対応表。`deploy.ts` / `secrets.ts` / テストが共有する唯一の定義元。
- `scripts/deployment-config.ts` — 変更。`gatekeepers` セクションの型を追加。
- `scripts/deploy.ts` — 変更。validate / generate / build / deploy に Gatekeeper を組み込む。`pnpm check` でリダイレクト URI と secret の一覧を出す。
- `scripts/deploy.test.ts` — 変更。Gatekeeper 生成のテストを追加。
- `scripts/secrets.ts` — 新規。`secrets.local.jsonc` を `wrangler secret bulk` へ流す。
- `scripts/secrets.test.ts` — 新規。
- `secrets.local.example.jsonc` — 新規。`.gitignore` に `secrets.local.jsonc` を追加。
- `deployment.jsonc` — 変更。PIAZZA の値。
- `pnpm-workspace.yaml` — 変更。`cloudflare-os/scripts` を追加、catalog を submodule に同期。
- `docs/setup-runbook.md` — 新規。外部サービス登録と初回デプロイの手順（日本語）。
- `README.md` — 変更。PIAZZA 向けの短い導入を先頭に追記。

---

### Task 1: deploy リポジトリの初期化と submodule のピン止め

**Files:**
- Create: `~/src/github.com/piazza-life/cloudflare-os-deploy/`（starter の clone）
- Modify: `.gitmodules`, `pnpm-workspace.yaml`

**Interfaces:**
- Produces: 以降の全タスクが前提とする作業ツリー。`pnpm test` が green な状態。

- [ ] **Step 1: starter を clone し、リモートを整理する**

```bash
cd ~/src/github.com/piazza-life
git clone https://github.com/cloudflare/cloudflare-os-starter.git cloudflare-os-deploy
cd cloudflare-os-deploy
git remote rename origin starter
git remote add origin ssh://git@github.com/piazza-life/cloudflare-os-deploy.git
git checkout -b main 2>/dev/null || git checkout main
```

`starter` リモートは将来 starter 側の更新を merge するために残す。GitHub 上のリポジトリ作成
（`gh repo create piazza-life/cloudflare-os-deploy --private`）と push は、ユーザーの承認を得てから行う。

- [ ] **Step 2: submodule をフォークに向け、コミットを固定する**

```bash
git submodule set-url cloudflare-os ssh://git@github.com/piazza-life/cloudflare-os.git
git submodule sync
git submodule update --init
git -C cloudflare-os fetch origin
git -C cloudflare-os checkout edfe520c7d544989fff3926576ffec840f80939c
git add .gitmodules cloudflare-os
```

確認: `git -C cloudflare-os rev-parse HEAD` が `edfe520c7d544989fff3926576ffec840f80939c` を返す。

- [ ] **Step 3: pnpm-workspace.yaml を submodule に同期する**

`packages:` に `cloudflare-os/scripts` を追加する（submodule の `error-reporting` が
`@gadgets/scripts: workspace:*` に依存するため、無いと `pnpm install` が失敗する）。
`catalog:` の各値を `cloudflare-os/pnpm-workspace.yaml` と一致させる。

```yaml
packages:
  - cloudflare-os/packages/workshop-shared
  - cloudflare-os/packages/error-reporting
  # error-reporting devDepends on @gadgets/scripts (workspace:*), which lives at the
  # submodule's top-level scripts/ directory.
  - cloudflare-os/scripts
  - packages/*

catalog:
  '@cloudflare/vitest-pool-workers': ^0.22.0
  capnweb: ^0.12.0
  capnweb-validate: 0.3.0
  typescript: 7.0.2
  vite: 7.3.6
  vite-plus: ^0.2.8
  vitest: ^4.1.10
  wrangler: ^4.125.0
```

既存のコメント・`overrides`・`allowBuilds` は残す。値は次のコマンドで突き合わせる:

```bash
diff <(sed -n '/^catalog:/,/^overrides:/p' cloudflare-os/pnpm-workspace.yaml | grep -v '^\s*#') \
     <(sed -n '/^catalog:/,/^overrides:/p' pnpm-workspace.yaml | grep -v '^\s*#')
```

期待: 差分なし。

- [ ] **Step 4: インストールして既存テストを通す**

```bash
pnpm install
pnpm --dir cloudflare-os install
pnpm test
```

期待: `scripts/deploy.test.ts` の全テスト PASS、`custom-gatekeeper` / `error-reporter` のテスト PASS。
失敗した場合は catalog の不一致か workspace メンバーの不足を疑う。upstream 側の変更で
starter の `buildCommands` が壊れている場合はこの時点で分かるので、submodule を
`6478a1448a11524e2f7c2575ad66fab0bc47c433`（starter が検証済みのコミット）に戻し、
ユーザーに報告して判断を仰ぐ。

- [ ] **Step 5: コミット**

```bash
git add .gitmodules cloudflare-os pnpm-workspace.yaml pnpm-lock.yaml
git commit -m "chore: pin cloudflare-os submodule to piazza-life fork and sync workspace catalog

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 2: Gatekeeper メタデータと deployment 型

**Files:**
- Create: `scripts/gatekeepers.ts`
- Modify: `scripts/deployment-config.ts`
- Test: `scripts/gatekeepers.test.ts`

**Interfaces:**
- Produces:
  - `GATEKEEPER_KEYS: readonly GatekeeperKey[]`（`"google" | "github" | "slack" | "notion" | "mcp"`）
  - `GATEKEEPERS: Record<GatekeeperKey, GatekeeperPackage>`、`GatekeeperPackage = { dir: string; pkg: string; secrets: readonly string[]; redirectPath: string | null }`
  - `bindingName(key: GatekeeperKey): string` → `GATEKEEPER_GOOGLE` 等
  - `enabledGatekeepers(config: DeploymentConfig): GatekeeperKey[]`（`GATEKEEPER_KEYS` の順）
  - `DeploymentConfig.gatekeepers?: GatekeepersConfig`
  - `GeneratedConfigs.gatekeepers: Partial<Record<GatekeeperKey, ProdWranglerConfig>>`
  - `BaseConfigs.gatekeepers: Partial<Record<GatekeeperKey, ProdWranglerConfig>>`

- [ ] **Step 1: 失敗するテストを書く**

`scripts/gatekeepers.test.ts`:

```ts
import assert from "node:assert/strict";
import test from "node:test";
import { GATEKEEPERS, GATEKEEPER_KEYS, bindingName, enabledGatekeepers } from "./gatekeepers.ts";
import type { DeploymentConfig } from "./deployment-config.ts";

test("binding names follow the router's GATEKEEPER_<NAME> convention", () => {
  assert.equal(bindingName("google"), "GATEKEEPER_GOOGLE");
  assert.equal(bindingName("mcp"), "GATEKEEPER_MCP");
});

test("every gatekeeper names its upstream package and redirect path", () => {
  for (const key of GATEKEEPER_KEYS) {
    const entry = GATEKEEPERS[key];
    assert.equal(entry.dir, `cloudflare-os/packages/gatekeeper-${key}`);
    assert.equal(entry.pkg, `@gadgets/${key}-gatekeeper`);
    if (key === "mcp") {
      assert.deepEqual(entry.secrets, []);
      assert.equal(entry.redirectPath, null);
    } else {
      assert.deepEqual(entry.secrets, ["CLIENT_ID", "CLIENT_SECRET"]);
      assert.equal(entry.redirectPath, `/gatekeeper/${key}/oauth`);
    }
  }
});

test("enabledGatekeepers returns configured keys in catalog order", () => {
  const config = {
    gatekeepers: { mcp: { name: "x-mcp" }, google: { name: "x-google" } },
  } as unknown as DeploymentConfig;
  assert.deepEqual(enabledGatekeepers(config), ["google", "mcp"]);
  assert.deepEqual(enabledGatekeepers({} as DeploymentConfig), []);
});
```

- [ ] **Step 2: テストが失敗することを確認する**

Run: `node --test scripts/gatekeepers.test.ts`
Expected: FAIL（`./gatekeepers.ts` が見つからない）

- [ ] **Step 3: `scripts/gatekeepers.ts` を書く**

```ts
// Which upstream Gatekeepers this deployment can add on top of the starter's three, and what
// each one needs. The single source for deploy.ts, secrets.ts and the tests.
import type { DeploymentConfig } from "./deployment-config.ts";

export const GATEKEEPER_KEYS = ["google", "github", "slack", "notion", "mcp"] as const;
export type GatekeeperKey = (typeof GATEKEEPER_KEYS)[number];

export interface GatekeeperPackage {
  /** Package directory, relative to the repository root. */
  dir: string;
  /** pnpm package name inside the submodule workspace, for `vp run -F`. */
  pkg: string;
  /** Worker secrets the Gatekeeper reads. Empty when it needs no OAuth app of its own. */
  secrets: readonly string[];
  /** OAuth redirect path under the public origin, or null when the vendor registers nothing. */
  redirectPath: string | null;
}

const oauthSecrets = ["CLIENT_ID", "CLIENT_SECRET"] as const;

export const GATEKEEPERS: Record<GatekeeperKey, GatekeeperPackage> = {
  google: {
    dir: "cloudflare-os/packages/gatekeeper-google",
    pkg: "@gadgets/google-gatekeeper",
    secrets: oauthSecrets,
    redirectPath: "/gatekeeper/google/oauth",
  },
  github: {
    dir: "cloudflare-os/packages/gatekeeper-github",
    pkg: "@gadgets/github-gatekeeper",
    secrets: oauthSecrets,
    redirectPath: "/gatekeeper/github/oauth",
  },
  slack: {
    dir: "cloudflare-os/packages/gatekeeper-slack",
    pkg: "@gadgets/slack-gatekeeper",
    secrets: oauthSecrets,
    redirectPath: "/gatekeeper/slack/oauth",
  },
  notion: {
    dir: "cloudflare-os/packages/gatekeeper-notion",
    pkg: "@gadgets/notion-gatekeeper",
    secrets: oauthSecrets,
    redirectPath: "/gatekeeper/notion/oauth",
  },
  mcp: {
    dir: "cloudflare-os/packages/gatekeeper-mcp",
    pkg: "@gadgets/mcp-gatekeeper",
    secrets: [],
    redirectPath: null,
  },
};

/** The router maps `GATEKEEPER_<NAME>` to `/gatekeeper/<name>`; keep the two in lockstep here. */
export function bindingName(key: GatekeeperKey): string {
  return `GATEKEEPER_${key.toUpperCase()}`;
}

/** Configured Gatekeepers, in `GATEKEEPER_KEYS` order so generated output is stable. */
export function enabledGatekeepers(config: DeploymentConfig): GatekeeperKey[] {
  const configured = config.gatekeepers ?? {};
  return GATEKEEPER_KEYS.filter((key) => configured[key] !== undefined);
}
```

- [ ] **Step 4: `scripts/deployment-config.ts` に型を追加する**

`DeploymentConfig` の `customGatekeeper` の直後に追加:

```ts
  /**
   * Upstream vendor Gatekeepers to deploy alongside the starter's three. A key that is absent is
   * not deployed. Keys are the upstream package suffixes (`gatekeeper-<key>`).
   */
  gatekeepers?: GatekeepersConfig;
```

ファイル冒頭の import の下に追加:

```ts
import type { GatekeeperKey } from "./gatekeepers.ts";

/** One deployed vendor Gatekeeper. */
export interface GatekeeperConfig {
  /** Permanent Worker name. Unique within the account. */
  name: string;
}

/** The MCP Gatekeeper additionally names itself to the servers it connects to. */
export interface McpGatekeeperConfig extends GatekeeperConfig {
  /** Sent in MCP `initialize` and dynamic client registration. Defaults to "PIAZZA OS". */
  clientName?: string;
}

export type GatekeepersConfig =
  Partial<Record<Exclude<GatekeeperKey, "mcp">, GatekeeperConfig>> & { mcp?: McpGatekeeperConfig };
```

`GeneratedConfigs` と `BaseConfigs` の両方に追加:

```ts
  /** Vendor Gatekeepers, keyed like `deployment.jsonc`'s `gatekeepers`. Only enabled keys. */
  gatekeepers: Partial<Record<GatekeeperKey, ProdWranglerConfig>>;
```

- [ ] **Step 5: テストが通ることを確認する**

Run: `node --test scripts/gatekeepers.test.ts && pnpm types:scripts`
Expected: PASS。型チェックでは `deploy.ts` が `GeneratedConfigs.gatekeepers` を返していないエラーが出る場合がある。その場合は Task 4 まで一時的に `gatekeepers: {}` を `generateConfigs` の戻り値に足しておく。

- [ ] **Step 6: コミット**

```bash
git add scripts/gatekeepers.ts scripts/gatekeepers.test.ts scripts/deployment-config.ts scripts/deploy.ts
git commit -m "feat: describe deployable vendor gatekeepers and their deployment config

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 3: `validateConfig` に Gatekeeper の検証を足す

**Files:**
- Modify: `scripts/deploy.ts`（`validateConfig`、Worker 名の一意性チェック付近）
- Test: `scripts/deploy.test.ts`

**Interfaces:**
- Consumes: `GATEKEEPER_KEYS`, `enabledGatekeepers`（Task 2）
- Produces: `validateConfig` が `gatekeepers` の不正を投げる。

- [ ] **Step 1: 失敗するテストを書く**

`scripts/deploy.test.ts` の `validConfig` に `gatekeepers` を追加する（`customGatekeeper` の直後）:

```ts
  gatekeepers: {
    google: { name: "acme-cloudflare-os-gk-google" },
    github: { name: "acme-cloudflare-os-gk-github" },
    slack: { name: "acme-cloudflare-os-gk-slack" },
    notion: { name: "acme-cloudflare-os-gk-notion" },
    mcp: { name: "acme-cloudflare-os-gk-mcp", clientName: "Acme OS" },
  },
```

テストを追加:

```ts
test("rejects malformed gatekeeper configuration", () => {
  assert.throws(
    () => validateConfig(variant((c) => { c.gatekeepers.google.name = c.workers.workshop.name; })),
    /unique/i);
  assert.throws(
    () => validateConfig(variant((c) => { c.gatekeepers.slack = { name: "Bad_Name" }; })),
    /lowercase/i);
  assert.throws(
    () => validateConfig(variant((c) => { c.gatekeepers.linear = { name: "acme-linear" }; })),
    /unknown gatekeeper/i);
  assert.throws(
    () => validateConfig(variant((c) => { c.gatekeepers.notion = {}; })),
    /gatekeepers\.notion\.name/);
  assert.throws(
    () => validateConfig(variant((c) => { c.gatekeepers.mcp.clientName = "  "; })),
    /clientName/);
  assert.throws(
    () => validateConfig(variant((c) => { c.gatekeepers = []; })),
    /gatekeepers must be an object/i);
});

test("accepts a deployment with no vendor gatekeepers", () => {
  assert.doesNotThrow(() => validateConfig(variant((c) => { delete c.gatekeepers; })));
});
```

- [ ] **Step 2: テストが失敗することを確認する**

Run: `node --test scripts/deploy.test.ts`
Expected: 上の2テストが FAIL（未知のキーや `{}` が通ってしまう）。

- [ ] **Step 3: `validateConfig` を拡張する**

`deploy.ts` の import に追加:

```ts
import { GATEKEEPER_KEYS, GATEKEEPERS, bindingName, enabledGatekeepers } from "./gatekeepers.ts";
import type { GatekeeperKey } from "./gatekeepers.ts";
```

`validateConfig` 内、`const workerNames = ...` の直前に追加:

```ts
  const gatekeepers = config.gatekeepers;
  if (gatekeepers !== undefined &&
      (gatekeepers === null || typeof gatekeepers !== "object" || Array.isArray(gatekeepers))) {
    throw new Error("gatekeepers must be an object keyed by gatekeeper name.");
  }
  for (const key of Object.keys(gatekeepers ?? {})) {
    if (!(GATEKEEPER_KEYS as readonly string[]).includes(key)) {
      throw new Error(
        `Unknown gatekeeper "${key}". Deployable gatekeepers: ${GATEKEEPER_KEYS.join(", ")}.`);
    }
    const entry = (gatekeepers as Record<string, { name?: unknown; clientName?: unknown }>)[key];
    if (!entry || typeof entry !== "object" || typeof entry.name !== "string" || !entry.name) {
      throw new Error(`Missing required deployment value: gatekeepers.${key}.name`);
    }
    if (key === "mcp" && entry.clientName !== undefined &&
        (typeof entry.clientName !== "string" || !entry.clientName.trim())) {
      throw new Error("gatekeepers.mcp.clientName must be omitted or a non-blank string.");
    }
  }
```

Worker 名の収集と重複チェックを次のように置き換える:

```ts
  const workerNames = [
    ...Object.entries(config.workers)
      .filter(([key]) => key !== "errorReporter" || config.errorReporting.enabled)
      .map(([, worker]) => worker.name),
    ...enabledGatekeepers(config).map((key) => config.gatekeepers![key]!.name),
  ];
  if (new Set(workerNames).size !== workerNames.length) {
    throw new Error(
      "Router, Workshop, Context, Scheduler, custom Gatekeeper, and vendor Gatekeeper names " +
      "must be unique.");
  }
```

既存の小文字チェック（`workerNames.every(...)`）はそのまま Gatekeeper 名にも効く。

- [ ] **Step 4: テストが通ることを確認する**

Run: `node --test scripts/deploy.test.ts && pnpm types:scripts`
Expected: PASS

- [ ] **Step 5: コミット**

```bash
git add scripts/deploy.ts scripts/deploy.test.ts
git commit -m "feat: validate vendor gatekeeper deployment configuration

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 4: `generateConfigs` で Gatekeeper の config と binding を生成する

**Files:**
- Modify: `scripts/deploy.ts`（`generateConfigs`）
- Test: `scripts/deploy.test.ts`

**Interfaces:**
- Consumes: `GATEKEEPERS`, `bindingName`, `enabledGatekeepers`, `BaseConfigs.gatekeepers`
- Produces: `GeneratedConfigs.gatekeepers[key]`（`name`, `account_id`, `workers_dev: false`, `preview_urls: false`, `vars.BASE_URL`）、workshop / router の `services` 追加分。

- [ ] **Step 1: テストの `baseConfigs()` を拡張する**

`scripts/deploy.test.ts` の `baseConfigs()` の戻り値に追加:

```ts
    gatekeepers: {
      google: await baseConfig("../cloudflare-os/packages/gatekeeper-google/wrangler.jsonc"),
      github: await baseConfig("../cloudflare-os/packages/gatekeeper-github/wrangler.jsonc"),
      slack: await baseConfig("../cloudflare-os/packages/gatekeeper-slack/wrangler.jsonc"),
      notion: await baseConfig("../cloudflare-os/packages/gatekeeper-notion/wrangler.jsonc"),
      mcp: await baseConfig("../cloudflare-os/packages/gatekeeper-mcp/wrangler.jsonc"),
    },
```

- [ ] **Step 2: 失敗するテストを書く**

```ts
test("generates vendor Gatekeeper configs bound to the Workshop and the router", async () => {
  const generated = generateConfigs(validConfig, await baseConfigs());
  const google = generated.gatekeepers.google!;
  assert.equal(google.name, "acme-cloudflare-os-gk-google");
  assert.equal(google.account_id, validConfig.accountId);
  assert.equal(google.workers_dev, false);
  assert.equal(google.preview_urls, false);
  assert.equal(google.vars!.BASE_URL, "https://os.example.com/gatekeeper/google");
  // Upstream's own vars survive; only BASE_URL (and MCP_CLIENT_NAME) are added.
  assert.equal(generated.gatekeepers.mcp!.vars!.MCP_ALLOW_INSECURE, "false");
  assert.equal(generated.gatekeepers.mcp!.vars!.MCP_CLIENT_NAME, "Acme OS");
  assert.equal(generated.gatekeepers.slack!.vars!.MCP_CLIENT_NAME, undefined);

  for (const key of ["google", "github", "slack", "notion", "mcp"] as const) {
    const binding = `GATEKEEPER_${key.toUpperCase()}`;
    const name = validConfig.gatekeepers![key]!.name;
    assert.deepEqual(
      generated.workshop.services!.find((s) => s.binding === binding),
      { binding, service: name, entrypoint: "GatekeeperVendor" });
    assert.deepEqual(
      generated.router.services!.find((s) => s.binding === binding),
      { binding, service: name });
  }
});

test("defaults the MCP client name and skips unconfigured gatekeepers", async () => {
  const config = variant((c) => { c.gatekeepers = { mcp: { name: "acme-mcp" } }; });
  const generated = generateConfigs(config, await baseConfigs());
  assert.deepEqual(Object.keys(generated.gatekeepers), ["mcp"]);
  assert.equal(generated.gatekeepers.mcp!.vars!.MCP_CLIENT_NAME, "PIAZZA OS");
  assert.equal(generated.workshop.services!.some((s) => s.binding === "GATEKEEPER_GOOGLE"), false);
  assert.equal(generated.router.services!.some((s) => s.binding === "GATEKEEPER_GOOGLE"), false);
});

test("matches the starter's output when no vendor gatekeeper is configured", async () => {
  const bases = await baseConfigs();
  const generated = generateConfigs(variant((c) => { delete c.gatekeepers; }), bases);
  assert.deepEqual(generated.gatekeepers, {});
  assert.deepEqual(
    generated.router.services!.map((s) => s.binding),
    ["WORKSHOP_BACKEND", "GATEKEEPER_CONTEXT", "GATEKEEPER_SCHEDULER", "GATEKEEPER_CUSTOM"]);
});

test("refuses to generate a gatekeeper whose base config was not read", async () => {
  const bases = await baseConfigs();
  delete bases.gatekeepers.slack;
  assert.throws(() => generateConfigs(validConfig, bases), /slack/);
});
```

- [ ] **Step 3: テストが失敗することを確認する**

Run: `node --test scripts/deploy.test.ts`
Expected: 新規4テストが FAIL。

- [ ] **Step 4: `generateConfigs` を拡張する**

`generateConfigs` 内、`router.services = [...]` と `workshop.services = [...]` が設定された後
（`workshop.kv_namespaces = ...` の直前）に追加:

```ts
  const gatekeepers: GeneratedConfigs["gatekeepers"] = {};
  for (const key of enabledGatekeepers(config)) {
    const base = bases.gatekeepers[key];
    if (!base) {
      throw new Error(`No base wrangler config was read for gatekeeper "${key}".`);
    }
    const entry = config.gatekeepers![key]!;
    const generated = structuredClone(base);
    setCommon(generated, config, entry.name);
    // Upstream's wrangler.jsonc carries no BASE_URL: the deploy owns the public origin, so it is
    // the only place that can say where /gatekeeper/<key>/oauth answers.
    generated.vars = {
      ...generated.vars,
      BASE_URL: `${origin}/gatekeeper/${key}`,
      ...(key === "mcp"
        ? { MCP_CLIENT_NAME: (entry as { clientName?: string }).clientName ?? "PIAZZA OS" }
        : {}),
    };
    gatekeepers[key] = generated;
    router.services!.push({ binding: bindingName(key), service: entry.name });
    workshop.services!.push({
      binding: bindingName(key),
      service: entry.name,
      entrypoint: "GatekeeperVendor",
    });
  }
```

戻り値に `gatekeepers` を含める:

```ts
  return {
    router, workshop, context, scheduler, customGatekeeper, gatekeepers,
    ...(errorReporter && { errorReporter }),
  };
```

- [ ] **Step 5: テストが通ることを確認する**

Run: `node --test scripts/deploy.test.ts && pnpm types:scripts`
Expected: PASS（starter 由来の既存テストも含めて全件）。

- [ ] **Step 6: コミット**

```bash
git add scripts/deploy.ts scripts/deploy.test.ts
git commit -m "feat: generate vendor gatekeeper configs and bind them to the Workshop and router

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 5: ビルド順・デプロイ順に Gatekeeper を組み込む

**Files:**
- Modify: `scripts/deploy.ts`（`buildCommands`, `main`, `packageDirs` / `generatedPaths`）
- Test: `scripts/deploy.test.ts`

**Interfaces:**
- Consumes: `GATEKEEPERS[key].dir` / `.pkg`
- Produces: `buildCommands(config)` に各 Gatekeeper の `build:configurator` が入る。`main()` が Gatekeeper の `wrangler.prod.jsonc` を書き、デプロイし、終了時に削除する。

- [ ] **Step 1: 失敗するテストを書く**

```ts
test("builds each vendor Gatekeeper's configurator before the Workshop and the router", () => {
  const commands = buildCommands(validConfig).map((c) => c.args.join(" "));
  const configurator = (pkg: string) =>
    commands.indexOf(`--dir cloudflare-os exec vp run -F ${pkg} --no-cache build:configurator`);
  const workshop = commands.indexOf("--dir cloudflare-os exec vp run -F @gadgets/workshop-backend --no-cache build");
  for (const pkg of [
    "@gadgets/google-gatekeeper", "@gadgets/github-gatekeeper", "@gadgets/slack-gatekeeper",
    "@gadgets/notion-gatekeeper", "@gadgets/mcp-gatekeeper",
  ]) {
    assert.notEqual(configurator(pkg), -1, `${pkg} is built`);
    assert.ok(configurator(pkg) < workshop, `${pkg} is built before the Workshop`);
  }
});

test("builds nothing extra when no vendor gatekeeper is configured", () => {
  const commands = buildCommands(variant((c) => { delete c.gatekeepers; }))
    .map((c) => c.args.join(" "));
  assert.equal(commands.some((c) => c.includes("-gatekeeper --no-cache build:configurator")), false);
});
```

- [ ] **Step 2: テストが失敗することを確認する**

Run: `node --test scripts/deploy.test.ts`
Expected: 1つ目が FAIL（configurator のコマンドが無い）。

- [ ] **Step 3: `buildCommands` を拡張する**

`buildCommands` 内、`{ args: ownBuild("custom-gatekeeper") }` の直後に追加:

```ts
    // Vendor Gatekeepers: upstream's own `deploy` script is `vp run --no-cache build:configurator
    // && wrangler deploy`, and wrangler's `build.command` (capnweb-validate) runs at deploy time.
    // The configurator step writes `src/generated/`, which that command reads, so it is the one
    // build this script has to run itself.
    ...enabledGatekeepers(config).map((key) => ({
      args: submoduleBuild(GATEKEEPERS[key].pkg, "build:configurator"),
    })),
```

- [ ] **Step 4: `main()` で Gatekeeper の生成ファイルを書き、デプロイし、削除する**

`generatedPaths` は固定キーで作られているので、Gatekeeper 用のパスを別に持つ。`main()` を次のように変更する:

```ts
async function main(): Promise<void> {
  requireSubmodule();
  const config = await readDeployment(join(root, "deployment.jsonc"));
  const gatekeeperKeys = enabledGatekeepers(config);
  const gatekeeperBases: BaseConfigs["gatekeepers"] = {};
  for (const key of gatekeeperKeys) {
    gatekeeperBases[key] = await readJsonc(join(root, GATEKEEPERS[key].dir, "wrangler.jsonc"));
  }
  const generated = generateConfigs(config, {
    router: await readJsonc(join(root, packageDirs.router, "wrangler.jsonc")),
    workshop: await readJsonc(join(root, packageDirs.workshop, "wrangler.jsonc")),
    context: await readJsonc(join(root, packageDirs.context, "wrangler.jsonc")),
    scheduler: await readJsonc(join(root, packageDirs.scheduler, "wrangler.jsonc")),
    customGatekeeper: await readJsonc(join(root, packageDirs.customGatekeeper, "wrangler.jsonc")),
    errorReporter: await readJsonc(join(root, packageDirs.errorReporter, "wrangler.jsonc")),
    gatekeepers: gatekeeperBases,
  });
  reportAiGateway(config);
  reportGatekeepers(config);

  const gatekeeperPaths = Object.fromEntries(
    gatekeeperKeys.map((key) => [key, join(root, GATEKEEPERS[key].dir, generatedName)]),
  ) as Partial<Record<GatekeeperKey, string>>;
  const allGeneratedPaths = [...Object.values(generatedPaths), ...Object.values(gatekeeperPaths)];

  try {
    const { gatekeepers, ...fixed } = generated;
    for (const [name, generatedConfig] of Object.entries(fixed)) {
      await writeFile(
        generatedPaths[name as keyof typeof generatedPaths],
        JSON.stringify(generatedConfig, null, 2) + "\n");
    }
    for (const key of gatekeeperKeys) {
      await writeFile(gatekeeperPaths[key]!, JSON.stringify(gatekeepers[key], null, 2) + "\n");
    }
    const check = process.argv.includes("--check");
    if (check) run(["test"]);
    build(config);
    const deployArgs = check ? ["--dry-run"] : [];
    if (config.errorReporting.enabled) {
      deployWorker(packageDirs.errorReporter, deployArgs);
    }
    deployWorker(packageDirs.context, deployArgs);
    deployWorker(packageDirs.scheduler, deployArgs);
    deployWorker(packageDirs.customGatekeeper, deployArgs);
    for (const key of gatekeeperKeys) {
      deployWorker(GATEKEEPERS[key].dir, deployArgs);
    }
    deployWorker(packageDirs.workshop, deployArgs);
    // Last: it binds every one of the above.
    deployWorker(packageDirs.router, deployArgs);
  } finally {
    await Promise.all(allGeneratedPaths.map((path) => rm(path, { force: true })));
  }
}
```

`reportGatekeepers` は Task 6 で実装する。このタスクの時点では空関数を置いておく:

```ts
function reportGatekeepers(_config: DeploymentConfig): void {}
```

- [ ] **Step 5: テストと型チェック**

Run: `node --test scripts/deploy.test.ts && pnpm types:scripts`
Expected: PASS

- [ ] **Step 6: dry-run で生成ファイルの往復を確認する**

`deployment.jsonc` はまだ starter のプレースホルダーなので `pnpm check` は
「Replace deployment placeholder」で止まる。ここでは一時的に `deployment.jsonc` を
`scripts/deploy.test.ts` の `validConfig` 相当の値で埋めたコピーを使い、生成と削除だけ確認する:

```bash
cp deployment.jsonc /tmp/deployment.backup.jsonc
node -e '
const fs=require("fs");
const c={accountId:"0123456789abcdef0123456789abcdef",publicBaseUrl:null,
workers:{router:{name:"t-os",route:{customDomain:"os.example.com"}},workshop:{name:"t-workshop"},context:{name:"t-context"},scheduler:{name:"t-scheduler"},customGatekeeper:{name:"t-custom"},errorReporter:{name:"t-errors"}},
access:{issuer:"https://t.cloudflareaccess.com",audience:"aud",admins:["a@example.com"]},
aiGateway:{enabled:true,name:"default",accountId:null,providers:["cloudflare"]},
context:{sharingDomain:null,kvNamespaceId:null},
customGatekeeper:{name:"T",message:"m"},
gatekeepers:{google:{name:"t-gk-google"},mcp:{name:"t-gk-mcp"}},
errorReporting:{enabled:true,environment:"production",release:null},
resources:{blueprintsKvNamespaceId:null,avatarsKvNamespaceId:null,blueprintContentBucket:null},
observability:{enabled:true,headSamplingRate:1,logs:{invocationLogs:false},traces:{enabled:false,headSamplingRate:0.1}}};
fs.writeFileSync("deployment.jsonc", JSON.stringify(c,null,2));'
pnpm check; echo "exit=$?"
ls cloudflare-os/packages/gatekeeper-google/wrangler.prod.jsonc 2>&1
cp /tmp/deployment.backup.jsonc deployment.jsonc
```

期待: `pnpm check` が dry-run まで完走し（`account_id` が架空なので wrangler の dry-run が
アカウント確認で失敗する場合は、それ以前の build まで通っていることを出力で確認する）、
終了後に `wrangler.prod.jsonc` が残っていない（`ls` が "No such file"）。

- [ ] **Step 7: コミット**

```bash
git add scripts/deploy.ts scripts/deploy.test.ts
git commit -m "feat: build and deploy vendor gatekeepers between the ambient gatekeepers and the Workshop

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 6: `pnpm check` にリダイレクト URI と secret の一覧を出す

**Files:**
- Modify: `scripts/deploy.ts`（`redirectUris`, `reportGatekeepers`）
- Test: `scripts/deploy.test.ts`

**Interfaces:**
- Produces: `export function redirectUris(config: DeploymentConfig): { key: GatekeeperKey; worker: string; redirectUri: string | null; secrets: readonly string[] }[]`

- [ ] **Step 1: 失敗するテストを書く**

```ts
test("lists the OAuth redirect URIs and secrets each vendor Gatekeeper needs", () => {
  const rows = redirectUris(validConfig);
  assert.deepEqual(rows.map((r) => r.key), ["google", "github", "slack", "notion", "mcp"]);
  assert.deepEqual(rows[0], {
    key: "google",
    worker: "acme-cloudflare-os-gk-google",
    redirectUri: "https://os.example.com/gatekeeper/google/oauth",
    secrets: ["CLIENT_ID", "CLIENT_SECRET"],
  });
  assert.deepEqual(rows[4], {
    key: "mcp", worker: "acme-cloudflare-os-gk-mcp", redirectUri: null, secrets: [],
  });
});
```

import に `redirectUris` を追加する。

- [ ] **Step 2: テストが失敗することを確認する**

Run: `node --test scripts/deploy.test.ts`
Expected: FAIL（`redirectUris` が export されていない）

- [ ] **Step 3: 実装する**

`deploy.ts` の `reportAiGateway` の近くに追加:

```ts
/** What has to be registered at each vendor before its Gatekeeper can connect anyone. */
export function redirectUris(config: DeploymentConfig): {
  key: GatekeeperKey;
  worker: string;
  redirectUri: string | null;
  secrets: readonly string[];
}[] {
  const origin = publicOrigin(config);
  return enabledGatekeepers(config).map((key) => {
    const { redirectPath, secrets } = GATEKEEPERS[key];
    return {
      key,
      worker: config.gatekeepers![key]!.name,
      redirectUri: redirectPath ? `${origin}${redirectPath}` : null,
      secrets,
    };
  });
}

// Said once, up front: the OAuth apps are registered by hand, and a wrong redirect URI only shows
// up as a vendor error page in somebody's first connect attempt.
function reportGatekeepers(config: DeploymentConfig): void {
  const rows = redirectUris(config);
  if (!rows.length) return;
  console.warn("\nVendor Gatekeepers in this deployment:");
  for (const row of rows) {
    console.warn(`  - ${row.key} (${row.worker})`);
    if (row.redirectUri) console.warn(`      redirect URI to register: ${row.redirectUri}`);
    if (row.secrets.length) {
      console.warn(`      secrets: ${row.secrets.join(", ")} -- install with: pnpm secrets`);
    }
  }
}
```

Task 5 で置いた空の `reportGatekeepers` は削除する。

- [ ] **Step 4: テストが通ることを確認する**

Run: `node --test scripts/deploy.test.ts && pnpm types:scripts`
Expected: PASS

- [ ] **Step 5: コミット**

```bash
git add scripts/deploy.ts scripts/deploy.test.ts
git commit -m "feat: report vendor gatekeeper redirect URIs and secrets during check

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 7: `pnpm secrets` で Worker secret を投入する

**Files:**
- Create: `scripts/secrets.ts`, `scripts/secrets.test.ts`, `secrets.local.example.jsonc`
- Modify: `.gitignore`, `package.json`

**Interfaces:**
- Consumes: `GATEKEEPERS`, `enabledGatekeepers`, `readDeployment`（`deploy.ts` から export する）
- Produces: `export function planSecretUploads(config: DeploymentConfig, file: Record<string, Record<string, string>>): { worker: string; values: Record<string, string> }[]`（純粋関数。テスト対象）と CLI 本体。

- [ ] **Step 1: 失敗するテストを書く**

`scripts/secrets.test.ts`:

```ts
import assert from "node:assert/strict";
import test from "node:test";
import { planSecretUploads } from "./secrets.ts";
import type { DeploymentConfig } from "./deployment-config.ts";

const config = {
  gatekeepers: {
    google: { name: "acme-gk-google" },
    slack: { name: "acme-gk-slack" },
    mcp: { name: "acme-gk-mcp" },
  },
} as unknown as DeploymentConfig;

test("plans one bulk upload per gatekeeper worker with every required secret", () => {
  const plan = planSecretUploads(config, {
    "acme-gk-google": { CLIENT_ID: "g-id", CLIENT_SECRET: "g-secret" },
    "acme-gk-slack": { CLIENT_ID: "s-id", CLIENT_SECRET: "s-secret" },
  });
  assert.deepEqual(plan, [
    { worker: "acme-gk-google", values: { CLIENT_ID: "g-id", CLIENT_SECRET: "g-secret" } },
    { worker: "acme-gk-slack", values: { CLIENT_ID: "s-id", CLIENT_SECRET: "s-secret" } },
  ]);
});

test("rejects a worker the deployment does not know", () => {
  assert.throws(
    () => planSecretUploads(config, { "someone-else": { CLIENT_ID: "x", CLIENT_SECRET: "y" } }),
    /someone-else/);
});

test("rejects a missing or blank required secret", () => {
  assert.throws(
    () => planSecretUploads(config, { "acme-gk-google": { CLIENT_ID: "g-id" } }),
    /CLIENT_SECRET/);
  assert.throws(
    () => planSecretUploads(config, { "acme-gk-google": { CLIENT_ID: " ", CLIENT_SECRET: "y" } }),
    /CLIENT_ID/);
});

test("rejects secrets for a gatekeeper that takes none", () => {
  assert.throws(
    () => planSecretUploads(config, { "acme-gk-mcp": { TOKEN: "x" } }),
    /acme-gk-mcp/);
});
```

- [ ] **Step 2: テストが失敗することを確認する**

Run: `node --test scripts/secrets.test.ts`
Expected: FAIL（`./secrets.ts` が無い）

- [ ] **Step 3: `deploy.ts` から `readDeployment` を export する**

`deploy.ts` の `async function readDeployment` を `export async function readDeployment` に変える。

- [ ] **Step 4: `scripts/secrets.ts` を書く**

```ts
// Installs vendor Gatekeeper secrets from a gitignored file, one `wrangler secret bulk` per
// Worker, values passed over stdin so they never reach a command line or a log.
import { existsSync } from "node:fs";
import { readFile } from "node:fs/promises";
import { spawnSync } from "node:child_process";
import { fileURLToPath, pathToFileURL } from "node:url";
import { dirname, join, resolve } from "node:path";
import { parse, type ParseError } from "jsonc-parser";
import { resolveBinEntry } from "../cloudflare-os/scripts/bin-entry.ts";
import { pnpmCommand } from "../cloudflare-os/scripts/pnpm-command.ts";
import { readDeployment } from "./deploy.ts";
import { GATEKEEPERS, enabledGatekeepers } from "./gatekeepers.ts";
import type { DeploymentConfig } from "./deployment-config.ts";

const root = resolve(dirname(fileURLToPath(import.meta.url)), "..");
const secretsFile = join(root, "secrets.local.jsonc");

export interface SecretUpload {
  worker: string;
  values: Record<string, string>;
}

/**
 * Pairs the file's entries with the deployment's Gatekeepers and checks both directions: every
 * listed Worker is one this deployment deploys, and every required secret is present and
 * non-blank. Order follows the file.
 */
export function planSecretUploads(
  config: DeploymentConfig,
  file: Record<string, Record<string, string>>,
): SecretUpload[] {
  const required = new Map<string, readonly string[]>();
  for (const key of enabledGatekeepers(config)) {
    required.set(config.gatekeepers![key]!.name, GATEKEEPERS[key].secrets);
  }
  const plan: SecretUpload[] = [];
  for (const [worker, values] of Object.entries(file)) {
    const secrets = required.get(worker);
    if (secrets === undefined) {
      throw new Error(
        `secrets.local.jsonc names Worker "${worker}", which is not a vendor Gatekeeper in ` +
        "deployment.jsonc.");
    }
    if (secrets.length === 0) {
      throw new Error(`Worker "${worker}" takes no secrets; remove it from secrets.local.jsonc.`);
    }
    if (!values || typeof values !== "object") {
      throw new Error(`secrets.local.jsonc entry for "${worker}" must be an object.`);
    }
    for (const name of secrets) {
      const value = values[name];
      if (typeof value !== "string" || !value.trim()) {
        throw new Error(`Worker "${worker}" is missing secret ${name}.`);
      }
    }
    for (const name of Object.keys(values)) {
      if (!secrets.includes(name)) {
        throw new Error(`Worker "${worker}" does not read a secret named ${name}.`);
      }
    }
    plan.push({ worker, values: { ...values } });
  }
  return plan;
}

function upload(config: DeploymentConfig, { worker, values }: SecretUpload): void {
  const args = ["secret", "bulk", "--name", worker];
  const env = { ...process.env, CLOUDFLARE_ACCOUNT_ID: config.accountId };
  const input = JSON.stringify(values);
  const entry = resolveBinEntry(root, "wrangler");
  const result = entry
    ? spawnSync(process.execPath, [entry, ...args], { cwd: root, env, input, stdio: ["pipe", "inherit", "inherit"] })
    : (() => {
        const [command, argv] = pnpmCommand(["exec", "wrangler", ...args], env);
        return spawnSync(command, argv, { cwd: root, env, input, stdio: ["pipe", "inherit", "inherit"] });
      })();
  if (result.error) throw result.error;
  if (result.status !== 0) {
    throw new Error(`wrangler secret bulk failed for ${worker}. Its output is above.`);
  }
}

async function main(): Promise<void> {
  if (!existsSync(secretsFile)) {
    throw new Error(
      "secrets.local.jsonc not found. Copy secrets.local.example.jsonc to secrets.local.jsonc " +
      "and fill in each Gatekeeper's OAuth client id and secret.");
  }
  const config = await readDeployment(join(root, "deployment.jsonc"));
  const errors: ParseError[] = [];
  const file = parse(await readFile(secretsFile, "utf8"), errors, { allowTrailingComma: true });
  if (errors.length) throw new Error("secrets.local.jsonc does not parse as JSONC.");
  const plan = planSecretUploads(config, file);
  if (!plan.length) {
    console.warn("secrets.local.jsonc lists no Workers; nothing to install.");
    return;
  }
  for (const item of plan) {
    console.warn(`Installing ${Object.keys(item.values).join(", ")} on ${item.worker}`);
    upload(config, item);
  }
}

if (process.argv[1] && import.meta.url === pathToFileURL(resolve(process.argv[1])).href) {
  try {
    await main();
  } catch (error) {
    console.error(`\nSecret install failed. ${(error as Error).message}`);
    process.exitCode = 1;
  }
}
```

- [ ] **Step 5: example ファイル、`.gitignore`、`package.json` を更新する**

`secrets.local.example.jsonc`:

```jsonc
{
  // Copy to secrets.local.jsonc (gitignored). Keys are Worker names from deployment.jsonc's
  // `gatekeepers`; values are that Gatekeeper's OAuth app credentials. Install with `pnpm secrets`.
  "piazza-os-gk-google": { "CLIENT_ID": "", "CLIENT_SECRET": "" },
  "piazza-os-gk-github": { "CLIENT_ID": "", "CLIENT_SECRET": "" },
  "piazza-os-gk-slack": { "CLIENT_ID": "", "CLIENT_SECRET": "" },
  "piazza-os-gk-notion": { "CLIENT_ID": "", "CLIENT_SECRET": "" }
}
```

`.gitignore` に1行追加:

```
secrets.local.jsonc
```

`package.json` の `scripts` に追加:

```json
    "secrets": "node scripts/secrets.ts"
```

- [ ] **Step 6: テストが通ることを確認する**

Run: `node --test scripts/secrets.test.ts && pnpm types:scripts && pnpm test`
Expected: PASS。`pnpm test` は `scripts/**/*.test.ts` を拾うので新テストも走る。

- [ ] **Step 7: コミット**

```bash
git add scripts/secrets.ts scripts/secrets.test.ts scripts/deploy.ts secrets.local.example.jsonc .gitignore package.json
git commit -m "feat: install vendor gatekeeper secrets from a gitignored file

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 8: PIAZZA の `deployment.jsonc`、手順書、README

**Files:**
- Modify: `deployment.jsonc`, `README.md`
- Create: `docs/setup-runbook.md`

**Interfaces:**
- Consumes: Task 6 の `pnpm check` 出力（リダイレクト URI）
- Produces: ユーザーが値を埋めれば `pnpm check` が通る設定と、外部登録の手順。

- [ ] **Step 1: `deployment.jsonc` を PIAZZA の値で書く**

アカウント ID と workers.dev サブドメインと Access の AUD はユーザーから受け取るまで
`<...>` プレースホルダーのままにする（`pnpm check` が埋め忘れを止める）。

```jsonc
{
  "accountId": "<PIAZZA_CLOUDFLARE_ACCOUNT_ID>",
  // workers.dev route: the account's workers.dev subdomain is shown in the dashboard under
  // Workers & Pages > Overview > "Your subdomain".
  "publicBaseUrl": "https://piazza-os.<PIAZZA_WORKERS_DEV_SUBDOMAIN>.workers.dev",
  "workers": {
    "router": { "name": "piazza-os", "route": { "workersDev": true } },
    "workshop": { "name": "piazza-os-workshop" },
    "context": { "name": "piazza-os-context" },
    "scheduler": { "name": "piazza-os-scheduler" },
    "customGatekeeper": { "name": "piazza-os-custom" },
    "errorReporter": { "name": "piazza-os-error-reporter" }
  },
  "gatekeepers": {
    "google": { "name": "piazza-os-gk-google" },
    "github": { "name": "piazza-os-gk-github" },
    "slack": { "name": "piazza-os-gk-slack" },
    "notion": { "name": "piazza-os-gk-notion" },
    "mcp": { "name": "piazza-os-gk-mcp", "clientName": "PIAZZA OS" }
  },
  "access": {
    "issuer": "https://<PIAZZA_ZERO_TRUST_TEAM>.cloudflareaccess.com",
    "audience": "<ACCESS_AUDIENCE>",
    "admins": ["hiromitsu.sasaki@piazza-life.com"]
  },
  "aiGateway": {
    "enabled": true,
    "name": "piazza-os",
    "accountId": null,
    "providers": ["anthropic", "cloudflare"]
  },
  "context": { "sharingDomain": null, "kvNamespaceId": null },
  "customGatekeeper": {
    "name": "PIAZZA",
    "message": "PIAZZA（ピアッツァ）の社内 AI 環境です。社内データは BigQuery プロジェクト piazza-1174、社内ドキュメントは Notion と Google Drive、共有ページは poi にあります。個人情報を含むデータは必要最小限だけ扱ってください。"
  },
  "errorReporting": { "enabled": true, "environment": "production", "release": null },
  "resources": {
    "blueprintsKvNamespaceId": null,
    "avatarsKvNamespaceId": null,
    "blueprintContentBucket": null
  },
  "observability": {
    "enabled": true,
    "headSamplingRate": 1,
    "logs": { "invocationLogs": false },
    "traces": { "enabled": false, "headSamplingRate": 0.1 }
  }
}
```

- [ ] **Step 2: `docs/setup-runbook.md` を書く**

以下を含める（すべて具体的な画面名と値で。`<origin>` は `publicBaseUrl` の値）:

1. **piazza-life アカウントでの wrangler 認証**
   `pnpm exec wrangler logout` の後 `pnpm exec wrangler login` を piazza-life.com の Google アカウントで行う。
   `pnpm exec wrangler whoami` に piazza-life のアカウントが出ることを確認し、Account ID を
   `deployment.jsonc` の `accountId` に書く。個人アカウント（promitsu.jp）が出たら別ブラウザプロファイルでやり直す。
2. **workers.dev サブドメイン**: Workers & Pages > Overview の "Your subdomain" を `publicBaseUrl` に反映。
3. **Zero Trust**: チーム作成（未作成なら）。Settings > Authentication > Login methods で Google を追加
   （Google Cloud で OAuth クライアントを作り、リダイレクト URI に
   `https://<team>.cloudflareaccess.com/cdn-cgi/access/callback` を登録）。One-time PIN は無効化。
4. **Access アプリ**（順番に注意。AUD は Worker があると作りやすい）:
   1. `deployment.jsonc` の `access.audience` を仮値 `pending` にして一度 `pnpm deploy`。
   2. Workers & Pages > `piazza-os` > Access タブ > "Protect this Worker behind Access" > All traffic。
      ポリシー: Allow, Include "Emails ending in" `@piazza-life.com`。
   3. Zero Trust > Access controls > Applications で作られたアプリを開き、AUD タグをコピーして
      `access.audience` に書き、もう一度 `pnpm deploy`。
   仮値の間はフロントは表示されるが API は全て 403 になる。
5. **Google OAuth クライアント**（Cloudflare OS 用。既存の poi/BigQuery 用とは別に作る）:
   Google Cloud プロジェクト `piazza-1174`（または新規）で APIs & Services > Library から
   Gmail / Google Docs / Google Drive / Google Sheets / Google Calendar / BigQuery を有効化。
   OAuth 同意画面は **Internal**。認証情報 > OAuth クライアント ID（Web application）、
   承認済みリダイレクト URI に `<origin>/gatekeeper/google/oauth`。
6. **GitHub OAuth App**（org `piazza-life` の Settings > Developer settings > OAuth Apps）:
   Authorization callback URL `<origin>/gatekeeper/github/oauth`。GitHub App ではなく OAuth App。
7. **Slack App**（api.slack.com/apps > Create New App > From scratch）:
   OAuth & Permissions > Redirect URLs に `<origin>/gatekeeper/slack/oauth`、Token Rotation を有効化。
   User Token Scopes: `team:read`, `channels:read`, `channels:history`, `groups:read`,
   `groups:history`, `im:read`, `im:history`, `mpim:read`, `mpim:history`, `search:read`, `users:read`。
8. **Notion**（notion.so/my-integrations > New integration > Public）:
   Redirect URI `<origin>/gatekeeper/notion/oauth`。
9. **AI Gateway**: AI > AI Gateway で `piazza-os` を作成。Provider Keys（BYOK）に Anthropic の
   組織 API キーを保存。Workers AI はキー不要。
10. **secret 投入**: `cp secrets.local.example.jsonc secrets.local.jsonc` → 各 CLIENT_ID / CLIENT_SECRET を
    記入 → `pnpm secrets`。
11. **デプロイ**: `pnpm check` → `pnpm deploy` → `pnpm secrets`（secret は Worker が存在してから）。
12. **検証チェックリスト**: spec 第9節の 1〜9 をそのまま列挙する。
13. **更新手順**: フォークに upstream を取り込む → `git -C cloudflare-os checkout <commit>` →
    catalog 同期（Task 1 Step 3 の diff コマンド）→ `pnpm install` と `pnpm --dir cloudflare-os install` →
    `pnpm check` → `pnpm deploy`。戻しは `wrangler rollback --name <worker>` か submodule を戻して再デプロイ。

- [ ] **Step 3: README に PIAZZA 向け導入を追記する**

README 先頭の `<h1>` の直後に、次の内容の短いセクションを日本語で追加する:
「このリポジトリは PIAZZA 向けの Cloudflare OS デプロイ。upstream は
`piazza-life/cloudflare-os`（submodule）。手順は `docs/setup-runbook.md`。starter からの差分は
`gatekeepers` セクション（vendor Gatekeeper の追加デプロイ）と `pnpm secrets`。」
既存の starter の README 本文はそのまま残す。

- [ ] **Step 4: 検証**

```bash
pnpm test
pnpm check 2>&1 | head -5
```

期待: `pnpm test` PASS。`pnpm check` は `Replace deployment placeholder <PIAZZA_CLOUDFLARE_ACCOUNT_ID>`
で止まる（プレースホルダー検出が効いていることの確認）。

- [ ] **Step 5: コミット**

```bash
git add deployment.jsonc docs/setup-runbook.md README.md
git commit -m "docs: PIAZZA deployment configuration and setup runbook

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 9: 初回デプロイと検証（ユーザー作業を含む）

**Files:**
- Modify: `deployment.jsonc`（実値）
- Create: `secrets.local.jsonc`（gitignore 済み、コミットしない）

**Interfaces:**
- Consumes: Task 8 の手順書。
- Produces: 稼働中のデプロイと、検証結果の記録（`docs/setup-runbook.md` 末尾に日付付きで追記）。

- [ ] **Step 1: ユーザーに手順書 1〜3 と 5〜9 の外部登録を依頼し、値を受け取る**

受け取るもの: アカウント ID、workers.dev サブドメイン、Zero Trust チーム名、各 OAuth アプリの
CLIENT_ID / CLIENT_SECRET（`secrets.local.jsonc` にユーザー自身が記入）。エージェントは平文の
secret を会話やコミットに載せない。

- [ ] **Step 2: `deployment.jsonc` に実値を入れ、`access.audience` は `pending` で一次デプロイ**

```bash
pnpm check
pnpm deploy
```

期待: 11 Worker が順にデプロイされ、最後に router の workers.dev URL が表示される。
失敗した場合はその Worker の出力を確認し、原因が上流の config か本リポの生成かを切り分ける
（生成結果は `pnpm check` 中に `packages/*/wrangler.prod.jsonc` として一時的に存在する）。

- [ ] **Step 3: Access を有効化し AUD を反映して二次デプロイ**

手順書 4 の 2〜3 をユーザーが実施。`access.audience` を更新して:

```bash
pnpm check && pnpm deploy && pnpm secrets
```

- [ ] **Step 4: 検証チェックリストを実行し結果を記録する**

spec 第9節の 1〜7 と 9 を順に確認（8 の poi は poi-mcp 側の対応後）。結果を
`docs/setup-runbook.md` 末尾の「検証記録」に日付と担当者付きで追記する。

- [ ] **Step 5: コミット**

```bash
git add deployment.jsonc docs/setup-runbook.md
git commit -m "chore: first PIAZZA deployment values and verification record

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

push と GitHub リポジトリ作成はユーザーの指示を待つ。

---

## セルフレビュー

- **Spec カバレッジ**: §4.1 Worker 一覧 → Task 4/5/8。§4.3 binding → Task 4。§4.4 vars/secrets → Task 4/7。
  §5 Access → Task 8 手順書 3〜4。§6 外部登録 → Task 8 手順書 5〜9。§7.2〜7.5 → Task 2〜7。
  §9 検証 → Task 9。§10 運用 → Task 8 手順書 13。§8 poi は別計画（本計画の対象外と明記）。
- **プレースホルダー**: `deployment.jsonc` の `<...>` は意図的（`pnpm check` が止める設計）。
  コード側に TBD/TODO はない。
- **型の整合**: `GeneratedConfigs.gatekeepers` / `BaseConfigs.gatekeepers` を Task 2 で定義し、
  Task 4/5 のコードとテストの `baseConfigs()` が同じキー名を使う。`redirectUris` の戻り値の
  形は Task 6 のテストと実装で一致。`readDeployment` の export は Task 7 Step 3 で行う。
