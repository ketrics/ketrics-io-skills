# Frontend Reference

Patterns for the React frontend of a Ketrics tenant application. The frontend is built with Vite + React 18 + TypeScript and embedded as an iframe in the Ketrics platform.

## Service layer

The service layer provides a class-based `APIClient` with a `run(fnName, payload)` method. In dev mode, a `MockAPIClient` is used instead. In production, calls go to the Ketrics Runtime API with JWT auth.

### `frontend/src/services/index.ts`

```typescript
import { createAuthManager } from "@ketrics/sdk-frontend";

interface APIClientInterface {
  run(fnName: string, payload?: unknown): Promise<unknown>;
}

class APIClient implements APIClientInterface {
  private auth;

  constructor(authManager: ReturnType<typeof createAuthManager>) {
    this.auth = authManager;
  }

  async run(fnName: string, payload?: unknown) {
    const runtimeApiUrl = this.auth.getRuntimeApiUrl();
    const tenantId = this.auth.getTenantId();
    const applicationId = this.auth.getApplicationId();
    const accessToken = this.auth.getAccessToken();

    if (!runtimeApiUrl || !tenantId || !accessToken) {
      throw new Error("Missing authentication context");
    }

    const response = await fetch(
      `${runtimeApiUrl}/tenants/${tenantId}/applications/${applicationId}/functions/${fnName}`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${accessToken}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ payload: payload ?? null }),
      }
    );

    if (!response.ok) {
      const errorBody = await response.json().catch(() => null);
      const message = errorBody?.error?.message || `API request failed with status ${response.status}`;
      throw new Error(message);
    }

    return response.json();
  }
}

async function createClient(): Promise<APIClientInterface> {
  if (import.meta.env.DEV) {
    const { createMockClient } = await import("../mocks/mock-client");
    return createMockClient();
  }

  const auth = createAuthManager();
  auth.initAutoRefresh({ refreshBuffer: 60, onTokenUpdated: () => {} });
  return new APIClient(auth);
}

const apiClient = await createClient();
export { apiClient };
```

### `frontend/src/mocks/mock-client.ts`

Separate file for the mock client — tree-shaken out of production builds by Vite:

```typescript
import { handlers } from "./handlers";

class MockAPIClient {
  async run(fnName: string, payload?: unknown) {
    const handler = handlers[fnName];
    if (!handler) {
      throw new Error(`[Mock] No handler for "${fnName}". Add it to src/mocks/handlers.ts`);
    }
    await new Promise((r) => setTimeout(r, 200)); // Simulate network latency
    try {
      const result = await handler(payload);
      console.log(`[Mock] ${fnName}`, { payload, result });
      return { success: true, result };
    } catch (err) {
      const message = err instanceof Error ? err.message : String(err);
      console.error(`[Mock] ${fnName} error:`, message);
      throw new Error(message);
    }
  }
}

export function createMockClient() {
  console.log("%c[Ketrics] Running with mock backend", "color: #f59e0b; font-weight: bold;");
  return new MockAPIClient();
}
```

### Key points

- `import.meta.env.DEV` — Vite's built-in env flag for dev mode detection
- `createAuthManager()` — Returns an auth manager that handles JWT tokens in the iframe context
- `auth.initAutoRefresh()` — Keeps the token fresh; `refreshBuffer: 60` means refresh 60s before expiry
- Dynamic import of mock client keeps mock code out of production builds
- Handler names must match exactly between `apiClient.run("handlerName", ...)` and `backend/src/index.ts` exports

## Calling backend handlers

```typescript
import { apiClient } from "../services";

// Simple call (no payload)
const { result } = await apiClient.run("getConnections") as { result: { connections: Connection[] } };

// Call with payload
const { result } = await apiClient.run("createItem", {
  name: "My Item",
  description: "Item description",
}) as { result: { item: Item } };

// Common pattern: wrap in a helper for cleaner component code
const loadItems = async () => {
  const { result } = await apiClient.run("listItems") as { result: { items: Item[] } };
  return result.items;
};
```

## Mock handlers

Create mock handlers for local development. These let you run the frontend without a backend.

### `frontend/src/mocks/handlers.ts`

