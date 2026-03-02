# Backend SDK Reference

Complete reference for the `ketrics` global object and backend handler patterns. The `ketrics` object is automatically available in all backend handlers — no imports needed. Types come from `@ketrics/sdk-backend` (devDependency).

## Handler function pattern

Every handler is an async function that takes a typed payload and returns a result. Handlers are exported by name and must match the `actions` array in `ketrics.config.json`.

```typescript
interface MyPayload {
  id: string;
  name: string;
}

const myHandler = async (payload: MyPayload) => {
  requireEditor(); // permission check
  const { id, name } = payload;

  // Validate input
  if (!id) throw new Error("id is required");
  if (!name?.trim()) throw new Error("name is required");

  // Get requestor context
  const userId = ketrics.requestor.userId;
  if (!userId) throw new Error("User context is required");

  // ... handler logic ...

  return { item: { id, name } };
};

export { myHandler };
```

## ketrics.environment

Read application environment variables configured in the Ketrics dashboard.

```typescript
// Simple string value
const apiKey = ketrics.environment["API_KEY"];

// JSON-encoded config
const raw = ketrics.environment["DB_CONNECTIONS"];
const connections: { code: string; name: string }[] = JSON.parse(raw);

// With fallback
const docDbCode = ketrics.environment["DOCDB_CODE"] || "default-store";

// Volume reference
const volumeCode = ketrics.environment["EXPORTS_VOLUME"];
if (!volumeCode) throw new Error("EXPORTS_VOLUME not configured");
```

Common environment variables:
- `DB_CONNECTIONS` — JSON array of `{code, name}` for database connection dropdown
- `DOCDB_*` — DocumentDB resource codes
- `EXPORTS_VOLUME` — Volume code for file exports

## ketrics.requestor

Context about the authenticated user making the request.

```typescript
ketrics.requestor.userId              // string — User ID
ketrics.requestor.name                // string — Display name
ketrics.requestor.email               // string — Email address
ketrics.requestor.applicationPermissions  // string[] — e.g., ["editor"]
```

### Permission checking pattern

Define a helper for each role used in the application:

```typescript
function requireEditor(): void {
  if (!ketrics.requestor.applicationPermissions.includes("editor")) {
    throw new Error("Permission denied: editor role required");
  }
}

function requireApprover(): void {
  if (!ketrics.requestor.applicationPermissions.includes("approver")) {
    throw new Error("Permission denied: approver role required");
  }
}

function requireAdmin(): void {
  if (!ketrics.requestor.applicationPermissions.includes("admin")) {
    throw new Error("Permission denied: admin role required");
  }
}

// Usage: call at the start of handlers — different actions require different roles
const createItem = async (payload: CreatePayload) => { requireEditor(); /* ... */ };
const approveItem = async (payload: ApprovePayload) => { requireApprover(); /* ... */ };
const revertItem = async (payload: RevertPayload) => { requireAdmin(); /* ... */ };
```

## ketrics.Database

Connect to SQL databases configured for the tenant. Supports parameterized queries.

### Connect and query

```typescript
const db = await ketrics.Database.connect(connectionCode);
try {
  const result = await db.query<Record<string, unknown>>(sql, params);
  // result.rows: Record<string, unknown>[]
  // result.rowCount: number
} finally {
  await db.close(); // ALWAYS close in finally block
}
```

### Read-only enforcement

Block DML/DDL keywords to ensure read-only access:

```typescript
const FORBIDDEN_KEYWORDS = /\b(INSERT|UPDATE|DELETE|DROP|CREATE|ALTER|TRUNCATE)\b/i;

if (FORBIDDEN_KEYWORDS.test(sql)) {
  throw new Error("Only SELECT queries are allowed.");
}
```

### Row limiting

Wrap queries to enforce server-side row limits:

```typescript
const MAX_ROWS = 500;
const limitedSql = `SELECT * FROM (${sql}) AS __limited_result LIMIT ${MAX_ROWS}`;
const result = await db.query<Record<string, unknown>>(limitedSql, params);
```

### Parameterized queries (positional)

Use `$1`, `$2`, etc. for positional parameters:

```typescript
const sql = "SELECT * FROM users WHERE status = $1 AND role = $2";
const params = ["active", "admin"];
const result = await db.query<Record<string, unknown>>(sql, params);
```

