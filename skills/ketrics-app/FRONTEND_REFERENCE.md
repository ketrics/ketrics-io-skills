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
    canWrite: true,
    canApprove: true,
    canExport: true,
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
    a.target = "_blank";
    a.rel = "noopener";
    a.click();
  } catch (err) {
    setError(err instanceof Error ? err.message : "Export failed");
  }
};
```

**`target="_blank"` is required, and the backend must set `Content-Disposition: attachment` on the presigned URL.** Two CSP-related gotchas combine here:

1. The frontend runs inside an iframe with `frame-src https://cdn.ketrics.io`. A bare `a.click()` (no target) navigates the current frame — but the URL points at S3, which isn't on the CSP allowlist, so the navigation is blocked. `target="_blank"` makes the click open a new top-level browsing context that isn't bound by the iframe's CSP.
2. The `download` attribute is **ignored for cross-origin URLs**. Setting `a.download = filename` does nothing for S3 URLs — the browser only honors it for same-origin links. The only way to force a save is to have the *response* carry `Content-Disposition: attachment`, which the backend sets via `responseContentDisposition` on `generateDownloadUrl`. With that header in place, the new tab triggered by `target="_blank"` turns into a download instead of actually opening a tab — the user just sees their file save.

Never use `window.open(url, "_blank")` for Volume files — pop-up blockers can rewrite it to in-frame navigation, which then trips the CSP. The anchor pattern above is more reliable.

If you ever see `Framing 'https://ketrics-volumes-...amazonaws.com/' violates the following Content Security Policy directive: "frame-src https://cdn.ketrics.io"` in the browser console, one of the two pieces is missing.

### Permission-based UI (multiple capabilities)

The `getPermissions` handler reports what the current user **can do**, not which roles they hold.
Mirror the backend's capabilities (`actions` in `ketrics.config.json`) as `can*` booleans — a user
may hold several roles, and the UI only ever cares about the union of their capabilities.

```typescript
interface Permissions {
  canWrite: boolean;
  canApprove: boolean;
  canExport: boolean;
  userId: string;
  userName: string;
}

const [permissions, setPermissions] = useState<Permissions>({
  canWrite: false, canApprove: false, canExport: false, userId: "", userName: "",
});

useEffect(() => {
  apiClient.run("getPermissions")
    .then(({ result }: any) => setPermissions(result))
    .catch(() => {});
}, []);

// In JSX — different capabilities control different actions
{permissions.canWrite && <button onClick={handleCreate}>Create</button>}
{permissions.canApprove && <button onClick={handleApprove}>Approve</button>}
{permissions.canExport && <button onClick={handleExport}>Export</button>}
```

The matching backend handler lives in `permissions.ts` and reads the granted capabilities straight
from the requestor:

```typescript
// permissions.ts
const has = (permission: Permission): boolean => {
  const granted = ketrics.requestor.applicationPermissions;
  return granted.includes("*") || granted.includes(permission);
};

export const getPermissions = async () => ({
  canWrite: has("write"),
  canApprove: has("approve"),
  canExport: has("export"),
  userId: ketrics.requestor.userId,
  userName: ketrics.requestor.name,
});
```

