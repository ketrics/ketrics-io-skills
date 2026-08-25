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
  "actions": ["admin", "editor", "user"],
  "functions": [
    "getPermissions",
    "listItems",
    "createItem"
  ],
  "environmentVariables": [
    { "name": "APP_DATA_DOCDB", "description": "DocumentDB resource code for the main data store" },
    { "name": "EXPORTS_VOLUME", "description": "Volume resource code for generated exports" },
    { "name": "STRIPE_API_KEY", "description": "Secret resource code for the Stripe API key" }
  ],
  "entry": "dist/index.js",
  "include": ["dist/**/*"],
  "exclude": ["node_modules", "*.test.js", "*.spec.js"],
  "resources": {
    "documentdb": [
      {
        "code": "main-store",
        "description": "What this DocumentDB stores",
        "environmentVariable": "APP_DATA_DOCDB"
      }
    ],
    "volume": [
      {
        "code": "exports",
        "description": "What files this volume stores",
        "environmentVariable": "EXPORTS_VOLUME"
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

### Field descriptions

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Application identifier (lowercase, hyphens) |
| `version` | Yes | Semver version string |
| `description` | No | Human-readable description |
| `runtime` | Yes | Runtime environment. Use `"nodejs18"` |
| `actions` | Yes | Role-style permission slots the app exposes (e.g. `admin`, `editor`, `user`). On deploy these **become the app's "Supported Actions"** |
| `functions` | No | Handler function names. SHOULD match the exports in `backend/src/index.ts`. Stored as metadata |
| `environmentVariables` | No | Env vars the app needs, as `{ name, description }`. Missing ones are created (empty) on deploy |
| `entry` | Yes | Path to the built backend bundle (relative to `backend/`) |
| `include` | No | Glob patterns for files to include in deployment |
| `exclude` | No | Glob patterns for files to exclude |
| `resources` | No | Declares the DocumentDB, Volume and Secret resources the app uses (metadata only — see below) |

### What deploy does with this file

On `ketrics deploy` the CLI bundles `ketrics.config.json` into the deployment ZIP and the platform syncs the application record from it:

- **`actions`** replaces the app's Supported Actions (skipped for marketplace-installed apps).
- **`environmentVariables`** — every declared variable that does not already exist on the app is created with an **empty value**; existing values are never overwritten. Fill the real values in the Ketrics dashboard.
- **`functions`** and **`resources`** are stored as metadata.

### `actions` are roles, not handlers

`actions` is the list of **role-style permission slots** the app defines — the values that appear in `ketrics.requestor.applicationPermissions` and that permission helpers check (`requireEditor`, `requireAdmin`, ...). Typical values: `admin`, `editor`, `user`.

`actions` is **not** the list of handler functions. Handler names go in `functions`.

### Functions sync rule

The `functions` array SHOULD list every exported handler name from `backend/src/index.ts`, including background handlers prefixed with `_`. It is metadata describing the app's callable surface.

```
backend/src/index.ts exports  <-->  ketrics.config.json functions
```

Keep these in sync when adding or removing handlers.

### Resource declarations — never hardcode resource codes

Each `resources` entry declares a resource the app depends on. An entry has three fields:

- **`code`** — a **logical identifier** for the resource slot (e.g. `main-store`, `exports`, `stripe-key`). It is a stable label for the declaration — **not** the platform resource code, and not an environment variable name.
- **`description`** — optional; what the resource is for.
- **`environmentVariable`** — the **name of an environment variable** that holds the real resource code at runtime. It **must** reference a variable declared in `environmentVariables` (deploy validation rejects an unknown reference).

The real resource code is never written in the config or in handler source. It reaches the app through the environment variable: in the Ketrics dashboard, the application's Update form shows a **resource picker** for every declared resource, letting an admin select an existing resource of the matching type — the chosen code is saved into the linked environment variable.

Why: resource codes differ per tenant and per environment. A hardcoded code ties the bundle to one tenant's infrastructure and breaks on deploy to another.

The pattern, applied consistently:

1. Declare an env var in `environmentVariables` for each resource the app uses.
2. Add a `resources` entry with a logical `code` and `environmentVariable` set to that env var's name.
3. In handlers, read the code via `ketrics.environment["VAR_NAME"]` — never a string literal.

Resource types: `documentdb`, `volume`, and `secret`.

**DocumentDB**: NoSQL document stores.

```json
"environmentVariables": [
  { "name": "APP_DATA_DOCDB", "description": "DocumentDB code for the main data store" },
  { "name": "AUDIT_LOG_DOCDB", "description": "DocumentDB code for audit-trail entries" }
],
"resources": {
  "documentdb": [
    { "code": "main-store", "description": "Main application data", "environmentVariable": "APP_DATA_DOCDB" },
    { "code": "audit-log", "description": "Audit trail entries", "environmentVariable": "AUDIT_LOG_DOCDB" }
  ]
}
```

```typescript
// ✅ read the code from the environment
const docdb = await ketrics.DocumentDb.connect(ketrics.environment["APP_DATA_DOCDB"]);

// ❌ never hardcode the resource code
const docdb = await ketrics.DocumentDb.connect("app-data");
```

**Volume**: File storage buckets — same rule.

```json
"environmentVariables": [
  { "name": "EXPORTS_VOLUME", "description": "Volume code for Excel/PDF exports" }
],
"resources": {
  "volume": [
    { "code": "exports", "description": "Excel and PDF exports", "environmentVariable": "EXPORTS_VOLUME" }
  ]
}
```

**Secret**: Encrypted secrets (API keys, tokens) — same rule.

```json
"environmentVariables": [
  { "name": "STRIPE_API_KEY", "description": "Secret code for the Stripe API key" }
],
"resources": {
  "secret": [
    { "code": "stripe-key", "description": "Stripe API key", "environmentVariable": "STRIPE_API_KEY" }
  ]
}
```

**Database connections**: SQL connection codes follow the same env-var rule — store the connection code in an environment variable and pass `ketrics.environment["..."]` to `ketrics.Database.connect(...)`. Never hardcode a connection code. Note: database connections are **not** a `resources` type — declare only the environment variable for them.

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

Declare every environment variable the app needs in `ketrics.config.json` under `environmentVariables`. On deploy, any declared variable that does not yet exist on the app is created with an **empty value** — an admin then fills the real value in the Ketrics dashboard. Existing values are never overwritten by a deploy.

Access them at runtime via `ketrics.environment["VAR_NAME"]` in backend handlers.

**All resource codes belong in environment variables.** DocumentDB, Volume, and Secret resource codes, and Database connection codes, must never be hardcoded in source — declare an env var for each and reference it from the resource's `environmentVariable` field and the handler code (see "Resource declarations" above).

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
          node-version: 18

      # Install Ketrics CLI
      - run: npm install -g @ketrics/ketrics-cli

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

1. Create a `.env` file at the project root:

```
KETRICS_API_URL=https://api.ketrics.io/api/v1
KETRICS_RUNTIME_URL=https://runtime.ketrics.io
KETRICS_TENANT_ID=<your-tenant-id>
KETRICS_APPLICATION_ID=<your-application-id>
KETRICS_TOKEN=<your-token>
```

2. Build and deploy:

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
| `@ketrics/ketrics-cli` | latest | CLI for deployment |

## Deployment checklist

```
- [ ] ketrics.config.json actions list the app's role slots (admin/editor/user, ...)
- [ ] ketrics.config.json functions list every backend handler export
- [ ] ketrics.config.json environmentVariables declare every env var the app reads
- [ ] ketrics.config.json resources entries set environmentVariable to a declared env var name
- [ ] No DocumentDB / Volume / Secret / Database connection codes hardcoded in source
- [ ] Backend builds successfully: cd backend && npm run build
- [ ] Frontend builds successfully: cd frontend && npm run build
- [ ] .env file has all required variables (or GitHub secrets configured)
- [ ] Environment variable values filled in the Ketrics dashboard after first deploy
- [ ] KETRICS_TOKEN secret set in GitHub repo settings
- [ ] GitHub Actions workflow file in .github/workflows/deploy.yml
```