### Template parameter replacement

Convert `{{paramName}}` placeholders to positional parameters:

```typescript
// Input: "SELECT * FROM users WHERE status = {{status}}"
// Output: "SELECT * FROM users WHERE status = $1"  with values = ["active"]

let parameterizedSql = rawSql;
const values: unknown[] = [];
let paramIndex = 0;

// Detect all {{paramName}} placeholders
const paramNames: string[] = [];
const regex = /\{\{(\w+)\}\}/g;
let match;
while ((match = regex.exec(parameterizedSql)) !== null) {
  if (!paramNames.includes(match[1])) paramNames.push(match[1]);
}

// Replace each with positional $N
for (const pName of paramNames) {
  paramIndex++;
  parameterizedSql = parameterizedSql.replace(
    new RegExp(`\\{\\{${pName}\\}\\}`, "g"),
    () => `$${paramIndex}`
  );
  values.push(paramValues[pName] ?? "");
}
```

### Identifier parameters (table/column names)

For dynamic table or column names, validate and interpolate directly (cannot use positional params):

```typescript
const VALID_IDENTIFIER = /^[a-zA-Z0-9_.\-]+$/;

if (param.isIdentifier) {
  if (!val || !VALID_IDENTIFIER.test(val)) {
    throw new Error(`Invalid identifier: "${val}"`);
  }
  // Direct string interpolation (safe after validation)
  sql = sql.replace(`{{${paramName}}}`, val);
}
```

### Conditional SQL blocks

Support optional sections in SQL using `{{#if param}}...{{/if}}`:

```typescript
parameterizedSql = parameterizedSql.replace(
  /\{\{#if\s+(\w+)\}\}([\s\S]*?)\{\{\/if\}\}/g,
  (_, condParam: string, content: string) => {
    const val = paramValues[condParam];
    return (val != null && val.trim() !== "") ? content : "";
  }
);
```

Example SQL:
```sql
SELECT * FROM orders
WHERE status = {{status}}
{{#if date_from}} AND created_at >= {{date_from}} {{/if}}
{{#if date_to}} AND created_at <= {{date_to}} {{/if}}
```

## ketrics.DocumentDb

NoSQL document store with DynamoDB-style partition key (pk) and sort key (sk).

### Connect

```typescript
const docdb = await ketrics.DocumentDb.connect(resourceCode);
// resourceCode matches the "code" in ketrics.config.json resources.documentdb
```

### Put (create or overwrite)

```typescript
const item: Record<string, unknown> = {
  id: crypto.randomUUID(),
  name: "My Item",
  createdBy: userId,
  createdAt: new Date().toISOString(),
  updatedAt: new Date().toISOString(),
};

await docdb.put(`USER#${userId}`, `ITEM#${item.id}`, item);
```

### Get (single item)

```typescript
const item = await docdb.get(`USER#${userId}`, `ITEM#${itemId}`);
if (!item) throw new Error("Not found");
```

### List (query by pk with optional sk prefix)

```typescript
const result = await docdb.list(`USER#${userId}`, {
  skPrefix: "ITEM#",        // Filter by sort key prefix
  scanForward: false,        // Reverse chronological order
  limit: 100,                // Max items to return
});
// result.items: Record<string, unknown>[]
// result.cursor: string | undefined — present when more results exist
```

### Pagination with cursor

When a `list()` call returns more items than the `limit`, the response includes a `cursor` string. Pass it back to fetch the next page:

```typescript
const docdb = await ketrics.DocumentDb.connect(resourceCode);

// First page
const page1 = await docdb.list("users", { limit: 50 });

