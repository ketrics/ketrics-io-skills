# Backend SDK Reference

Complete reference for the `ketrics` global object and backend handler patterns. The `ketrics` object is automatically available in all backend handlers — no imports needed. Types come from `@ketrics/sdk-backend` (devDependency).

## Handler function pattern

Every handler is an async function that takes a typed payload and returns a result. Handlers are exported by name from `backend/src/index.ts` and listed in the `functions` array in `ketrics.config.json`. (The `actions` array is something different — it holds the capabilities the app enforces, not handler names. See `CONFIG_AND_DEPLOY.md`.)

```typescript
interface MyPayload {
  id: string;
  name: string;
}

const myHandler = async (payload: MyPayload) => {
  requirePermission("write"); // capability check
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

Read application environment variables. Declare each one in `ketrics.config.json` under `environment` — on deploy, any declared variable that does not yet exist is created with an empty value for an admin to fill in the Ketrics dashboard. (The key is `environment`; the old `environmentVariables` name now fails the deploy outright.) Variables bound to a `resources` entry are declared implicitly and need no `environment` entry of their own.

```typescript
// Simple string value
const apiKey = ketrics.environment["API_KEY"];

// JSON-encoded config
const raw = ketrics.environment["DB_CONNECTIONS"];
const connections: { code: string; name: string }[] = JSON.parse(raw);
```

### Never hardcode resource codes

DocumentDB, Volume, Secret, Parameter and Connection codes differ per tenant and per environment. They must **never** appear as string literals in handler code — read them from `ketrics.environment`, and **never fall back to a hardcoded default**.

```typescript
// ❌ never — hardcoded code, and a literal fallback is just as bad
const docdb = await ketrics.DocumentDb.connect("app-data");
const code = ketrics.environment["APP_DATA_DOCDB"] || "default-store";

// ✅ always — read the code from the environment, fail loudly if missing
const docdb = await ketrics.DocumentDb.connect(ketrics.environment["APP_DATA_DOCDB"]);
```

Define a small accessor in `helpers.ts` for each resource so the env-var name lives in exactly one place and a missing value fails with a clear message:

```typescript
// helpers.ts
function requireEnv(name: string): string {
  const value = ketrics.environment[name];
  if (!value) throw new Error(`Environment variable ${name} is not configured`);
  return value;
}

// One accessor per resource — the name matches the resource `code` in ketrics.config.json
export const getDocDbCode = (): string => requireEnv("APP_DATA_DOCDB");
export const getExportsVolumeCode = (): string => requireEnv("EXPORTS_VOLUME");
export const getMainDbConnectionCode = (): string => requireEnv("MAIN_DB_CONNECTION");
export const getBillingConfigCode = (): string => requireEnv("BILLING_CONFIG_PARAMETER");
```

Common environment variables:

- `*_DOCDB` — DocumentDB resource code (one env var per DocumentDB resource)
- `*_VOLUME` — Volume resource code (one env var per Volume)
- `*_SECRET` — Secret resource code (one env var per Secret)
- `*_PARAMETER` — Parameter resource code (one env var per Parameter)
- `*_CONNECTION` — SQL data connection code (one env var per Connection)
- `DB_CONNECTIONS` — JSON array of `{code, name}` when the app offers a connection dropdown
- `*_APP_CODE` — application code for cross-app invocation

### Environment variable, or Parameter?

Both carry configuration, and the split is about **scope and shape**:

- **`environment`** — a single string, private to this application, edited per application in the
  portal. Right for a threshold, a base URL, a feature flag, and for the resource codes above.
- **`ketrics.Parameter`** — a JSON **object**, owned by the tenant and read by **several
  applications**. Right for a shared chart of accounts, tax tables, a company registry: edit once,
  every app that declares the parameter sees the change on its next invocation.

Neither is for credentials. Those go in `ketrics.Secret`, which is encrypted.

## ketrics.Parameter

Shared, **unencrypted** JSON configuration — the non-secret twin of `ketrics.Secret`. A parameter
is a tenant-level JSON object that several applications can read, so config that would otherwise
be copied into each app's environment lives in one place.

Requires `@ketrics/sdk-backend` >= 0.17.0 — that release added the typings. If `tsc` reports that
`Parameter` does not exist on the `ketrics` object, the devDependency is older.

```typescript
const config = await ketrics.Parameter.get(ketrics.environment["BILLING_CONFIG_PARAMETER"]);
console.log(config.currency);
```

`get()` returns the value **already parsed** — not a string, so no `JSON.parse` at the call site.
Type it with the generic:

```typescript
interface BillingConfig {
  currency: string;
  taxRate: number;
  paymentTerms: { code: string; days: number }[];
}

