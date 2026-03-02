---
name: ketrics-app
description: Scaffolds and builds Ketrics tenant applications with backend handlers, frontend React UI, and platform SDK integrations. Use when creating a new Ketrics app, adding backend handlers, setting up database connections, DocumentDB storage, Excel exports, Volume file storage, messaging, comments, environment variables, or deploying to the Ketrics platform.
---

# Ketrics Application Builder

Build tenant applications on the Ketrics platform. This skill covers project scaffolding, backend handler development, frontend React integration, and deployment.

## Architecture overview

A Ketrics app has two parts:

1. **Backend** (`backend/src/index.ts`): Single TypeScript file exporting async handler functions. Uses the global `ketrics` object (typed by `@ketrics/sdk-backend`) with no imports needed. Built with esbuild into a single bundle.
2. **Frontend** (`frontend/src/`): React app (Vite + TypeScript) embedded as an iframe. Uses `@ketrics/sdk-frontend` for auth. Calls backend handlers via a service layer.

A `ketrics.config.json` at the project root declares the app name, runtime, action names, entry point, and resources.

## Project structure

```
my-ketrics-app/
├── ketrics.config.json          # App config (actions, resources)
├── CLAUDE.md                    # Dev instructions
├── backend/
│   ├── package.json             # devDeps: @ketrics/sdk-backend, esbuild, typescript
│   ├── tsconfig.json
│   └── src/
│       └── index.ts             # All handler functions (single file)
├── frontend/
│   ├── package.json             # deps: react, @ketrics/sdk-frontend, vite
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── index.html
│   └── src/
│       ├── App.tsx              # Main component
│       ├── main.tsx             # React entry point
│       ├── types.ts             # Shared TypeScript interfaces
│       ├── services/
│       │   └── index.ts         # callFunction service layer
│       └── mocks/
│           └── handlers.ts      # Dev-mode mock handlers
└── .github/
    └── workflows/
        └── deploy.yml           # CI/CD pipeline
```

## Step-by-step: Create a new app

### 1. Initialize the project

```bash
mkdir my-ketrics-app && cd my-ketrics-app
```

### 2. Create `ketrics.config.json`

```json
{
  "name": "my-ketrics-app",
  "version": "1.0.0",
  "description": "My Ketrics application",
  "runtime": "nodejs18",
  "actions": [
    "listItems",
    "getItem",
    "createItem",
    "updateItem",
    "deleteItem"
  ],
  "entry": "dist/index.js",
  "include": ["dist/**/*"],
  "exclude": ["node_modules", "*.test.js"],
  "resources": {
    "documentdb": [
      { "code": "app-data", "description": "Main data store" }
    ]
  }
}
```

**Key rules:**
- `actions` array MUST match the exported handler names in `backend/src/index.ts`
- `resources` declares DocumentDB collections and Volumes the app needs
- Add `"volume"` entries under resources when the app needs file storage

### 3. Set up the backend

```bash
mkdir -p backend/src
cd backend
npm init -y
npm install -D @ketrics/sdk-backend@0.13.1 esbuild typescript @types/node
```

**`backend/package.json` scripts:**
```json
{
  "scripts": {
    "build": "esbuild src/index.ts --bundle --platform=node --target=es2020 --outfile=dist/index.js"
  }
}
```

### 4. Write backend handlers

See [BACKEND_REFERENCE.md](BACKEND_REFERENCE.md) for the complete SDK API and patterns.

Every handler is an `async` function that receives a typed payload and returns a result object:

```typescript
const myHandler = async (payload: { id: string }) => {
  const userId = ketrics.requestor.userId;
  // ... handler logic using ketrics.* SDK
  return { result: "value" };
};

export { myHandler };
```

### 5. Set up the frontend

```bash
mkdir -p frontend/src/services frontend/src/mocks
cd frontend
npm init -y
npm install react react-dom @ketrics/sdk-frontend lucide-react
npm install -D @vitejs/plugin-react vite typescript @types/react @types/react-dom
```

See [FRONTEND_REFERENCE.md](FRONTEND_REFERENCE.md) for the service layer, auth, and mock handler patterns.

### 6. Deploy

See [CONFIG_AND_DEPLOY.md](CONFIG_AND_DEPLOY.md) for ketrics.config.json details and GitHub Actions CI/CD.

```bash
# Manual deploy (requires .env with KETRICS_TOKEN)
ketrics deploy --env .env
```