// Next page (if cursor exists)
if (page1.cursor) {
  const page2 = await docdb.list("users", { limit: 50, cursor: page1.cursor });
}
```

### Fetching all pages

Loop until there is no cursor to collect every item:

```typescript
const fetchAll = async (pk: string, options: { skPrefix?: string; limit?: number } = {}) => {
  const docdb = await ketrics.DocumentDb.connect(getDocDbCode());
  const allItems: Record<string, unknown>[] = [];
  let cursor: string | undefined;

  do {
    const result = await docdb.list(pk, { ...options, cursor });
    allItems.push(...result.items);
    cursor = result.cursor;
  } while (cursor);

  return allItems;
};
```

### Query on secondary index with pagination

```typescript
const result = await docdb.query("status#active", {
  limit: 100,
  cursor: someCursor,  // from a previous query result
});
// result.items, result.cursor
```

### Delete

```typescript
await docdb.delete(`USER#${userId}`, `ITEM#${itemId}`);
```

### Key design patterns

Design composite keys with prefixes for multi-entity storage in a single DocumentDB:

**User-scoped items (each user owns their items):**
```typescript
pk = `USER#${userId}`
sk = `ITEM#${itemId}`
```

**Tenant-wide items (shared across all users):**
```typescript
pk = `TENANT_ITEMS`
sk = `ITEM#${itemId}`
```

**Comments on a target entity:**
```typescript
pk = `COMMENTS#${connectionCode}#${targetKey}`
sk = `COMMENT#${createdAt}#${commentId}`
// Using createdAt in sk gives chronological ordering
```

**Count index for fast lookups:**
```typescript
pk = `INDEX#${scope}`
sk = `KEY#${lookupKey}`
// Store: { lookupKey, count: N, lastUpdatedAt }
```

### CRUD handler template

```typescript
// CREATE
const createItem = async (payload: CreatePayload) => {
  requireEditor();
  const { name } = payload;
  if (!name?.trim()) throw new Error("name is required");

  const userId = ketrics.requestor.userId;
  const docdb = await ketrics.DocumentDb.connect(getDocDbCode());
  const itemId = crypto.randomUUID();
  const now = new Date().toISOString();

  const item = { id: itemId, name: name.trim(), createdBy: userId, createdAt: now, updatedAt: now };
  await docdb.put(`USER#${userId}`, `ITEM#${itemId}`, item);
  return { item };
};

// READ (single)
const getItem = async (payload: { id: string }) => {
  const { id } = payload;
  if (!id) throw new Error("id is required");

  const userId = ketrics.requestor.userId;
  const docdb = await ketrics.DocumentDb.connect(getDocDbCode());
  const item = await docdb.get(`USER#${userId}`, `ITEM#${id}`);
  if (!item) throw new Error("Not found");
  if (item.createdBy !== userId) throw new Error("Access denied");
  return { item };
};

// READ (list)
const listItems = async () => {
  const userId = ketrics.requestor.userId;
  const docdb = await ketrics.DocumentDb.connect(getDocDbCode());
  const result = await docdb.list(`USER#${userId}`, { skPrefix: "ITEM#", scanForward: false });
  return { items: result.items };
};

// UPDATE
const updateItem = async (payload: UpdatePayload) => {
  requireEditor();
  const { id, name } = payload;
  if (!id) throw new Error("id is required");

  const userId = ketrics.requestor.userId;
  const docdb = await ketrics.DocumentDb.connect(getDocDbCode());
  const pk = `USER#${userId}`;
  const sk = `ITEM#${id}`;

  const existing = await docdb.get(pk, sk);
  if (!existing) throw new Error("Not found");
  if (existing.createdBy !== userId) throw new Error("You can only update your own items");

  const item = { ...existing, name: name.trim(), updatedAt: new Date().toISOString() };
  await docdb.put(pk, sk, item);
  return { item };
};

// DELETE
const deleteItem = async (payload: { id: string }) => {
  requireEditor();
  const { id } = payload;
  if (!id) throw new Error("id is required");

  const userId = ketrics.requestor.userId;
  const docdb = await ketrics.DocumentDb.connect(getDocDbCode());
  const pk = `USER#${userId}`;
  const sk = `ITEM#${id}`;

  const existing = await docdb.get(pk, sk);
  if (!existing) throw new Error("Not found");
  if (existing.createdBy !== userId) throw new Error("You can only delete your own items");

  await docdb.delete(pk, sk);
  return { success: true };
};
```

## ketrics.Excel

Create Excel workbooks in memory.

```typescript
const excel = ketrics.Excel.create();
const sheet = excel.addWorksheet("Sheet Name");

// Set columns with headers and widths
sheet.columns = [
  { header: "Name", key: "name", width: 20 },
  { header: "Email", key: "email", width: 30 },
  { header: "Status", key: "status", width: 15 },
];

