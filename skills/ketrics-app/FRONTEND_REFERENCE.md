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

    const response = await fetch(`${runtimeApiUrl}/tenants/${tenantId}/applications/${applicationId}/functions/${fnName}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${accessToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ payload: payload ?? null }),
    });

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
const { result } = (await apiClient.run("getConnections")) as { result: { connections: Connection[] } };

// Call with payload
const { result } = (await apiClient.run("createItem", {
  name: "My Item",
  description: "Item description",
})) as { result: { item: Item } };

// Common pattern: wrap in a helper for cleaner component code
const loadItems = async () => {
  const { result } = (await apiClient.run("listItems")) as { result: { items: Item[] } };
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
    const item = itemsStore.find((i) => i.id === id);
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
    const idx = itemsStore.findIndex((i) => i.id === id);
    if (idx === -1) throw new Error("Not found");
    itemsStore[idx] = { ...itemsStore[idx], name, description: description || "", updatedAt: new Date().toISOString() };
    return { item: itemsStore[idx] };
  },

  deleteItem: (payload: unknown) => {
    const { id } = payload as { id: string };
    itemsStore = itemsStore.filter((i) => i.id !== id);
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
  base: "./", // Important for iframe embedding
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

| Icon                        | Import         | Use case                              |
| --------------------------- | -------------- | ------------------------------------- |
| `Plus`                      | `lucide-react` | Create / Add actions                  |
| `Trash2`                    | `lucide-react` | Delete actions                        |
| `Edit`                      | `lucide-react` | Edit actions (alias: `Pencil`)        |
| `Search`                    | `lucide-react` | Search inputs                         |
| `Loader2`                   | `lucide-react` | Loading spinners (add `animate-spin`) |
| `CheckCircle`               | `lucide-react` | Success / active status               |
| `XCircle`                   | `lucide-react` | Error / inactive status               |
| `AlertTriangle`             | `lucide-react` | Warnings                              |
| `Download`                  | `lucide-react` | Export / download actions             |
| `Upload`                    | `lucide-react` | Import / upload actions               |
| `ChevronDown` / `ChevronUp` | `lucide-react` | Expand / collapse toggles             |
| `ArrowLeft`                 | `lucide-react` | Back navigation                       |
| `Settings`                  | `lucide-react` | Configuration views                   |
| `Eye` / `EyeOff`            | `lucide-react` | Visibility toggles                    |
| `Filter`                    | `lucide-react` | Filter controls                       |

## Common frontend patterns

### Loading and error states

```typescript
const [loading, setLoading] = useState(false);
const [error, setError] = useState<string | null>(null);

const fetchData = async () => {
  setLoading(true);
  setError(null);
  try {
    const { result } = (await apiClient.run("listItems")) as { result: { items: Item[] } };
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
    const { result } = (await apiClient.run("exportData", {
      /* payload */
    })) as {
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

## Application header layout

The app header uses a fixed 56px bar with title on the left and configuration controls aligned to the right.

### Structure

- `.app-header`: flex container with `space-between`, white background, bottom border
- `.app-header-left`: app title (`<h1>`)
- `.app-header-right`: empresa selector, settings button (role-gated)

### Example

```tsx
<header className="app-header">
  <div className="app-header-left">
    <h1>App Title</h1>
  </div>
  <div className="app-header-right">
    <EmpresaSelector ... />
    {permissions.isApprover && (
      <button className="btn btn-secondary btn-sm" title="Configuración">
        <Settings size={18} />
      </button>
    )}
  </div>
</header>
```

### CSS

```css
.app-header {
  background: #fff;
  border-bottom: 1px solid #e0e0e0;
  padding: 0 24px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 56px;
}
.app-header-left {
  display: flex;
  align-items: center;
  gap: 16px;
}
.app-header-right {
  display: flex;
  align-items: center;
  gap: 12px;
}
.app-header h1 {
  font-size: 18px;
  font-weight: 600;
  color: #1a1a1a;
}
```

## Toolbar pattern

Toolbars appear above tables or lists with left/right groups for actions and controls.

### Structure

- `.toolbar`: flex container with `space-between`
- `.toolbar-left`: search inputs, filter toggles, bulk actions
- `.toolbar-right`: primary action buttons (Create, Export, Import)

### Example

```tsx
<div className="toolbar">
  <div className="toolbar-left">
    <div style={{ position: "relative" }}>
      <Search size={16} style={{ position: "absolute", left: 10, top: "50%", transform: "translateY(-50%)", color: "#999" }} />
      <input className="form-input" placeholder="Buscar..." style={{ paddingLeft: 32 }} />
    </div>
    <button className="btn btn-secondary btn-sm">
      <Filter size={14} /> Filtros
    </button>
  </div>
  <div className="toolbar-right">
    {permissions.isEditor && (
      <button className="btn btn-primary btn-sm">
        <Plus size={14} /> Nueva solicitud
      </button>
    )}
    <button className="btn btn-secondary btn-sm">
      <Download size={14} /> Exportar
    </button>
  </div>
</div>
```

### CSS

```css
.toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 16px;
}
.toolbar-left {
  display: flex;
  align-items: center;
  gap: 8px;
}
.toolbar-right {
  display: flex;
  align-items: center;
  gap: 8px;
}
```

## Styling conventions

Ketrics apps use **vanilla CSS** in a single `App.css` file (no Tailwind). All styles follow these conventions.

### Color palette

| Token              | Value                 | Usage                                          |
| ------------------ | --------------------- | ---------------------------------------------- |
| Primary blue       | `#2563eb`             | Buttons, active tabs, links, focus rings       |
| Primary hover      | `#1d4ed8`             | Hover states for primary elements              |
| Success green      | `#16a34a`             | Approve buttons, success badges, boolean "yes" |
| Danger red         | `#dc2626`             | Reject buttons, error messages, danger badges  |
| Text primary       | `#1a1a1a` / `#333`    | Headings, body text                            |
| Text secondary     | `#555` / `#666`       | Labels, descriptions                           |
| Text muted         | `#888` / `#999`       | Metadata, placeholders, empty states           |
| Border             | `#e0e0e0` / `#d1d5db` | Cards, tables, inputs                          |
| Background page    | `#f5f5f5`             | Page background                                |
| Background card    | `#fff`                | Cards, modals, header                          |
| Background alt row | `#fafafa`             | Alternating table rows                         |
| Row hover          | `#eef2ff`             | Table row hover                                |

### Typography

- **Font stack**: `-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif`
- **Base size**: 14px on body
- **Title** (`h1`): 18px, weight 600
- **Section title**: 16px, weight 600
- **Body / table cells**: 13px
- **Labels**: 12px, weight 600, uppercase, `letter-spacing: 0.3px`
- **Small text** (badges, metadata): 11-12px

### Buttons

Base class `.btn` with variant modifiers:

```css
.btn          /* inline-flex, gap 6px, padding 8px 16px, border-radius 6px, font 13px 500 */
.btn-primary  /* bg #2563eb, white text */
.btn-secondary /* bg white, gray border #d1d5db */
.btn-success  /* bg #16a34a, white text */
.btn-danger   /* bg #dc2626, white text */
.btn-sm       /* padding 5px 10px, font 12px */
.btn-link     /* no bg/border, blue underlined text */
.btn:disabled /* opacity 0.5, cursor not-allowed */
```

### Badges

Base class `.badge` (inline-block, pill shape, 11px uppercase):

| Class              | Background | Text      | Use                 |
| ------------------ | ---------- | --------- | ------------------- |
| `.badge-pendiente` | `#fef3c7`  | `#92400e` | Pending status      |
| `.badge-aprobada`  | `#d1fae5`  | `#065f46` | Approved status     |
| `.badge-rechazada` | `#fee2e2`  | `#991b1b` | Rejected status     |
| `.badge-softland`  | `#2563eb`  | white     | Softland field tag  |
| `.badge-banking`   | `#dc2626`  | white     | Banking field tag   |
| `.badge-attrs`     | `#6b7280`  | white     | Attribute field tag |

### Tables

```css
.table-container  /* white bg, border, rounded corners, overflow hidden */
.data-table       /* full width, collapse borders */
.data-table th    /* bg #f8f9fa, 12px uppercase, gray text */
.data-table td    /* 13px, light bottom border */
.data-table tbody tr:nth-child(even) /* bg #fafafa */
.data-table tbody tr:hover           /* bg #eef2ff, pointer cursor */
```

### Modals

```css
.modal-overlay  /* fixed fullscreen, semi-transparent black bg, z-index 1000 */
.modal-card     /* white, rounded 12px, max-width 900px, flex column, shadow */
.modal-card-sm  /* max-width 640px variant */
.modal-header   /* flex space-between, bottom border */
.modal-body     /* padding 20px, overflow-y auto, flex 1 */
.modal-footer   /* flex end, gap 8px, top border */
```

### Tabs

Two tab styles are used:

1. **Navigation tabs** (`.tabs` + `.tab`): underline style, used for main app navigation
2. **Filter tabs** (`.cr-filter-tabs` + `.cr-filter-tab`): pill style with rounded borders, used for status filtering

Active state for both uses primary blue (`#2563eb`).

### Forms

```css
.form-group    /* flex column, gap 4px */
.form-label    /* 12px, weight 600, uppercase, color #555 */
.form-input    /* padding 7px 10px, border #d1d5db, rounded 6px, 13px */
.form-select   /* same as form-input */
.form-textarea /* same + resize vertical, min-height 80px, full width */
/* Focus state: border #2563eb, blue box-shadow ring */
```

### Grid layouts

Responsive grids use CSS Grid with `auto-fill`:

```css
/* Filters grid */
grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));

/* Attributes grid */
grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
```

### Loading, empty, and error states

```css
.loading-container  /* flex center, padding 40px, gray text */
.spinner           /* 20px circle, blue top border, CSS spin animation */
.empty-state       /* text-align center, padding 40px, gray #999 text */
.error-message     /* red text #dc2626, pink bg #fef2f2, red border #fecaca, rounded */
```

### Pagination

```css
.pagination      /* flex space-between, top border, bg #fafafa */
.pagination-btn  /* bordered button, 13px, min-width 32px */
.pagination-btn.active /* blue bg, white text */
```