## SDK quick reference

The `ketrics` global object is available in all backend handlers with no imports. Here's a quick overview of each subsystem:

### Environment variables

```typescript
const value = ketrics.environment["MY_VAR"];
```

Read app-specific configuration. Common pattern: store JSON arrays for connection configs, resource codes, feature flags.

### Requestor context

```typescript
ketrics.requestor.userId              // Current user ID
ketrics.requestor.name                // Display name
ketrics.requestor.email               // Email
ketrics.requestor.applicationPermissions  // ["editor", ...]
```

### Database (SQL queries)

```typescript
const db = await ketrics.Database.connect(connectionCode);
try {
  const result = await db.query<Record<string, unknown>>(sql, params);
  // result.rows, result.rowCount
} finally {
  await db.close();
}
```

### DocumentDB (NoSQL storage)

DynamoDB-style pk/sk model with `put`, `get`, `delete`, `list` operations. Supports cursor-based pagination.

```typescript
const docdb = await ketrics.DocumentDb.connect(resourceCode);
await docdb.put(pk, sk, item);
const item = await docdb.get(pk, sk);
const result = await docdb.list(pk, { skPrefix: "PREFIX#", limit: 50 });
// result.items: Record<string, unknown>[], result.cursor: string | undefined
await docdb.delete(pk, sk);

// Pagination: pass cursor from previous result to get next page
if (result.cursor) {
  const nextPage = await docdb.list(pk, { skPrefix: "PREFIX#", limit: 50, cursor: result.cursor });
}
```

### Excel generation

```typescript
const excel = ketrics.Excel.create();
const sheet = excel.addWorksheet("Sheet1");
sheet.columns = columns.map(col => ({ header: col, key: col, width: 15 }));
sheet.addRows(rows);
const buffer = await excel.toBuffer();
```

### Volume (file storage)

```typescript
const volume = await ketrics.Volume.connect(volumeCode);
await volume.put(fileKey, buffer);
const { url } = await volume.generateDownloadUrl(fileKey);
```

### Messages (notifications)

```typescript
await ketrics.Messages.sendBulk({
  userIds: ["user1", "user2"],
  type: "CUSTOM_TYPE",
  subject: "Subject line",
  body: "**Markdown** body",
  priority: "MEDIUM",
});
```

### Users

```typescript
const users = await ketrics.Users.list();
// [{ id, firstName, lastName, email }, ...]
```

### Logging

```typescript
ketrics.console.error("Non-critical error message");
```

### Background jobs

```typescript
// Launch a long-running task in the background
const jobId = await ketrics.Job.runInBackground({
  function: "_syncMyTask",       // Handler name (must be in actions array)
  payload: { entityId, data },   // Passed to the background handler
});
// The background handler runs asynchronously; store jobId for tracking
```

Background handler naming convention: prefix with `_` (e.g., `_syncCreateRecord`). These handlers are called by the platform, not by the frontend.

### Cross-application invocation

```typescript
// Call a handler in another Ketrics application
const appCode = ketrics.environment["OTHER_APP_CODE"];
const app = await ketrics.Applications.connect(appCode);
const result = await app.invoke<PayloadType, ResponseType>("handlerName", payload);
// result.success: boolean, result.result: ResponseType
```

### HTTP client (external API requests)

```typescript
const response = await ketrics.http.get("https://api.example.com/users", {
  params: { status: "active", limit: 20 },
  headers: { Authorization: `Bearer ${apiKey}` },
  timeout: 10000,
});
// response.status, response.statusText, response.data, response.headers

await ketrics.http.post("https://api.example.com/orders", { customerId, items }, { headers });
await ketrics.http.put(`https://api.example.com/users/${userId}`, { name, email }, { headers });
await ketrics.http.delete(`https://api.example.com/users/${userId}`, { headers });
```

### Application context

```typescript
ketrics.application.id  // Application UUID
```

## Common patterns

### Permission checking

```typescript
function requireEditor(): void {
  if (!ketrics.requestor.applicationPermissions.includes("editor")) {
    throw new Error("Permission denied: editor role required");
  }
}
```

### Ownership verification

```typescript
const existing = await docdb.get(pk, sk);
if (!existing) throw new Error("Not found");
if (existing.createdBy !== userId) {
  throw new Error("You can only modify your own resources");
}
```

### DocumentDB key design

Use prefixed composite keys for multi-entity storage in a single DocumentDB:

| Entity | pk | sk |
|--------|----|----|
| User items | `USER#${userId}` | `ITEM#${itemId}` |
| Tenant-wide | `TENANT_ITEMS` | `ITEM#${itemId}` |
| Comments | `COMMENTS#${targetId}` | `COMMENT#${createdAt}#${commentId}` |
| Index | `INDEX#${scope}` | `KEY#${lookupKey}` |