// Add rows (array of arrays, matching column order)
sheet.addRows(
  data.map(row => [row.name, row.email, row.status])
);

// Get buffer for saving to Volume
const buffer = await excel.toBuffer();
```

### Dynamic column widths

```typescript
sheet.columns = columns.map(col => ({
  header: col,
  key: col,
  width: Math.max(12, col.length + 4),
}));
```

## ketrics.Volume

File storage with presigned download URLs.

```typescript
const volume = await ketrics.Volume.connect(volumeCode);

// Upload a file
const fileKey = `${ketrics.application.id}/${filename}`;
await volume.put(fileKey, buffer);

// Generate download URL
const presigned = await volume.generateDownloadUrl(fileKey);
// presigned.url — time-limited download URL
```

### Complete export pattern (Excel + Volume)

```typescript
const exportData = async (payload: { data: Record<string, unknown>[] }) => {
  requireEditor();
  const { data } = payload;
  if (!data?.length) throw new Error("No data to export");

  const volumeCode = ketrics.environment["EXPORTS_VOLUME"];
  if (!volumeCode) throw new Error("EXPORTS_VOLUME not configured");

  const columns = Object.keys(data[0]);

  // Build Excel
  const excel = ketrics.Excel.create();
  const sheet = excel.addWorksheet("Export");
  sheet.columns = columns.map(col => ({
    header: col,
    key: col,
    width: Math.max(12, col.length + 4),
  }));
  sheet.addRows(data.map(row => columns.map(col => row[col] ?? "")));
  const buffer = await excel.toBuffer();

  // Save to Volume
  const filename = `export_${Date.now()}.xlsx`;
  const fileKey = `${ketrics.application.id}/${filename}`;
  const volume = await ketrics.Volume.connect(volumeCode);
  await volume.put(fileKey, buffer);
  const { url } = await volume.generateDownloadUrl(fileKey);

  return { url, filename };
};
```

## ketrics.Messages

Send notifications to users in the tenant.

```typescript
// Bulk send to specific users
await ketrics.Messages.sendBulk({
  userIds: ["user-id-1", "user-id-2"],
  type: "CUSTOM_EVENT_TYPE",
  subject: "Notification subject",
  body: "**Markdown** content is supported in the body.",
  priority: "MEDIUM",  // "LOW" | "MEDIUM" | "HIGH"
});
```

### Send to a single user

```typescript
await ketrics.Messages.send({
  userId: ketrics.requestor.userId!,
  type: "notification",
  subject: "Task completed",
  body: `Your task **${taskName}** has finished successfully.`,
});
```

Use `send()` for targeted notifications (e.g., background job completion). Use `sendBulk()` for multi-user notifications.

### Notification on share pattern

```typescript
if (visibility === "shared" && Array.isArray(sharedWith) && sharedWith.length > 0) {
  const senderName = ketrics.requestor.name || ketrics.requestor.email;
  try {
    await ketrics.Messages.sendBulk({
      userIds: sharedWith,
      type: "RESOURCE_SHARED",
      subject: `${senderName} shared a resource with you`,
      body: `**${senderName}** shared **${resourceName}** with you.`,
      priority: "MEDIUM",
    });
  } catch {
    // Non-critical — don't fail the main operation
    ketrics.console.error("Failed to send share notifications");
  }
}
```

## ketrics.Users

List users in the current tenant.

```typescript
const tenantUsers = await ketrics.Users.list();
// Returns: { id: string, firstName: string, lastName: string, email: string }[]