const config = await ketrics.Parameter.get<BillingConfig>(getBillingConfigCode());
```

`exists()` answers without throwing, for genuinely optional config:

```typescript
const code = ketrics.environment["FEATURE_FLAGS_PARAMETER"];
const flags = (await ketrics.Parameter.exists(code))
  ? await ketrics.Parameter.get<FeatureFlags>(code)
  : DEFAULTS;
```

It returns `false` both when the parameter does not exist and when the app has no grant for it —
the two are deliberately indistinguishable, so an app cannot probe for codes outside its scope.

### Declare the parameter in ketrics.config.json

Access is granted by declaration, not by knowing the code:

```json
"resources": {
  "parameter": [
    { "code": "billing-config", "description": "Currency, tax rate and invoice terms" }
  ]
}
```

Deploy binds it to `BILLING_CONFIG_PARAMETER` (derived, so it needs no `environment` entry) and
seeds `parameter:billing-config` on the application role. Reading a parameter the app never
declared throws `AccessDeniedError`. As with every resource, read the code from the environment —
never a string literal.

### The returned object is frozen, and memoized per invocation

- **Deep-frozen.** Mutating any level of the returned object fails (silently in sloppy mode,
  with a `TypeError` under `"use strict"`). Copy before changing anything locally:
  `const local = { ...config, currency: "USD" };`
- **Memoized for the invocation.** A successful `get()` is cached, so reading the same parameter
  in five helpers costs one lookup and returns the same object. The flip side: a parameter edited
  while an invocation is running is not observed until the next invocation. Failed lookups are
  never cached, so a denial cannot outlive the grant that fixed it.

Cache it in a module-level variable only if you understand the consequence — the sandbox may reuse
a warm process across invocations, so a hand-rolled cache can outlive the memo and go stale.

### Error handling

```typescript
try {
  const config = await ketrics.Parameter.get<BillingConfig>(getBillingConfigCode());
  return { taxRate: config.taxRate };
} catch (error) {
  if (error instanceof ketrics.Parameter.NotFoundError) {
    throw new Error("Billing configuration has not been created for this tenant");
  }
  if (error instanceof ketrics.Parameter.AccessDeniedError) {
    throw new Error("This application is not granted access to the billing configuration");
  }
  throw error;
}
```

| Error | Thrown when |
|-------|-------------|
| `ketrics.Parameter.NotFoundError` | No parameter with that code exists in the tenant |
| `ketrics.Parameter.AccessDeniedError` | The app has no grant for it — usually a missing `resources.parameter` declaration |
| `ketrics.Parameter.InvalidJsonError` | The stored value is not a JSON object (only possible for a record written outside the API) |

Every error carries `parameterCode`, `operation` and `timestamp`, and `ketrics.Parameter.isError`
/ `isErrorType` are available as type guards.

### Limits and the security rule

- The value must be a JSON **object** — arrays, scalars and `null` are rejected at write time.
- Serialized value: max **64 KiB**. Parameters per tenant: **100** (1000 on Enterprise).
- **Values are stored in plaintext and are returned by the API.** Anyone holding
  `parameter:DescribeParameter` for a matching pattern can read one. **Credentials, API keys and
  tokens belong in `ketrics.Secret`, never in a Parameter.**

## ketrics.requestor

Context about the authenticated user making the request.

```typescript
ketrics.requestor.userId; // string — User ID
ketrics.requestor.name; // string — Display name
ketrics.requestor.email; // string — Email address
ketrics.requestor.applicationPermissions; // string[] — granted capabilities, e.g. ["read", "write"] (or ["*"])
```

`applicationPermissions` holds the **capabilities** the requestor was granted through their roles — the same codes declared in `actions` in `ketrics.config.json` — or `["*"]` for full access. Check against a capability, never against a role code.

### Permission checking pattern

`permissions.ts` owns the app's capability list and a single typed guard:

```typescript
// permissions.ts