### Export to Excel via Volume

```typescript
const exportToExcel = async (payload: { data: unknown[] }) => {
  const volumeCode = ketrics.environment["EXPORTS_VOLUME"];
  if (!volumeCode) throw new Error("EXPORTS_VOLUME not configured");

  // Generate Excel
  const excel = ketrics.Excel.create();
  const sheet = excel.addWorksheet("Export");
  // ... add columns and rows
  const buffer = await excel.toBuffer();

  // Save to Volume and get download URL
  const filename = `export_${Date.now()}.xlsx`;
  const fileKey = `${ketrics.application.id}/${filename}`;
  const volume = await ketrics.Volume.connect(volumeCode);
  await volume.put(fileKey, buffer);
  const { url } = await volume.generateDownloadUrl(fileKey);

  return { url, filename };
};
```

### Comment system

Store comments with composite keys for efficient queries. Use a separate index for fast count lookups:

```typescript
// Store comment
await docdb.put(
  `COMMENTS#${targetId}`,
  `COMMENT#${createdAt}#${commentId}`,
  commentData
);

// Update count index
const indexItem = await docdb.get(`INDEX#${scope}`, `KEY#${targetId}`);
const count = indexItem ? (indexItem.count as number) : 0;
await docdb.put(`INDEX#${scope}`, `KEY#${targetId}`, {
  targetId,
  count: count + 1,
  lastCommentAt: now,
});
```

### State machine pattern

```typescript
type EntityState = "PENDIENTE" | "APROBADA" | "EN_PROCESO" | "CONTABILIZADA" | "ERROR";

// Validate transitions in each handler
const approveEntity = async (payload: { id: string }) => {
  requireApprover();
  const entity = await docdb.get(pk, sk) as unknown as MyEntity;
  if (entity.estado !== "PENDIENTE") {
    throw new Error("Solo se puede aprobar en estado PENDIENTE");
  }
  // Use intermediate states for async ops: APROBADA → EN_PROCESO → CONTABILIZADA
  // Only allow modifications in PENDIENTE state
  const updated = { ...entity, estado: "APROBADA" as EntityState, updatedAt: new Date().toISOString() };
  await docdb.put(pk, sk, updated as unknown as Record<string, unknown>);
  return { entity: updated };
};
```

### Shared resources with access control

```typescript
// Filter: owned by me OR shared with me OR shared with all
const items = result.items.filter((item) => {
  if (item.owner === userId) return true;
  if (item.visibility !== "shared") return false;
  const sw = item.sharedWith;
  if (sw === "all") return true;
  if (Array.isArray(sw) && sw.includes(userId)) return true;
  return false;
});
```

## Additional resources

- **Backend SDK reference**: See [BACKEND_REFERENCE.md](BACKEND_REFERENCE.md) for complete API docs and all handler patterns
- **Frontend patterns**: See [FRONTEND_REFERENCE.md](FRONTEND_REFERENCE.md) for the service layer, auth manager, mock handlers, and type definitions
- **Config and deployment**: See [CONFIG_AND_DEPLOY.md](CONFIG_AND_DEPLOY.md) for ketrics.config.json schema, GitHub Actions workflow, and environment setup

## Checklist for new apps

```
- [ ] Create ketrics.config.json with correct actions and resources
- [ ] Set up backend with @ketrics/sdk-backend and esbuild
- [ ] Write handler functions using ketrics.* global
- [ ] Export all handlers and sync with config actions array
- [ ] Background job handlers prefixed with _ and listed in actions array
- [ ] State machine transitions validated in each handler
- [ ] Set up frontend with React, Vite, and @ketrics/sdk-frontend
- [ ] Create APIClient service layer with dev/prod branching
- [ ] Write mock handlers for local development
- [ ] Define TypeScript interfaces in types.ts
- [ ] Set up GitHub Actions deploy workflow
- [ ] Configure environment variables in Ketrics dashboard
```
