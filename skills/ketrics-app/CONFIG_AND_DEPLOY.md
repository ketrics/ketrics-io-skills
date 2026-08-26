# Configuration and Deployment

## ketrics.config.json

The config file at the project root declares the application to the Ketrics platform.

### Full schema

```json
{
  "name": "my-ketrics-app",
  "version": "1.0.0",
  "description": "Description of the application",
  "runtime": "nodejs18",
  "actions": [
    { "code": "read", "description": "View items and reports" },
    { "code": "write", "description": "Create and modify items" },
    { "code": "export", "description": "Generate Excel and PDF exports" },
    { "code": "approve", "description": "Authorise items above the approval threshold" }
  ],
  "functions": [
    "getPermissions",
    "listItems",
    "createItem"
  ],
  "roles": [
    { "code": "viewer", "name": "Viewer", "description": "Read-only access", "actions": ["read"] },
    { "code": "clerk", "name": "Clerk", "description": "Day-to-day data entry", "actions": ["read", "write", "export"] },
    { "code": "approver", "name": "Approver", "description": "Authorises items", "actions": ["read", "approve"] }
  ],
  "environment": [
    { "name": "STRIPE_API_KEY", "description": "Secret resource code for the Stripe API key" },
    { "name": "MAIN_DB_CONNECTION", "description": "SQL data connection code" }
  ],
  "entry": "dist/index.js",
  "include": ["dist/**/*"],
  "exclude": ["node_modules", "*.test.js", "*.spec.js"],
  "resources": {
    "documentdb": [
      {
        "code": "app-data",
        "description": "What this DocumentDB stores"
      }
    ],
    "volume": [
      {
        "code": "exports",
        "description": "What files this volume stores"
      }
    ],
    "secret": [
      {
        "code": "stripe-key",
        "description": "What this secret holds",
        "environmentVariable": "STRIPE_API_KEY"
      }
    ]
  }
}
```