```typescript
type MockHandler = (payload?: unknown) => unknown | Promise<unknown>;

// In-memory stores
let itemsStore: Item[] = [
  { id: "1", name: "Sample Item", createdBy: "mock-user", createdAt: "2024-01-01T00:00:00Z", updatedAt: "2024-01-01T00:00:00Z" },
];

const handlers: Record<string, MockHandler> = {
  listItems: () => ({
    items: itemsStore,
  }),

  getItem: (payload: unknown) => {
    const { id } = payload as { id: string };
    const item = itemsStore.find(i => i.id === id);
    if (!item) throw new Error("Not found");
    return { item };
  },

  createItem: (payload: unknown) => {
    const { name, description } = payload as { name: string; description?: string };
    const now = new Date().toISOString();
    const newItem: Item = {
      id: crypto.randomUUID(),
      name,
      description: description || "",
      createdBy: "mock-user",
      createdAt: now,
      updatedAt: now,
    };
    itemsStore.push(newItem);
    return { item: newItem };
  },

  updateItem: (payload: unknown) => {
    const { id, name, description } = payload as { id: string; name: string; description?: string };
    const idx = itemsStore.findIndex(i => i.id === id);
    if (idx === -1) throw new Error("Not found");
    itemsStore[idx] = { ...itemsStore[idx], name, description: description || "", updatedAt: new Date().toISOString() };
    return { item: itemsStore[idx] };
  },

  deleteItem: (payload: unknown) => {
    const { id } = payload as { id: string };
    itemsStore = itemsStore.filter(i => i.id !== id);
    return { success: true };
  },

  getPermissions: () => ({
    isEditor: true,
    userId: "mock-user",
    userName: "Mock User",
  }),
};

export { handlers };
```

### Mock handler rules

- Handler names MUST match the backend exports exactly
- Use in-memory arrays/objects for state (persists only during the dev session)
- Cast `payload` to the expected type (mock handlers receive `unknown`)
- Simulate realistic response shapes matching what the backend returns

## TypeScript types

Define shared interfaces in `frontend/src/types.ts`. These mirror the backend types and define the shape of API responses.

```typescript
// Common pattern: one full interface + one summary for list views
export interface Item {
  id: string;
  name: string;
  description: string;
  createdBy: string;
  createdAt: string;
  updatedAt: string;
}

export interface ItemSummary {
  id: string;
  name: string;
  description: string;
  createdAt: string;
  updatedAt: string;
}
```

## Vite configuration

### `frontend/vite.config.ts`

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  base: "./",  // Important for iframe embedding
});
```

### `frontend/index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My Ketrics App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### `frontend/src/main.tsx`

```typescript
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

## Frontend package.json

```json
{
  "name": "my-ketrics-app",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@ketrics/sdk-frontend": "^0.3.0",
    "lucide-react": "^0.400.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "@vitejs/plugin-react": "^4.2.0",
    "typescript": "^5.3.3",
    "vite": "^5.0.0"
  }
}
```

## Icons with Lucide React

Use [lucide-react](https://lucide.dev) for icons. It provides a comprehensive set of clean, consistent SVG icons as React components.

### Installation

```bash
npm install lucide-react
```

### Basic usage

Import icons individually — each icon is tree-shakeable:

```typescript
import { Plus, Trash2, Edit, Search, ChevronDown, Loader2 } from "lucide-react";

// Use as JSX components with optional size and color props
<Plus size={16} />
<Trash2 size={18} color="red" />
<Search size={20} strokeWidth={1.5} />
```

### Common icon patterns

```typescript
// Button with icon
<button onClick={handleCreate}>
  <Plus size={16} /> Create Item
</button>

// Icon-only button (e.g., delete action in a table row)
<button onClick={() => handleDelete(item.id)} title="Delete">
  <Trash2 size={16} />
</button>

// Loading spinner
{loading && <Loader2 size={20} className="animate-spin" />}

// Status indicators
{item.status === "active" ? <CheckCircle size={16} color="green" /> : <XCircle size={16} color="red" />}

// Collapsible section toggle
<button onClick={() => setExpanded(!expanded)}>
  {expanded ? <ChevronUp size={16} /> : <ChevronDown size={16} />}
  Details
</button>
```

### Frequently used icons

| Icon | Import | Use case |
|------|--------|----------|
| `Plus` | `lucide-react` | Create / Add actions |
| `Trash2` | `lucide-react` | Delete actions |
| `Edit` | `lucide-react` | Edit actions (alias: `Pencil`) |
| `Search` | `lucide-react` | Search inputs |
| `Loader2` | `lucide-react` | Loading spinners (add `animate-spin`) |
| `CheckCircle` | `lucide-react` | Success / active status |
| `XCircle` | `lucide-react` | Error / inactive status |
| `AlertTriangle` | `lucide-react` | Warnings |
| `Download` | `lucide-react` | Export / download actions |
| `Upload` | `lucide-react` | Import / upload actions |
| `ChevronDown` / `ChevronUp` | `lucide-react` | Expand / collapse toggles |
| `ArrowLeft` | `lucide-react` | Back navigation |
| `Settings` | `lucide-react` | Configuration views |
| `Eye` / `EyeOff` | `lucide-react` | Visibility toggles |
| `Filter` | `lucide-react` | Filter controls |

## Common frontend patterns

### Loading and error states

```typescript
const [loading, setLoading] = useState(false);
const [error, setError] = useState<string | null>(null);

