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
    "handlerName1",
    "handlerName2"
  ],
  "entry": "dist/index.js",
  "include": ["dist/**/*"],
  "exclude": ["node_modules", "*.test.js", "*.spec.js"],
  "resources": {
    "documentdb": [
      {
        "code": "resource-code",
        "description": "What this DocumentDB stores"
      }
    ],
    "volume": [
      {
        "code": "volume-code",
        "description": "What files this volume stores"
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
| `actions` | Yes | Array of handler function names. MUST match exports in `backend/src/index.ts` |
| `entry` | Yes | Path to the built backend bundle (relative to `backend/`) |
| `include` | No | Glob patterns for files to include in deployment |
| `exclude` | No | Glob patterns for files to exclude |
| `resources` | No | Declares DocumentDB collections and Volumes the app uses |

### Actions sync rule

The `actions` array MUST list every exported handler name from `backend/src/index.ts`. If a handler is exported but not in the actions array, it won't be callable from the frontend. If a name is in actions but not exported, deployment will reference a non-existent function.

```
backend/src/index.ts exports  <-->  ketrics.config.json actions
```

Always keep these in sync when adding or removing handlers.

### Resource declarations

**DocumentDB**: NoSQL document stores. The `code` is used in `ketrics.DocumentDb.connect(code)`.

```json
"documentdb": [
  { "code": "app-data", "description": "Main application data" },
  { "code": "audit-log", "description": "Audit trail entries" }
]
```

**Volume**: File storage buckets. The `code` is used in `ketrics.Volume.connect(code)`.

```json
"volume": [
  { "code": "exports", "description": "Excel and PDF exports" },
  { "code": "uploads", "description": "User-uploaded files" }
]
```

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

Environment variables are configured in the Ketrics dashboard (not in code). Access them at runtime via `ketrics.environment["VAR_NAME"]` in backend handlers.

Common variables to document for your app:

| Variable | Type | Description |
|----------|------|-------------|
| `DB_CONNECTIONS` | JSON string | Array of `{code, name}` for database connections |
| `DOCDB_*` | String | DocumentDB resource code overrides |
| `EXPORTS_VOLUME` | String | Volume code for file exports |
| Custom variables | String | App-specific configuration |

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
- [ ] ketrics.config.json actions match all backend exports
- [ ] ketrics.config.json resources list all DocumentDB and Volume codes used
- [ ] Backend builds successfully: cd backend && npm run build
- [ ] Frontend builds successfully: cd frontend && npm run build
- [ ] .env file has all required variables (or GitHub secrets configured)
- [ ] Environment variables configured in Ketrics dashboard
- [ ] KETRICS_TOKEN secret set in GitHub repo settings
- [ ] GitHub Actions workflow file in .github/workflows/deploy.yml
```