const users = tenantUsers.map(user => ({
  userId: user.id,
  name: `${user.firstName} ${user.lastName}`.trim(),
  email: user.email,
}));
```

## ketrics.application

```typescript
ketrics.application.id  // Application UUID, useful for namespacing Volume file keys
```

## ketrics.console

```typescript
ketrics.console.error("Error message for logs");
```

Use for non-critical errors where you don't want to throw and fail the handler.

## ketrics.Job

Run long-running operations asynchronously in the background.

### runInBackground

```typescript
const jobId = await ketrics.Job.runInBackground({
  function: "_syncMyBackgroundTask",  // Handler name — must be in actions array
  payload: { entityId, data },         // Passed as the handler's payload argument
});
```

### Background handler naming convention

Prefix background handler names with `_` (e.g., `_syncCreateRecord`). These handlers are invoked by the platform, not the frontend. They **must** be listed in `ketrics.config.json` `actions` and exported from `backend/src/index.ts`.

### Double try-catch error handling

Background handlers should use a double try-catch to prevent entities from getting stuck in intermediate states:

```typescript
const _syncMyBackgroundTask = async (payload: { entity: MyEntity; data: unknown }) => {
  const { entity, data } = payload;
  try {
    // Main operation
    const result = await performOperation(data);

    // Success: update entity to final state
    const docdb = await ketrics.DocumentDb.connect(getDocDbCode());
    const updated = { ...entity, estado: "COMPLETED", updatedAt: new Date().toISOString() };
    await docdb.put(pk, sk, updated as unknown as Record<string, unknown>);

    // Notify the user who triggered the job
    ketrics.Messages.send({
      userId: ketrics.requestor.userId!,
      type: "notification",
      subject: "Task completed",
      body: `Operation finished successfully.`,
    });
  } catch (err) {
    // Outer catch: set ERROR state so the entity doesn't stay in intermediate state
    try {
      const docdb = await ketrics.DocumentDb.connect(getDocDbCode());
      const errorEntity = {
        ...entity,
        estado: "ERROR",
        errorMessage: err instanceof Error ? err.message : "Unexpected error",
        updatedAt: new Date().toISOString(),
      };
      await docdb.put(pk, sk, errorEntity as unknown as Record<string, unknown>);
    } catch {
      // If even error-write fails, notify the user as last resort
      ketrics.Messages.send({
        userId: ketrics.requestor.userId!,
        type: "notification",
        subject: "Operation failed",
        body: `Could not complete the operation.`,
      });
    }
  }
};
```

### Launching a background job from a handler

The calling handler should:
1. Validate state (e.g., entity must be `APROBADA`)
2. Transition to intermediate state (`EN_PROCESO`) **before** launching the job
3. Launch `runInBackground`, store the returned `jobId` on the entity

```typescript
const processEntity = async (payload: { id: string }) => {
  requireEditor();
  const docdb = await ketrics.DocumentDb.connect(getDocDbCode());
  const entity = await docdb.get(pk, sk) as unknown as MyEntity;
  if (entity.estado !== "APROBADA") throw new Error("Must be APROBADA");

  // Transition to intermediate state first
  const updated = { ...entity, estado: "EN_PROCESO" as const, updatedAt: new Date().toISOString() };
  await docdb.put(pk, sk, updated as unknown as Record<string, unknown>);

  // Launch background job
  const jobId = await ketrics.Job.runInBackground({
    function: "_syncMyBackgroundTask",
    payload: { entity: updated, data: { /* ... */ } },
  });

  // Store jobId for tracking
  updated.jobId = jobId;
  await docdb.put(pk, sk, updated as unknown as Record<string, unknown>);

  return { jobId, entity: updated };
};
```

## ketrics.Applications

Invoke handlers in other Ketrics applications (cross-app communication).

### Connect and invoke

```typescript
import { ApplicationInvokeResult } from "@ketrics/sdk-backend";

const appCode = ketrics.environment["OTHER_APP_CODE"];
const app = await ketrics.Applications.connect(appCode);

const result: ApplicationInvokeResult<ResponseType> = await app.invoke<PayloadType, ResponseType>(
  "handlerName",
  payload
);