/** The permissions this application enforces. Keep in sync with `actions` in ketrics.config.json. */
export const PERMISSIONS = ["read", "write", "export", "approve"] as const;

export type Permission = (typeof PERMISSIONS)[number];

export function requirePermission(permission: Permission): void {
  const granted = ketrics.requestor.applicationPermissions;
  if (!granted.includes("*") && !granted.includes(permission)) {
    throw new Error(`Permission denied: "${permission}" required`);
  }
}
```

```typescript
// Usage: call at the start of handlers — different handlers require different capabilities
const createItem = async (payload: CreatePayload) => {
  requirePermission("write"); /* ... */
};
const approveItem = async (payload: ApprovePayload) => {
  requirePermission("approve"); /* ... */
};
const exportItems = async (payload: ExportPayload) => {
  requirePermission("export"); /* ... */
};
```

Named sugar is fine where it reads better, as long as it goes through the guard:

```typescript
export const requireApprove = () => requirePermission("approve");
```

`PERMISSIONS` is per-application: declare the capabilities *your* handlers enforce. The four above are a common starting set, not a fixed vocabulary — an app whose handlers must distinguish reverting a posted document from approving one declares a `revert` capability too. Whatever you choose, the same codes must appear in `actions` in `ketrics.config.json`; the two lists are hand-synced and nothing verifies them.

#### Why the parameter is typed

`requirePermission` takes a `Permission`, not a `string`, so a typo is a **compile error**:

```
error TS2345: Argument of type '"wirte"' is not assignable to parameter of type
'"read" | "write" | "export" | "approve"'.
```

With an untyped `requireAction(action: string)`, `requireAction("wirte")` compiles happily and produces a handler **no role can ever reach**: nobody is granted `wirte`, so every call throws, and nothing anywhere reports a problem. It surfaces months later as "that button doesn't work".

#### Never check a role code

The older convention defined one helper per role — `requireEditor()` testing `applicationPermissions.includes("editor")`. That compares an **action list against a role name** and can never match, so the handler is dead for every user. `applicationPermissions` contains capabilities; roles are how capabilities get granted, and their codes never appear in it.

Capabilities are also what make partial combinations possible: with `read`/`write`/`export`/`approve` as the atoms, a role can grant `approve` without `write`. Role-named actions collapse `roles` into a 1:1 restatement of `actions` and force a code change every time someone needs a new combination.

## ketrics.Database

Connect to SQL databases configured for the tenant. Supports parameterized queries.

### Connect and query/execute

The `db` object provides two methods for running SQL:

- **`db.query(sql, params)`** — Use for **SELECT** statements. Returns `{ rows, rowCount }`.
- **`db.execute(sql, params)`** — Use for **INSERT**, **UPDATE**, and **DELETE** statements.

```typescript
// Connection code comes from an environment variable — never a literal
const db = await ketrics.Database.connect(getMainDbConnectionCode());
try {
  // SELECT → use query()
  const result = await db.query<Record<string, unknown>>(sql, params);
  // result.rows: Record<string, unknown>[]
  // result.rowCount: number

  // INSERT/UPDATE/DELETE → use execute()
  await db.execute(insertSql, params);
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

### Parameterized queries — Placeholder syntax by database type

**IMPORTANT:** The placeholder syntax depends on the database engine:

| Database               | Placeholder         | Example                           |
| ---------------------- | ------------------- | --------------------------------- |
| **PostgreSQL**         | `$1`, `$2`, `$3`... | `WHERE status = $1 AND role = $2` |
| **MSSQL (SQL Server)** | `?`                 | `WHERE status = ? AND role = ?`   |

For **PostgreSQL** connections:

```typescript
const sql = "SELECT * FROM users WHERE status = $1 AND role = $2";
const params = ["active", "admin"];
const result = await db.query<Record<string, unknown>>(sql, params);
```

For **MSSQL** connections (e.g., Softland):

```typescript
const sql = "SELECT * FROM users WHERE status = ? AND role = ?";
const params = ["active", "admin"];
const result = await db.query<Record<string, unknown>>(sql, params);
```

When building dynamic IN clauses with MSSQL:

```typescript
const placeholders = cuentas.map(() => "?").join(", ");
const sql = `SELECT * FROM tabla WHERE ano = ? AND codigo IN (${placeholders})`;
const result = await db.query(sql, [ano, ...cuentas]);
```

### Table naming syntax by database type

| Database               | Syntax          | Example                   |
| ---------------------- | --------------- | ------------------------- |
| **PostgreSQL**         | Double quotes   | `"schema"."table"`        |
| **MSSQL (SQL Server)** | Square brackets | `[database].schema.table` |

MSSQL example with dynamic company name:

```typescript
const sql = `SELECT * FROM [${empresa}].softland.cwmovim WHERE CpbAno = ?`;
```

### Template parameter replacement (PostgreSQL)

Convert `{{paramName}}` placeholders to positional parameters. This pattern uses `$N` syntax and is for **PostgreSQL** connections. For MSSQL, replace `$${paramIndex}` with `"?"`:

```typescript
// Input: "SELECT * FROM users WHERE status = {{status}}"
// Output (PostgreSQL): "SELECT * FROM users WHERE status = $1"  with values = ["active"]
// Output (MSSQL):      "SELECT * FROM users WHERE status = ?"   with values = ["active"]

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
  parameterizedSql = parameterizedSql.replace(new RegExp(`\\{\\{${pName}\\}\\}`, "g"), () => `$${paramIndex}`);
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
parameterizedSql = parameterizedSql.replace(/\{\{#if\s+(\w+)\}\}([\s\S]*?)\{\{\/if\}\}/g, (_, condParam: string, content: string) => {
  const val = paramValues[condParam];
  return val != null && val.trim() !== "" ? content : "";
});
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
// The resource code is read from an environment variable, never hardcoded.
// getDocDbCode() is the helpers.ts accessor defined in the ketrics.environment section.
const docdb = await ketrics.DocumentDb.connect(getDocDbCode());
```

The env-var name (`APP_DATA_DOCDB` in the example helper) is the variable bound to an entry under `resources.documentdb` in `ketrics.config.json` — either the entry's explicit `environmentVariable`, or the name derived from its kind and `code` (`app-data` + `documentdb` → `APP_DATA_DOCDB`). A derived name needs no `environment` entry; an explicit one does.

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
  skPrefix: "ITEM#", // Filter by sort key prefix
  scanForward: false, // Reverse chronological order
  limit: 100, // Max items to return
});
// result.items: Record<string, unknown>[]
// result.cursor: string | undefined — present when more results exist
```

### Pagination with cursor

When a `list()` call returns more items than the `limit`, the response includes a `cursor` string. Pass it back to fetch the next page:

```typescript
const docdb = await ketrics.DocumentDb.connect(getDocDbCode());

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
  cursor: someCursor, // from a previous query result
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
pk = `USER#${userId}`;
sk = `ITEM#${itemId}`;
```

**Tenant-wide items (shared across all users):**

```typescript
pk = `TENANT_ITEMS`;
sk = `ITEM#${itemId}`;
```

**Comments on a target entity:**

```typescript
pk = `COMMENTS#${connectionCode}#${targetKey}`;
sk = `COMMENT#${createdAt}#${commentId}`;
// Using createdAt in sk gives chronological ordering
```

**Count index for fast lookups:**

```typescript
pk = `INDEX#${scope}`;
sk = `KEY#${lookupKey}`;
// Store: { lookupKey, count: N, lastUpdatedAt }
```

### CRUD handler template

```typescript
// CREATE
const createItem = async (payload: CreatePayload) => {
  requirePermission("write");
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
  requirePermission("write");
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
  requirePermission("write");
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
sheet.addRows(data.map((row) => [row.name, row.email, row.status]));

// Get buffer for saving to Volume
const buffer = await excel.toBuffer();
```

### Dynamic column widths

```typescript
sheet.columns = columns.map((col) => ({
  header: col,
  key: col,
  width: Math.max(12, col.length + 4),
}));
```

## ketrics.Volume

File storage with presigned download URLs.

```typescript
// Volume code comes from an environment variable — never a literal.
// getExportsVolumeCode() is the helpers.ts accessor from the ketrics.environment section.
const volume = await ketrics.Volume.connect(getExportsVolumeCode());

// Upload a file
const fileKey = `${ketrics.application.id}/${filename}`;
await volume.put(fileKey, buffer);

// Generate download URL — ALWAYS pass responseContentDisposition for browser downloads
const presigned = await volume.generateDownloadUrl(fileKey, {
  responseContentDisposition: attachmentDisposition(filename),
});
// presigned.url — time-limited download URL with Content-Disposition: attachment
```

### Why `responseContentDisposition` is required

The app frontend runs in an iframe under a strict `frame-src https://cdn.ketrics.io` CSP. Presigned Volume URLs point at the S3 bucket host, which is *not* in that allowlist. When the browser follows the URL without `Content-Disposition: attachment` set, it tries to render the file by navigating the iframe to S3 — CSP blocks the navigation and the download silently fails (the user sees an error like `Framing 'https://ketrics-volumes-...amazonaws.com/' violates the following Content Security Policy directive: "frame-src https://cdn.ketrics.io"`). Forcing `Content-Disposition: attachment` short-circuits the rendering path — the browser starts a download instead of navigating, so the CSP never applies. This applies to every Volume URL the frontend hits, not just "export" buttons (PDF preview, comprobante download, etc. all need it).

### `attachmentDisposition` helper

Filenames often contain accents, spaces, or other non-ASCII characters. Use RFC 5987 encoding so both legacy and modern browsers get a usable filename:

```typescript
function attachmentDisposition(filename: string): string {
  const ascii = filename.replace(/[^\x20-\x7E]/g, "_").replace(/"/g, "");
  return `attachment; filename="${ascii}"; filename*=UTF-8''${encodeURIComponent(filename)}`;
}
```

### Complete export pattern (Excel + Volume)

```typescript
const exportData = async (payload: { data: Record<string, unknown>[] }) => {
  requirePermission("export");
  const { data } = payload;
  if (!data?.length) throw new Error("No data to export");

  const volumeCode = ketrics.environment["EXPORTS_VOLUME"];
  if (!volumeCode) throw new Error("EXPORTS_VOLUME not configured");

  const columns = Object.keys(data[0]);

  // Build Excel
  const excel = ketrics.Excel.create();
  const sheet = excel.addWorksheet("Export");
  sheet.columns = columns.map((col) => ({
    header: col,
    key: col,
    width: Math.max(12, col.length + 4),
  }));
  sheet.addRows(data.map((row) => columns.map((col) => row[col] ?? "")));
  const buffer = await excel.toBuffer();

  // Save to Volume
  const filename = `export_${Date.now()}.xlsx`;
  const fileKey = `${ketrics.application.id}/${filename}`;
  const volume = await ketrics.Volume.connect(volumeCode);
  await volume.put(fileKey, buffer);
  const { url } = await volume.generateDownloadUrl(fileKey, {
    responseContentDisposition: attachmentDisposition(filename),
  });

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
  priority: "MEDIUM", // "LOW" | "MEDIUM" | "HIGH"
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

const users = tenantUsers.map((user) => ({
  userId: user.id,
  name: `${user.firstName} ${user.lastName}`.trim(),
  email: user.email,
}));
```

## ketrics.application

```typescript
ketrics.application.id; // Application UUID, useful for namespacing Volume file keys
```

## ketrics.console

```typescript
ketrics.console.error("Error message for logs");
```

The SDK exposes `log`, `error`, `warn`, `info`, and `debug` — each forwards to CloudWatch tagged with the matching severity, so you can filter by level there. Use it for non-critical errors where you don't want to throw and fail the handler. But don't sprinkle raw `ketrics.console.*` calls through a real app — wrap them in the leveled logger below.

## Logging: DEBUG_LEVEL-gated logger

### Why this exists

Once an app is deployed, CloudWatch is your only window into what it's doing — and the things that break in production (a sync that silently stores nothing, an external API quietly returning 401, a filter dropping every row) are exactly the things you can't reproduce locally. You want rich, stage-by-stage logging available on demand, but you don't want that verbosity (or its cost and noise) running all the time, and you don't want to redeploy code just to turn diagnostics on.

The fix is a tiny logger gated by a `DEBUG_LEVEL` environment variable. An administrator sets `DEBUG_LEVEL=debug` in the dashboard to capture a full trace of a misbehaving flow, reads the CloudWatch lines, then sets it back to `error` — all without a code change. This pattern was the difference between "no new videos appear, no idea why" and pinpointing a revoked OAuth token in one sync cycle: the silent failure became a labeled `[syncChannels] failed … invalid_grant` line.

### The logger

Create `src/logger.ts` verbatim — it reads `DEBUG_LEVEL` **once at module load** (env vars are static per deployment, so re-reading per call is wasted work):

```typescript
// src/logger.ts
type LevelName = "none" | "error" | "warn" | "info" | "debug";

const LEVELS: Record<LevelName, number> = { none: 0, error: 1, warn: 2, info: 3, debug: 4 };

const resolveLevel = (): number => {
  const raw = (ketrics.environment["DEBUG_LEVEL"] ?? "").toString().trim().toLowerCase();
  if (raw in LEVELS) return LEVELS[raw as LevelName];
  return LEVELS.error; // unset / unrecognized → errors only (preserves default behavior)
};

const active = resolveLevel();

const fmt = (scope: string, msg: string) => `[${scope}] ${msg}`;

export const log = {
  error: (scope: string, msg: string, data?: unknown) =>
    active >= LEVELS.error && ketrics.console.error(fmt(scope, msg), data ?? ""),
  warn: (scope: string, msg: string, data?: unknown) =>
    active >= LEVELS.warn && ketrics.console.warn(fmt(scope, msg), data ?? ""),
  info: (scope: string, msg: string, data?: unknown) =>
    active >= LEVELS.info && ketrics.console.info(fmt(scope, msg), data ?? ""),
  debug: (scope: string, msg: string, data?: unknown) =>
    active >= LEVELS.debug && ketrics.console.debug(fmt(scope, msg), data ?? ""),
};
```

Levels are ordered `none < error < warn < info < debug`, and each level **includes the ones above it** — `info` emits error + warn + info, `debug` emits everything, `none` silences all of it. Unset defaults to `error`, which matches the behavior of an app that only ever called `ketrics.console.error`, so adopting the logger never quietly drops your existing error logs.

Declare the variable in `ketrics.config.json`:

```json
{ "name": "DEBUG_LEVEL", "description": "Backend log verbosity: none | error | warn | info | debug (each level includes the levels above it). Unset defaults to error." }
```

### How to use it

The first argument is a **scope** — usually the function or subsystem name. It prefixes every line as `[scope]` so the logs are greppable in CloudWatch (`filter [syncChannels]`). Pick the level by audience:

| Level   | Use for                                                                                          |
| ------- | ------------------------------------------------------------------------------------------------ |
| `error` | A real failure: a thrown exception you caught, an external call that won't recover.              |
| `warn`  | Something suspicious but survivable: an empty result where you expected rows, a fallback taken.  |
| `info`  | One line per significant operation — start/end of a sync, counts in and counts out.              |
| `debug` | Per-item detail: each record written, request params, raw vs. filtered counts, the exact key.    |

```typescript
import { log } from "./logger";

const syncChannels = async (userId: string, channelIds: string[]) => {
  log.info("syncChannels", `syncing ${channelIds.length} channel(s)`);
  for (const channelId of channelIds) {
    try {
      const videos = await fetchChannelRecentVideos(userId, channelId);
      log.debug("syncChannels", `${channelId}: fetched ${videos.length}`);
    } catch (err) {
      log.error("syncChannels", `failed for ${channelId}: ${err instanceof Error ? err.message : err}`);
    }
  }
  log.info("syncChannels", `done: total=${total} errors=${errors.length}`);
};
```

### Where to put log statements

Log at the **boundaries where data can silently disappear** — that's where a count that drops to zero tells you exactly which stage failed:

- **Entry/exit of each sync or batch job**: count in, count out (`info`). When the input count is non-zero but the output is zero, you know the loss is inside.
- **Around every external call** (`ketrics.http`, cross-app `invoke`, token refresh): the request params (`debug`) and the response status (`debug`/`warn`). Remember `ketrics.http` does **not** throw on non-2xx — it returns the response with the error status — so a quiet 401/403 looks identical to an empty result unless you log `response.status`.
- **Read/filter paths**: raw count fetched vs. count after each filter (`info`). This distinguishes "the data was never stored" from "the data is stored but a filter hides it."
- **Each write**: the storage key being written (`debug`). Upserts that collide on a key overwrite instead of adding, which looks like "nothing new appeared."

### Guardrails

- **Never log secrets, tokens, refresh tokens, the client secret, or PII.** Log counts, IDs, storage keys, statuses, and timestamps — the shape of the data, not the data itself. When logging an HTTP error body, you're logging the provider's error JSON, not credentials; still scan it before adding the line.
- Keep message construction cheap. Heavy object dumps belong only at `debug`, behind the level gate.
- One scope per file/subsystem keeps CloudWatch filtering clean — reuse the same scope string for all lines in a given function.

## ketrics.Job

Run long-running operations asynchronously in the background.

### runInBackground

```typescript
const jobId = await ketrics.Job.runInBackground({
  function: "_syncMyBackgroundTask", // Handler name — must be exported and listed in functions
  payload: { entityId, data }, // Passed as the handler's payload argument
});
```

### Background handler naming convention

Prefix background handler names with `_` (e.g., `_syncCreateRecord`). These handlers are invoked by the platform, not the frontend. They **must** be exported from `backend/src/index.ts` and listed in `ketrics.config.json` `functions`.

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
  requirePermission("write");
  const docdb = await ketrics.DocumentDb.connect(getDocDbCode());
  const entity = (await docdb.get(pk, sk)) as unknown as MyEntity;
  if (entity.estado !== "APROBADA") throw new Error("Must be APROBADA");

  // Transition to intermediate state first
  const updated = { ...entity, estado: "EN_PROCESO" as const, updatedAt: new Date().toISOString() };
  await docdb.put(pk, sk, updated as unknown as Record<string, unknown>);

  // Launch background job
  const jobId = await ketrics.Job.runInBackground({
    function: "_syncMyBackgroundTask",
    payload: {
      entity: updated,
      data: {
        /* ... */
      },
    },
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

const result: ApplicationInvokeResult<ResponseType> = await app.invoke<PayloadType, ResponseType>("handlerName", payload);

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

  const response = await ketrics.http.post(
    "https://api.example.com/orders",
    {
      customerId,
      items,
      createdAt: new Date().toISOString(),
    },
    {
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
    },
  );

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

  const response = await ketrics.http.put(
    `https://api.example.com/users/${userId}`,
    {
      name,
      email,
    },
    {
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

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
response.status; // number — HTTP status code (200, 201, 404, etc.)
response.statusText; // string — HTTP status text ("OK", "Created", etc.)
response.data; // unknown — Parsed response body (JSON by default)
response.headers; // Record<string, string> — Response headers
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
- **CONTABILIZADA**: Final success state. A user granted `revert` can return it to `APROBADA`.
- **ERROR**: Background job failed. A user granted `revert` can return it to `APROBADA` to retry.

Note the capabilities this workflow needs: `approve` for the forward transition and `revert` for the backward one. They are separate because the handlers must distinguish them — a worked example of "a handler must behave differently → new capability" (see [CONFIG_AND_DEPLOY.md](CONFIG_AND_DEPLOY.md)). Both belong in `PERMISSIONS` and in `actions`; which roles get them is then a config decision.

### Validating transitions in handlers

```typescript
const approveEntity = async (payload: { id: string }) => {
  requirePermission("approve");
  const entity = (await docdb.get(pk, sk)) as unknown as MyEntity;
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
  requirePermission("revert");
  const entity = (await docdb.get(pk, sk)) as unknown as MyEntity;
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
```

Use the appropriate quoting syntax for the database type:

```typescript
// PostgreSQL — double quotes
const tableName = `"prod"."${empresa}_softland_saldos_docs"`;

// MSSQL (SQL Server) — square brackets
const tableName = `[${empresa}].softland.cwmovim`;
```

Never interpolate user input into SQL without validation. Use parameterized queries (`$1`/`$2` for PostgreSQL, `?` for MSSQL) for values, and regex validation for identifiers.

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