The `documentdb` and `volume` entries above declare no `environmentVariable`: their names are
**derived** (`APP_DATA_DOCDB`, `EXPORTS_VOLUME`). The `secret` entry names one explicitly, so
`STRIPE_API_KEY` must also appear in `environment`. Both forms are valid — see
[Resource declarations](#resource-declarations--never-hardcode-resource-codes).

> **`environmentVariables` was renamed to `environment`.** This is not a deprecation: a config
> that still uses the old key **fails the deploy** with
> `ketrics.config.json: "environmentVariables" was renamed to "environment" — rename the key`.
> It is never silently ignored.

### Field descriptions

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Application identifier (lowercase, hyphens) |
| `version` | Yes | Semver version string |
| `description` | No | Human-readable description |
| `runtime` | Yes | Runtime environment. Use `"nodejs18"` |
| `actions` | Yes | The **capabilities** the app enforces (e.g. `read`, `write`, `export`, `approve`). Each entry is a bare string or `{ code, description }`. On deploy these **become the app's "Supported Actions"**. Max 50 |
| `functions` | No | Handler function names. SHOULD match the exports in `backend/src/index.ts`. Stored as metadata |
| `roles` | No | Application roles to create, as `{ code, name, description?, actions }`. Every action listed must be declared in `actions`. Max 20 |
| `environment` | No | Env vars the app needs, as `{ name, description }`. Missing ones are created (empty) on deploy. Max 50 |
| `entry` | Yes | Path to the built backend bundle (relative to `backend/`) |
| `include` | No | Glob patterns for files to include in deployment |
| `exclude` | No | Glob patterns for files to exclude |
| `resources` | No | Declares the DocumentDB, Volume and Secret resources the app uses. Each binds to an environment variable, declared or derived (see below) |

### What deploy does with this file

On `ketrics deploy` the CLI bundles `ketrics.config.json` into the deployment ZIP and the platform syncs the application record from it:

- **`actions`** replaces the app's Supported Actions (skipped for marketplace-installed apps). Any descriptions you wrote are stored alongside them and surface in the portal's role builders.
- **`roles`** — one application role is created per declared role, granting `<appCode>:<action>` for every action it lists. Creation is idempotent: redeploying does not duplicate roles that already exist.
- **`environment`** — every declared variable that does not already exist on the app is created with an **empty value**; existing values are never overwritten. Fill the real values in the Ketrics dashboard.
- **`resources`** are persisted with every binding resolved — the platform fills in each entry's `environmentVariable`, deriving it when the config omitted one, so nothing downstream re-derives it.
- **`functions`** are stored as metadata.

The outcome is recorded on the deployment itself, so a sync that fails is visible rather than silent; a failed sync does not fail the deployment.

### `actions` are capabilities, not roles and not handlers

`actions` is the list of **capabilities** the app enforces — the values that appear in `ketrics.requestor.applicationPermissions` and that `requirePermission` checks. Typical values: `read`, `write`, `export`, `approve`.

Two things `actions` is **not**:

- **Not handler names.** Handler names go in `functions`. One permission per handler means a tenant admin composing a role has to know the codebase, and every renamed handler breaks a role.
- **Not role names.** `["admin", "editor", "user"]` looks reasonable and is the older convention, but it collapses the `roles` block into a 1:1 restatement of `actions` and makes partial combinations impossible. The moment someone needs "can approve but not edit", a role-named action list forces you to invent a new action and change code. Capabilities are the atoms; roles are the combinations.

An entry may carry a description, and the two forms may be mixed in one array:

```jsonc
"actions": [
  { "code": "approve", "description": "Authorise stock adjustments over the threshold" },
  "read"
]
```

Write the description for the tenant admin who will compose roles from it — what the capability lets someone do, not what the code does. Descriptions live only in the config; never duplicate them into `permissions.ts`, because nothing keeps two copies honest.

### Functions sync rule

The `functions` array SHOULD list every exported handler name from `backend/src/index.ts`, including background handlers prefixed with `_`. It is metadata describing the app's callable surface.

```
backend/src/index.ts exports  <-->  ketrics.config.json functions
```

Keep these in sync when adding or removing handlers.

## Choosing what goes in the config

The tables above say what each field *is*. Deciding what to put in them is the harder part. Work
in this order: capabilities, then roles, then the code that enforces them, then storage.

### Choosing capabilities for `actions`

Pick a handful of verbs that describe **what the app enforces**, not what it contains. Three to
six is the normal range:

| App | `actions` |
|---|---|
| Invoicing | `read`, `write`, `export`, `approve` |
| Inventory | `read`, `write`, `adjust_stock` |
| Read-only dashboard | `read`, `export` |

A capability earns its place when both are true:

- **The code enforces it.** At least one handler calls `requirePermission` with it. A capability
  no handler checks grants nothing — it is decoration in the role picker.
- **It is stable.** Adding a feature should usually add handlers, not capabilities. If every new
  screen needs a new action, the list is modelling features rather than permissions.

### New capability, or new role?

Ask what the *code* has to check:

- **A handler must behave differently → new capability.** If you would write
  `requirePermission("…")` with a code that does not exist yet, that is a real new capability.
  Guarding "post to the ledger" with `approve` rather than `write` is the usual case: the app
  genuinely needs to let someone edit records without letting them post.
- **Only the audience changes → new role.** "Warehouse staff read and write but never export"
  needs no code change: `read`, `write` and `export` already exist, so add a role granting the
  first two. Reorganising who-can-do-what is a config edit, not a deploy of new logic.

Reach for a new capability only when a guard needs it. Everything else is a role.

### Designing `roles` over those capabilities

A role is a named bundle of capabilities matching a job someone does. Name it after the job
(`viewer`, `clerk`, `approver`), never after the capability.

```json
"actions": [
  { "code": "read", "description": "View invoices and customers" },
  { "code": "write", "description": "Create and modify invoices" },
  { "code": "export", "description": "Generate Excel and PDF exports" },
  { "code": "approve", "description": "Authorise invoices above the approval threshold" }
],
"roles": [
  { "code": "viewer", "name": "Viewer", "actions": ["read"] },
  { "code": "clerk", "name": "Clerk", "actions": ["read", "write", "export"] },
  { "code": "approver", "name": "Approver", "actions": ["read", "approve"] }
]
```

The overlaps are the point:

- **`read` is shared by all three roles.** Capabilities are meant to be reused across roles —
  that reuse is what makes them composable. A "read-only role" is not a platform concept; it is
  simply a role whose only capability is `read`.
- **`approver` holds `approve` without `write`.** Someone can authorise an invoice without being
  able to edit it — a separation of duties that role-shaped actions cannot express.
- **Nobody holds all four.** Full access comes from the `*` wildcard, not from a role listing
  everything.

Validation: `code` unique and matching `/^[a-zA-Z][a-zA-Z0-9_-]*$/`, non-empty `name`, and every
entry in `roles[].actions` declared in the top-level `actions` — a role naming an undeclared
capability fails the deploy. Max 20 roles.

**Declared roles fully replace the old behaviour** where the platform generated one role per
action. If you omit `roles`, no per-user application roles are created at all — only the
APPLICATION resource role — and the tenant admin is left with capabilities and nothing to assign.
Declare the roles you want.

### Keeping the config and the code in agreement

The capabilities in `actions` and the `PERMISSIONS` tuple in `backend/src/permissions.ts` are two
lists that must match, and **nothing verifies them**. The typed guard catches typos *inside* the
code; it cannot catch a capability added to one file and forgotten in the other:

- in the config but not in `PERMISSIONS` → no handler can use it;
- in `PERMISSIONS` but not in the config → no role can grant it, so every handler guarding it
  throws for everyone.

Edit both in the same commit, and read the two lists side by side before deploying. See
[BACKEND_REFERENCE.md](BACKEND_REFERENCE.md) for the guard itself.

### Resource declarations — never hardcode resource codes

Each `resources` entry declares a resource the app depends on. An entry has three fields:

- **`code`** — a **logical identifier** for the resource slot (e.g. `app-data`, `exports`, `stripe-key`). It is a stable label for the declaration — **not** the platform resource code, and not an environment variable name. It must be unique within its kind.
- **`description`** — optional; what the resource is for.
- **`environmentVariable`** — **optional**; the name of an environment variable that holds the real resource code at runtime. If you name one, it must be declared in `environment` (deploy validation rejects an unknown reference). If you omit it, the name is **derived** from the kind and code.

The real resource code is never written in the config or in handler source. It reaches the app through the environment variable: in the Ketrics dashboard, the application's Update form shows a **resource picker** for every declared resource, letting an admin select an existing resource of the matching type — the chosen code is saved into the bound environment variable.

Why: resource codes differ per tenant and per environment. A hardcoded code ties the bundle to one tenant's infrastructure and breaks on deploy to another.

#### Derived environment variable names

When `environmentVariable` is omitted, the platform derives it: uppercase the `code`, replace every
character outside `A-Z0-9` with `_`, collapse runs of `_` and trim the edges, prefix `X_` if the
result would start with a digit, then append the kind's suffix.

| Kind | `code` | Derived variable |
|------|--------|------------------|
| `documentdb` | `app-data` | `APP_DATA_DOCDB` |
| `volume` | `exports` | `EXPORTS_VOLUME` |
| `secret` | `stripe-key` | `STRIPE_KEY_SECRET` |

A derived name needs **no** entry in `environment` — it is implicit, and it is created and made
mandatory exactly like a declared one. Name a variable explicitly only when you want something the
derivation would not produce; if a `code` cannot be mapped to a valid variable name at all, the
deploy fails and asks for an explicit `environmentVariable`.

Whichever form you use, the pattern in handlers is the same:

1. Declare the resource under its kind, with or without an explicit `environmentVariable`.
2. In handlers, read the code via `ketrics.environment["VAR_NAME"]` — never a string literal.

Resource types: `documentdb`, `volume`, and `secret`.

**DocumentDB**: NoSQL document stores.

```json
"resources": {
  "documentdb": [
    { "code": "app-data", "description": "Main application data" },
    { "code": "audit-log", "description": "Audit trail entries" }
  ]
}
```

Bound to the derived `APP_DATA_DOCDB` and `AUDIT_LOG_DOCDB` — no `environment` entries needed.

```typescript
// ✅ read the code from the environment
const docdb = await ketrics.DocumentDb.connect(ketrics.environment["APP_DATA_DOCDB"]);

// ❌ never hardcode the resource code
const docdb = await ketrics.DocumentDb.connect("app-data");
```

**Volume**: File storage buckets — same rule.

```json
"resources": {
  "volume": [
    { "code": "exports", "description": "Excel and PDF exports" }
  ]
}
```

```typescript
const volume = await ketrics.Volume.connect(ketrics.environment["EXPORTS_VOLUME"]);
```

**Secret**: Encrypted secrets (API keys, tokens) — same rule. Here the binding is named
explicitly, so it must also be declared in `environment`:

```json
"environment": [
  { "name": "STRIPE_API_KEY", "description": "Secret code for the Stripe API key" }
],
"resources": {
  "secret": [
    { "code": "stripe-key", "description": "Stripe API key", "environmentVariable": "STRIPE_API_KEY" }
  ]
}
```

#### `resources` entry, or plain `environment` entry?

Both end up as environment variables the handler reads. The only question is whether the platform
has a resource to pick.

**Use a `resources` entry when the value is the code of a Ketrics-managed resource** — and there
are exactly three kinds: `documentdb`, `volume`, `secret`. Declaring it makes the Application form
render a resource picker, so an admin chooses from the resources that exist instead of typing a
code from memory.

**Use a plain `environment` entry for everything else** — API base URLs, thresholds, feature
flags, recipient lists, and the codes of anything the platform does not model as a resource.

The trap is the third case: a value that names *some* platform object which is nonetheless not one
of the three kinds. **SQL data connections are the standard example** — they are real platform
objects, but they are not a `resources` kind, so there is no binding to declare and no picker to
render:

```json
"environment": [
  { "name": "MAIN_DB_CONNECTION", "description": "SQL data connection code" }
]
```

```typescript
const db = await ketrics.Database.connect(ketrics.environment["MAIN_DB_CONNECTION"]);
```

It is still a declared variable, so it is still created empty on deploy and still mandatory. **The
rule: if it is not `documentdb`, `volume` or `secret`, it is an `environment` entry, however
platform-ish it feels.**

## Backend build

Uses esbuild to bundle the single `backend/src/index.ts` into `backend/dist/index.js`.

### `backend/package.json`

```json
{
  "name": "my-ketrics-app-backend",
  "version": "1.0.0",
  "main": "dist/index.js",
  "scripts": {
    "build": "esbuild src/index.ts --bundle --platform=node --target=es2020 --outfile=dist/index.js",
    "clean": "rm -rf dist"
  },
  "devDependencies": {
    "@ketrics/sdk-backend": "0.13.1",
    "@types/node": ">=24.0.0",
    "esbuild": "^0.27.2",
    "typescript": "^5.3.3"
  }
}
```

### `backend/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "CommonJS",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "outDir": "dist",
    "rootDir": "src",
    "types": ["node", "@ketrics/sdk-backend"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Build command

```bash
cd backend && npm run build
# Outputs: backend/dist/index.js
```

## Frontend build

Uses tsc for type-checking and Vite for bundling.

### Build command

```bash
cd frontend && npm run build
# Outputs: frontend/dist/ (static files)
```

### Development

```bash
cd frontend && npm run dev
# Starts Vite dev server with mock handlers (no backend needed)
```

## Environment variables

Declare every environment variable the app needs in `ketrics.config.json` under `environment`. On deploy, any declared variable that does not yet exist on the app is created with an **empty value** — an admin then fills the real value in the Ketrics dashboard. Existing values are never overwritten by a deploy.

Access them at runtime via `ketrics.environment["VAR_NAME"]` in backend handlers.

Names must match `/^[A-Z][A-Z0-9_]*$/` (uppercase, digits and underscores, starting with a letter), be unique, and be at most 128 characters. Max 50 entries.

### Declared variables are mandatory

The app's declared variables are every entry in `environment` **plus** every variable bound to a
`resources` entry, whether that binding was named explicitly or derived. All of them are mandatory:

- They **cannot be deleted or renamed** in the portal while the config declares them.
- Their **values stay editable** — that is the whole point; the admin fills them in, and for a
  resource-bound variable the resource picker writes the chosen code through the same path.
- If a later deploy drops one from the config, the existing value is left untouched; it simply
  stops being mandatory and can be deleted again.

So declare only what the app actually reads. An unused declaration is a permanent empty row in
someone else's portal.

**All resource codes belong in environment variables.** DocumentDB, Volume, and Secret resource codes, and Database connection codes, must never be hardcoded in source — declare each resource (or an `environment` entry, for anything that is not one of the three kinds) and read the value from `ketrics.environment` in handler code (see "Resource declarations" above).

Common variables to declare for your app:

| Variable | Type | Description |
|----------|------|-------------|
| `*_DOCDB` | String | DocumentDB resource code (one env var per DocumentDB resource) |
| `*_VOLUME` | String | Volume resource code (one env var per Volume) |
| `*_SECRET` / `*_KEY` | String | Secret resource code (one env var per Secret) |
| `*_DB_CONNECTION` | String | SQL database connection code |
| `*_APP_CODE` | String | Application code for cross-app invocation |
| Custom variables | String | App-specific configuration (API keys, feature flags, ...) |

## GitHub Actions deployment

> **Never install the CLI unpinned.** `npm install -g @ketrics/ketrics-cli` on any Node older
> than 24 does **not** fail — npm walks back through the version history to `0.1.0`, whose
> `engines` field still allows old Node, and installs that. The ancient CLI then builds a ZIP with
> no `ketrics.config.json` in it and **reports a successful deploy**, so the app silently keeps its
> previous actions, roles and declared variables while CI stays green. Three things together
> prevent it: `node-version: 24`, a version range (`@^0.11`), and an `.npmrc` with
> `engine-strict=true` so an engine mismatch becomes an error instead of a silent downgrade.

### `.npmrc` (repository root)

```
engine-strict=true
```

### `.github/workflows/deploy.yml`

```yaml
name: Deploy to Ketrics

on:
  workflow_dispatch:
  push:
    branches: [master]
    paths:
      - 'backend/**'
      - 'frontend/**'
      - 'ketrics.config.json'

env:
  KETRICS_API_URL: https://api.ketrics.io/api/v1
  KETRICS_RUNTIME_URL: https://runtime.ketrics.io

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: prod
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 24

      # Install Ketrics CLI — ALWAYS pin. Unpinned silently installs 0.1.0 on old Node.
      - run: npm install -g @ketrics/ketrics-cli@^0.11

      # Generate .env file (only KETRICS_TOKEN is a secret)
      - name: Generate .env file
        run: |
          cat <<EOF > .env
          KETRICS_API_URL=$KETRICS_API_URL
          KETRICS_RUNTIME_URL=$KETRICS_RUNTIME_URL
          KETRICS_TENANT_ID=${{ vars.KETRICS_TENANT_ID }}
          KETRICS_APPLICATION_ID=${{ vars.KETRICS_APPLICATION_ID }}
          KETRICS_TOKEN=${{ secrets.KETRICS_TOKEN }}
          EOF

      # Backend: install + build
      - run: npm ci
        working-directory: backend
      - run: npm run build
        working-directory: backend

      # Frontend: install + build
      - run: npm ci
        working-directory: frontend
      - run: npm run build
        working-directory: frontend

      # Deploy
      - run: ketrics deploy --env .env
```

### Required secrets and variables

| Name | Type | Where to set | Description |
|------|------|-------------|-------------|
| `KETRICS_TOKEN` | Secret | GitHub repo > Settings > Secrets | API token from Ketrics dashboard |
| `KETRICS_TENANT_ID` | Variable | GitHub repo > Settings > Variables | Your tenant UUID |
| `KETRICS_APPLICATION_ID` | Variable | GitHub repo > Settings > Variables | Your application UUID |

Use `vars.*` for non-secret values (Tenant ID, Application ID) and `secrets.*` only for the API token.

## Manual deployment

For local deployment without CI/CD:

1. Install the CLI — **with an explicit tag**, on Node 24 or newer:

```bash
node --version              # must be >= 24
npm install -g @ketrics/ketrics-cli@latest
ketrics --version           # confirm you did not get 0.1.0
```

Plain `npm install -g @ketrics/ketrics-cli` can silently resolve to 0.1.0 on an older Node — see
the warning under GitHub Actions deployment. If `ketrics --version` prints `0.1.0`, upgrade Node
and reinstall before deploying anything.

2. Create a `.env` file at the project root:

```
KETRICS_API_URL=https://api.ketrics.io/api/v1
KETRICS_RUNTIME_URL=https://runtime.ketrics.io
KETRICS_TENANT_ID=<your-tenant-id>
KETRICS_APPLICATION_ID=<your-application-id>
KETRICS_TOKEN=<your-token>
```

3. Build and deploy:

```bash
cd backend && npm run build && cd ..
cd frontend && npm run build && cd ..
ketrics deploy --env .env
```

## SDK versions

| Package | Version | Purpose |
|---------|---------|---------|
| `@ketrics/sdk-backend` | 0.13.1 | Backend global types (devDependency) |
| `@ketrics/sdk-frontend` | ^0.3.0 | Frontend auth manager (dependency) |
| `@ketrics/ketrics-cli` | `^0.11` in CI, `@latest` by hand | CLI for deployment. **Never install unpinned** — see the warning above. Requires Node >= 24 |

## Deployment checklist

> **The one nothing checks for you:** `PERMISSIONS` in `backend/src/permissions.ts` and `actions`
> in `ketrics.config.json` are hand-synced. No tool compares them. Read the two lists side by side
> before every deploy.

```
- [ ] ketrics.config.json actions list the app's CAPABILITIES (read/write/export/approve, ...) —
      not role names, not handler names — each with a description for the role builder
- [ ] PERMISSIONS in backend/src/permissions.ts matches those codes exactly, no extras either way
- [ ] Every guarded handler calls requirePermission(...) with a capability from PERMISSIONS
- [ ] ketrics.config.json roles are named after jobs, and every role action is a declared action
- [ ] ketrics.config.json functions list every backend handler export
- [ ] ketrics.config.json uses "environment" — the legacy "environmentVariables" key now FAILS the deploy
- [ ] resources entries either omit environmentVariable (name is derived) or name one declared in environment
- [ ] Anything that is not documentdb / volume / secret is a plain environment entry (SQL data connections included)
- [ ] No DocumentDB / Volume / Secret / Database connection codes hardcoded in source
- [ ] Backend builds successfully: cd backend && npm run build
- [ ] Backend type-checks: cd backend && npx tsc --noEmit (catches typo'd permissions)
- [ ] Frontend builds successfully: cd frontend && npm run build
- [ ] .env file has all required variables (or GitHub secrets configured)
- [ ] Environment variable values filled in the Ketrics dashboard after first deploy
- [ ] KETRICS_TOKEN secret set in GitHub repo settings
- [ ] GitHub Actions workflow file in .github/workflows/deploy.yml
- [ ] Workflow pins the CLI (@^0.11), uses node-version 24, and the repo has .npmrc with engine-strict=true
- [ ] After deploying, confirm the app's actions/roles/variables actually changed — a 0.1.0 CLI
      reports success while shipping a ZIP with no config in it
```