if (result.success) {
  // result.result contains the typed response
  const data = result.result;
} else {
  // Handle failure — extract error from result
  const errorMsg = (result as any).error?.message || "Unknown error";
}
```

Store app codes in environment variables so they can be configured per-tenant without code changes.

## ketrics.http

HTTP client for making external API requests. Supports `get`, `post`, `put`, and `delete` methods.

### GET — Fetch a resource

```typescript
const getUser = async (payload: { userId: string }) => {
  const { userId } = payload;

  const response = await ketrics.http.get(`https://api.example.com/users/${userId}`);

  if (response.status !== 200) {
    return { success: false, status: response.status, error: response.statusText };
  }

  return { success: true, user: response.data };
};
```

### GET — With query params and custom headers

```typescript
const searchUsers = async (payload: { query: string; page?: number; apiKey: string }) => {
  const { query, page = 1, apiKey } = payload;

  const response = await ketrics.http.get("https://api.example.com/users", {
    params: { q: query, page, limit: 20 },
    headers: { Authorization: `Bearer ${apiKey}` },
    timeout: 10000, // 10s instead of the default 30s
  });

  return {
    results: response.data,
    status: response.status,
    headers: response.headers,
  };
};
```

### POST — Create a resource with a JSON body

```typescript
const createOrder = async (payload: { customerId: string; items: unknown[]; apiKey: string }) => {
  const { customerId, items, apiKey } = payload;

  const response = await ketrics.http.post("https://api.example.com/orders", {
    customerId,
    items,
    createdAt: new Date().toISOString(),
  }, {
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
  });

  if (response.status === 201) {
    return { success: true, orderId: response.data.id };
  }

  return { success: false, status: response.status, error: response.data };
};
```

### PUT — Update a resource

```typescript
const updateUser = async (payload: { userId: string; name: string; email: string; apiKey: string }) => {
  const { userId, name, email, apiKey } = payload;

  const response = await ketrics.http.put(`https://api.example.com/users/${userId}`, {
    name,
    email,
  }, {
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  return { success: response.status === 200, data: response.data };
};
```

### DELETE — Remove a resource

```typescript
const deleteUser = async (payload: { userId: string; apiKey: string }) => {
  const { userId, apiKey } = payload;

  const response = await ketrics.http.delete(`https://api.example.com/users/${userId}`, {
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  return { success: response.status === 204 };
};
```

### Full workflow — Chaining multiple requests

```typescript
const syncCustomerData = async (payload: { externalApiUrl: string; apiKey: string }) => {
  const { externalApiUrl, apiKey } = payload;

  // 1. Fetch customers from external system
  const listResponse = await ketrics.http.get(`${externalApiUrl}/customers`, {
    headers: { Authorization: `Bearer ${apiKey}` },
    params: { status: "active", limit: 100 },
  });

  if (listResponse.status !== 200) {
    return { success: false, error: `Failed to fetch customers: ${listResponse.statusText}` };
  }

  const customers = listResponse.data;

  // 2. Post each customer to our internal API
  const results = [];
  for (const customer of customers) {
    const saveResponse = await ketrics.http.post("https://internal-api.example.com/customers", {
      externalId: customer.id,
      name: customer.name,
      email: customer.email,
    });

    results.push({
      externalId: customer.id,
      status: saveResponse.status,
      saved: saveResponse.status === 201,
    });
  }

  return {
    success: true,
    total: customers.length,
    saved: results.filter((r) => r.saved).length,
    failed: results.filter((r) => !r.saved).length,
  };
};
```

### Response object

All methods return a response object with:

```typescript
response.status     // number — HTTP status code (200, 201, 404, etc.)
response.statusText // string — HTTP status text ("OK", "Created", etc.)
response.data       // unknown — Parsed response body (JSON by default)
response.headers    // Record<string, string> — Response headers
```

### Options

```typescript
{
  params?: Record<string, unknown>;   // Query string parameters
  headers?: Record<string, string>;   // Request headers
  timeout?: number;                   // Timeout in ms (default: 30000)
}
```

## DocumentDB type casting pattern

The SDK requires `Record<string, unknown>` for writes but returns it for reads. Use double casts:

```typescript
// Writing typed objects → double cast to satisfy SDK
const item: MyInterface = { id: "123", name: "test", status: "active" };
await docdb.put(pk, sk, item as unknown as Record<string, unknown>);

// Reading → cast SDK response to your interface
const raw = await docdb.get(pk, sk);
const typed = raw as unknown as MyInterface;

// List results
const result = await docdb.list(pk, { skPrefix: "ITEM#" });
const items = result.items as unknown as MyInterface[];
```

This pattern is used throughout production code and is required whenever storing typed objects in DocumentDB.

## Duplicate prevention index pattern

Prevent the same item from being used in multiple active entities (e.g., a document can only be in one active nomina):

```typescript
// Key design: pk=ACTIVE_ITEMS#{scope}  sk=ITEM#{itemKey}

// Check before adding
const existing = await docdb.get(`ACTIVE_ITEMS#${scope}`, `ITEM#${itemKey}`);
if (existing) throw new Error("Item is already in use in another active entity");

// Add to index when item is added to an entity
await docdb.put(`ACTIVE_ITEMS#${scope}`, `ITEM#${itemKey}`, {
  itemKey,
  entityId,
  addedAt: new Date().toISOString(),
});

// Remove from index when item is removed or entity is completed/deleted
await docdb.delete(`ACTIVE_ITEMS#${scope}`, `ITEM#${itemKey}`);

// Bulk load for UI display (mark items as "in use")
const activeResult = await docdb.list(`ACTIVE_ITEMS#${scope}`, { skPrefix: "ITEM#", limit: 5000 });
const activeMap = new Map<string, string>();
for (const item of activeResult.items) {
  activeMap.set(item.itemKey as string, item.entityId as string);
}
// When listing available items, check: activeMap.has(itemKey)
```

## State machine pattern

Define entity states as a union type and validate transitions in each handler:

```typescript
type EntityState = "PENDIENTE" | "APROBADA" | "EN_PROCESO" | "CONTABILIZADA" | "ERROR";

interface MyEntity {
  id: string;
  estado: EntityState;
  // ...
}
```

### Transition rules

- **PENDIENTE**: Items can be added/removed/modified. Can transition to `APROBADA` or be deleted.
- **APROBADA**: Locked for modifications. Can transition to `EN_PROCESO` (background job) or back to `PENDIENTE`.
- **EN_PROCESO**: Intermediate state during async operations. Background job transitions to `CONTABILIZADA` or `ERROR`.
- **CONTABILIZADA**: Final success state. Admin can revert to `APROBADA`.
- **ERROR**: Background job failed. Admin can revert to `APROBADA` to retry.

### Validating transitions in handlers

```typescript
const approveEntity = async (payload: { id: string }) => {
  requireApprover();
  const entity = await docdb.get(pk, sk) as unknown as MyEntity;
  if (entity.estado !== "PENDIENTE") {
    throw new Error("Can only approve entities in PENDIENTE state");
  }
  // Additional business rules (e.g., must have items)
  if (entity.totalItems === 0) {
    throw new Error("Cannot approve an empty entity");
  }
  // ...
};

const revertEntity = async (payload: { id: string }) => {
  requireAdmin();
  const entity = await docdb.get(pk, sk) as unknown as MyEntity;
  if (entity.estado !== "CONTABILIZADA" && entity.estado !== "ERROR") {
    throw new Error("Can only revert CONTABILIZADA or ERROR entities");
  }
  // ...
};
```

## Dynamic SQL table naming

When building SQL with dynamic table names (e.g., per-company tables), validate identifiers before interpolation:

```typescript
const VALID_IDENTIFIER = /^[a-zA-Z0-9_]+$/;

if (!VALID_IDENTIFIER.test(empresa)) {
  throw new Error("Invalid table name identifier");
}
const tableName = `"prod"."${empresa}_softland_saldos_docs"`;
const sql = `SELECT * FROM ${tableName} WHERE saldo > 0`;
```

Never interpolate user input into SQL without validation. Use parameterized queries (`$1`, `$2`) for values, and regex validation for identifiers.

## Comment system pattern

Comments use the CRUD handler template with chronological sort keys and a count index. Key design:

```typescript
// Store: pk=COMMENTS#{targetId}  sk=COMMENT#{createdAt}#{commentId}
// Index: pk=COMMENTINDEX#main     sk=TARGET#{targetId}  → { count, lastCommentAt }
// On add: increment count index. On delete: decrement (or remove if count <= 1).
// List: docdb.list(`COMMENTS#${targetId}`, { skPrefix: "COMMENT#", scanForward: false })
```

## Sharing and access control pattern

For resources with `owner`, `visibility` ("private"|"shared"), and `sharedWith` (string[]|"all") fields:

```typescript
// Access check helper — reuse in get/list/update handlers
function checkAccess(item: Record<string, unknown>, userId: string): void {
  if (item.owner === userId) return;
  if (item.visibility !== "shared") throw new Error("Access denied");
  const sw = item.sharedWith;
  if (sw === "all") return;
  if (Array.isArray(sw) && sw.includes(userId)) return;
  throw new Error("Access denied");
}

// List filter: items.filter(item => { checkAccess returns or throws })
```