Hiding a button is a convenience, never a control: every handler still calls `requirePermission`.

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
- `.app-header-right`: empresa selector, settings button (role-gated), user name, and — always last,
  for every user — the ⓘ help button that opens the user guide (see
  [In-app user guide](#in-app-user-guide-guía-de-uso--required))

### Example

```tsx
<header className="app-header">
  <div className="app-header-left">
    <h1>App Title</h1>
  </div>
  <div className="app-header-right">
    <EmpresaSelector ... />
    {permissions.canApprove && (
      <button className="btn btn-secondary btn-sm" title="Configuración" aria-label="Configuración">
        <Settings size={18} aria-hidden="true" />
      </button>
    )}
    {/* Always rendered, never role-gated: the user guide. */}
    <button type="button" className="btn-icon boton-ayuda" onClick={() => setAyuda(true)} aria-label="Ayuda" title="Ayuda">
      <CircleHelp size={20} aria-hidden="true" />
    </button>
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

## In-app user guide ("Guía de uso") — required

**Every app ships its user documentation inside the app**, one click away: an ⓘ help button at the
far right of the header opens a modal with the guide. Users of tenant apps rarely have other
documentation, and the rules they can't guess (what enters a file, why a notice appears, who can
do what) belong where they work. Build it with the first version of the screen, not later.

### Rules

- **The button is always there, for everyone.** Last element of `.app-header-right` (to the right of
  the settings cog and the user name), rendered unconditionally: never behind a permission check, and
  also while loading, on errors, or when the user has no read access — the guide explains those cases.
- **Icon-only button:** `CircleHelp` from lucide-react with `aria-hidden`, plus `aria-label="Ayuda"`
  and `title="Ayuda"`.
- **It opens a modal** built on the app's accessible `<Modal>` (`role="dialog"`, `aria-modal`, focus
  trap, Escape closes, focus returns to the ⓘ button). Large size, the body scrolls.
- **Static content, no backend call**, written in the users' language (Spanish for most tenants), in
  plain words — no handler names, no SQL, no jargon.
- **Table of contents** at the top: links that move focus to each section's heading **without
  changing the URL** (`preventDefault` + `scrollIntoView` + `focus`). The app runs in an iframe; a
  hash change there is pointless at best.
- **Content lives in `frontend/src/utils/guia.ts`** (sections, lists, FAQ) and the markup in
  `frontend/src/components/GuiaUsuarioDialog.tsx`, so the content can be tested without a DOM.
- **Anything the code already knows is derived or tested against the code**, never retyped freely:
  profiles against `roles` in `ketrics.config.json`, file columns against the backend's layout
  module, etc. — see [Keeping it true](#keeping-the-guide-true).

### Sections

Pick what applies; most apps need the first four and the last two.

| Section | What it says |
| --- | --- |
| Qué hace la aplicación | Purpose in two sentences; where the data comes from; what it does *not* do |
| El flujo | The main task step by step, in the order of the screen's buttons |
| Estados | Every status badge the user can see, and what moves it |
| Descargar / eliminar | Downloads (on click), what delete removes, what can't be undone |
| Formatos | Files the app produces: columns, number/date formats, file names |
| Perfiles | Each role in `ketrics.config.json`: name and what it can do, in plain words |
| Auditoría | What is recorded (and what never is), who can see it |
| Preguntas frecuentes | The error messages and surprises users actually hit, with what to do |

### Header button

```tsx
import { CircleHelp } from "lucide-react";
import { GuiaUsuarioDialog } from "./components/GuiaUsuarioDialog";

const [ayuda, setAyuda] = useState(false);

<div className="app-header-right">
  {/* …selector, settings cog (role-gated), user name… */}
  {/* Para todos, siempre (también cargando o con error): la guía explica esos casos. */}
  <button type="button" className="btn-icon boton-ayuda" onClick={() => setAyuda(true)} aria-label="Ayuda" title="Ayuda">
    <CircleHelp size={20} aria-hidden="true" />
  </button>
</div>

{ayuda && <GuiaUsuarioDialog onClose={() => setAyuda(false)} />}
```

### Content module — `frontend/src/utils/guia.ts`

```typescript
/**
 * Contenido de la guía de uso. Regla: cuando cambia el flujo, los formatos,
 * los perfiles o la auditoría, la guía cambia en el mismo PR.
 */
export interface SeccionGuia {
  id: string;
  titulo: string;
}

/** Las secciones, en el orden del índice. El id es el ancla (con prefijo "guia-"). */
export const SECCIONES_GUIA: readonly SeccionGuia[] = [
  { id: "que-hace", titulo: "Qué hace la aplicación" },
  { id: "flujo", titulo: "Cómo se usa" },
  { id: "estados", titulo: "Estados" },
  { id: "perfiles", titulo: "Perfiles" },
  { id: "preguntas", titulo: "Preguntas frecuentes" },
];

export const anclaGuia = (id: string): string => `guia-${id}`;

/** Un perfil por rol de ketrics.config.json (misma lista, mismo orden: una prueba lo revisa). */
export const PERFILES_GUIA = [
  { codigo: "viewer", nombre: "Consulta", puede: "Ve los registros y su estado." },
  { codigo: "editor", nombre: "Editor", puede: "Todo lo anterior, y además crea y modifica registros." },
] as const;

export const PREGUNTAS_GUIA = [
  { pregunta: "No veo el botón Nuevo.", respuesta: "Tu perfil es de consulta. Pide a un administrador el perfil Editor." },
] as const;
```

### Dialog — `frontend/src/components/GuiaUsuarioDialog.tsx`

```tsx
import type { MouseEvent, ReactNode } from "react";
import { BookOpen } from "lucide-react";
import { anclaGuia, PERFILES_GUIA, PREGUNTAS_GUIA, SECCIONES_GUIA } from "../utils/guia";
import { Modal } from "./Modal";

const titulo = (id: string) => SECCIONES_GUIA.find((s) => s.id === id)!.titulo;

/** Lleva el foco al título de la sección sin tocar la URL (la app vive en un iframe). */
const irA = (e: MouseEvent<HTMLAnchorElement>, id: string) => {
  e.preventDefault();
  const h = document.getElementById(anclaGuia(id));
  h?.scrollIntoView({ block: "start" });
  h?.focus();
};

function Seccion({ id, children }: { id: string; children: ReactNode }) {
  return (
    <section className="guia-seccion" aria-labelledby={anclaGuia(id)}>
      <h3 id={anclaGuia(id)} tabIndex={-1}>{titulo(id)}</h3>
      {children}
    </section>
  );
}

export function GuiaUsuarioDialog({ onClose }: { onClose: () => void }) {
  return (
    <Modal titulo="Guía de uso" icono={<BookOpen size={18} aria-hidden="true" />} tamano="modal-card-lg" onClose={onClose}>
      <div className="modal-body guia">
        <nav aria-label="Contenido de la guía">
          <ol className="guia-indice">
            {SECCIONES_GUIA.map((s) => (
              <li key={s.id}>
                <a href={`#${anclaGuia(s.id)}`} onClick={(e) => irA(e, s.id)}>{s.titulo}</a>
              </li>
            ))}
          </ol>
        </nav>

        <Seccion id="que-hace">
          <p>…</p>
        </Seccion>

        <Seccion id="perfiles">
          <dl className="guia-lista">
            {PERFILES_GUIA.map((p) => (
              <div key={p.codigo}>
                <dt>{p.nombre}</dt>
                <dd>{p.puede}</dd>
              </div>
            ))}
          </dl>
        </Seccion>

        <Seccion id="preguntas">
          <dl className="guia-lista">
            {PREGUNTAS_GUIA.map((q) => (
              <div key={q.pregunta}>
                <dt>{q.pregunta}</dt>
                <dd>{q.respuesta}</dd>
              </div>
            ))}
          </dl>
        </Seccion>
      </div>
      <div className="modal-footer">
        <button type="button" className="btn btn-secondary" onClick={onClose}>Cerrar</button>
      </div>
    </Modal>
  );
}
```

Tables inside the guide (e.g. a file's columns) get a `<caption>` and `<th scope="col">`.

### CSS

```css
.boton-ayuda { color: #1d4ed8; }

.guia-indice {
  columns: 2;
  margin: 0 0 20px;
  padding: 12px 12px 12px 32px;
  background: #f9fafb;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
}
.guia-indice a { color: #1d4ed8; }            /* AA on #f9fafb */
.guia-seccion { margin-bottom: 20px; }
.guia-seccion h3 { margin: 0 0 8px; font-size: 15px; color: #1a1a1a; scroll-margin-top: 8px; }
.guia-seccion h3:focus-visible,
.guia-indice a:focus-visible { outline: 2px solid #2563eb; outline-offset: 2px; }
.guia-lista dt { font-weight: 600; }
.guia-lista dd { margin: 0 0 10px; }

@media (max-width: 600px) {
  .guia-indice { columns: 1; }
}
```

**The modal must fit a phone screen.** On iPhone Safari `100vh` is taller than the visible area
(it ignores the browser bars), so a `max-height: calc(100vh - 32px)` modal hides its own title and
footer. Use `dvh` with a `vh` fallback, and keep header and footer from shrinking — this applies to
every modal, and the guide is the first one long enough to show it:

```css
.modal-card {
  max-height: calc(100vh - 32px);
  max-height: calc(100dvh - 32px);   /* the visible viewport */
  display: flex;
  flex-direction: column;
}
.modal-header, .modal-footer { flex-shrink: 0; }
.modal-body { flex: 1; min-height: 0; overflow-y: auto; }
```

### Keeping the guide true

A guide that describes last month's app is worse than none. Three things keep it honest:

1. **A rule in the app's `CLAUDE.md`:** *when the flow, a file format, the roles or what is audited
   changes, the guide changes in the same PR* — and a step "update the guide if what the user sees
   changes" in its "Adding a handler" checklist.
2. **Tests that tie the guide to the code** (`frontend/test/guia.test.ts`, run with `node:test`):

   ```typescript
   import { test } from "node:test";
   import assert from "node:assert/strict";
   import { readFileSync } from "node:fs";
   import { join } from "node:path";
   import { createElement } from "react";
   import { renderToStaticMarkup } from "react-dom/server";
   import { GuiaUsuarioDialog } from "../src/components/GuiaUsuarioDialog";
   import { anclaGuia, PERFILES_GUIA, SECCIONES_GUIA } from "../src/utils/guia";

   test("los perfiles son los roles de ketrics.config.json", () => {
     const config = JSON.parse(readFileSync(join(process.cwd(), "..", "ketrics.config.json"), "utf8"));
     assert.deepEqual(
       PERFILES_GUIA.map((p) => [p.codigo, p.nombre]),
       config.roles.map((r: { code: string; name: string }) => [r.code, r.name]),
     );
   });

   test("cada sección se renderiza con su título enfocable, y el índice enlaza a todas", () => {
     const html = renderToStaticMarkup(createElement(GuiaUsuarioDialog, { onClose: () => {} }));
     for (const s of SECCIONES_GUIA) assert.ok(html.includes(`<h3 id="${anclaGuia(s.id)}" tabindex="-1">`), s.id);
     const enlaces = [...html.matchAll(/<a href="#([\w-]+)">/g)].map((m) => m[1]);
     assert.deepEqual(enlaces, SECCIONES_GUIA.map((s) => anclaGuia(s.id)));
   });

   test("el botón ⓘ está en el header sin condición", () => {
     const app = readFileSync(join(process.cwd(), "src", "App.tsx"), "utf8");
     const header = /<div className="app-header-right">([\s\S]*?)<\/header>/.exec(app)![1].split("\n");
     const boton = header.findIndex((l) => l.includes("boton-ayuda"));
     assert.match(header[boton], /^\s*<button\b/, "no va dentro de un && ni de un ternario");
     assert.match(app, /\{ayuda && <GuiaUsuarioDialog/);
   });
   ```

   When the app produces a file, add one more: the guide's column list equals the backend's layout
   (import the backend module directly if it's pure — esbuild bundles it into the test).
3. **Keep the dialog free of `services/api` imports** (it's static): that is what lets
   `react-dom/server` render it in a plain Node test.

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
    {permissions.canWrite && (
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
.modal-card     /* white, rounded 12px, max-width 900px, flex column, shadow,
                   max-height calc(100dvh - 32px) with a 100vh fallback (iPhone: 100vh hides header/footer) */
.modal-card-sm  /* max-width 640px variant */
.modal-header   /* flex space-between, bottom border, flex-shrink 0 */
.modal-body     /* padding 20px, overflow-y auto, flex 1, min-height 0 */
.modal-footer   /* flex end, gap 8px, top border, flex-shrink 0 */
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