const fetchData = async () => {
  setLoading(true);
  setError(null);
  try {
    const { result } = await apiClient.run("listItems") as { result: { items: Item[] } };
    setItems(result.items);
  } catch (err) {
    setError(err instanceof Error ? err.message : "An error occurred");
  } finally {
    setLoading(false);
  }
};
```

### File download from presigned URL

```typescript
const handleExport = async () => {
  try {
    const { result } = await apiClient.run("exportData", { /* payload */ }) as {
      result: { url: string; filename: string };
    };
    const a = document.createElement("a");
    a.href = result.url;
    a.download = result.filename;
    a.click();
  } catch (err) {
    setError(err instanceof Error ? err.message : "Export failed");
  }
};
```

### Permission-based UI (multiple roles)

```typescript
interface Permissions {
  isEditor: boolean;
  isApprover: boolean;
  isAdmin: boolean;
  userId: string;
  userName: string;
}

const [permissions, setPermissions] = useState<Permissions>({
  isEditor: false, isApprover: false, isAdmin: false, userId: "", userName: "",
});

useEffect(() => {
  apiClient.run("getPermissions")
    .then(({ result }: any) => setPermissions(result))
    .catch(() => {});
}, []);

// In JSX — different roles control different actions
{permissions.isEditor && <button onClick={handleCreate}>Create</button>}
{permissions.isApprover && <button onClick={handleApprove}>Approve</button>}
{permissions.isAdmin && <button onClick={handleRevert}>Revert</button>}
```

### Multi-view pattern

```typescript
type View = "list" | "detail" | "config";
const [view, setView] = useState<View>("list");
const [selectedId, setSelectedId] = useState<string | null>(null);

const navigateToDetail = (id: string) => {
  setSelectedId(id);
  setView("detail");
};

const navigateToList = () => {
  setSelectedId(null);
  setView("list");
  fetchData(); // Refresh data when returning to list
};

// In JSX
{view === "list" && <ListView items={items} onSelect={navigateToDetail} />}
{view === "detail" && selectedId && <DetailView id={selectedId} onBack={navigateToList} />}
{view === "config" && <ConfigView onBack={navigateToList} />}
```

### Modal dialog pattern

```typescript
const [showModal, setShowModal] = useState(false);
const [modalForm, setModalForm] = useState({ name: "", description: "" });

const openModal = () => { setModalForm({ name: "", description: "" }); setShowModal(true); };
const closeModal = () => setShowModal(false);

const handleSubmit = async () => {
  try {
    await apiClient.run("createItem", modalForm);
    closeModal();
    await fetchData(); // Refresh list
  } catch (err) {
    setError(err instanceof Error ? err.message : "Error");
  }
};

// In JSX
{showModal && (
  <div style={{
    position: "fixed", inset: 0, backgroundColor: "rgba(0,0,0,0.5)",
    display: "flex", alignItems: "center", justifyContent: "center", zIndex: 1000,
  }} onClick={closeModal}>
    <div style={{ background: "white", padding: 24, borderRadius: 8, minWidth: 400 }}
         onClick={(e) => e.stopPropagation()}>
      {/* Form fields and submit button */}
    </div>
  </div>
)}
```

### Confirmation dialog pattern

```typescript
const handleDelete = async (id: string) => {
  if (!window.confirm("Are you sure you want to delete this item? This action cannot be undone.")) return;
  try {
    await apiClient.run("deleteItem", { id });
    await fetchData();
  } catch (err) {
    setError(err instanceof Error ? err.message : "Delete failed");
  }
};
```

### Multi-select with per-item values

```typescript
// Map: key → { item, amount } for selections with per-item editable values
const [selected, setSelected] = useState<Map<string, { item: Item; amount: number }>>(new Map());

const toggleSelect = (item: Item) => {
  setSelected((prev) => {
    const next = new Map(prev);
    if (next.has(item.id)) {
      next.delete(item.id);
    } else {
      next.set(item.id, { item, amount: item.defaultAmount });
    }
    return next;
  });
};

const updateAmount = (id: string, amount: number) => {
  setSelected((prev) => {
    const next = new Map(prev);
    const entry = next.get(id);
    if (entry) next.set(id, { ...entry, amount });
    return next;
  });
};

const handleSubmitSelected = async () => {
  const items = Array.from(selected.values()).map(({ item, amount }) => ({
    id: item.id,
    amount,
  }));
  await apiClient.run("addItems", { items });
  setSelected(new Map()); // Clear selection
};
```
