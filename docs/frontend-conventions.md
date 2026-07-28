# Frontend Conventions

## Folder Layout

```
web/src/
├── app.tsx              # Root component (provider tree)
├── main.tsx             # ReactDOM entry point
├── components/          # Reusable UI components (barrel-exported via index.ts)
│   ├── button/
│   ├── card/
│   ├── data-table/
│   ├── input/
│   ├── loading/
│   ├── modal/
│   ├── multi-select/
│   ├── pagination/
│   ├── search-bar/
│   ├── select/
│   ├── skeleton/
│   ├── stat-card/
│   ├── tabs/
│   ├── page-header/
│   └── ... # (add components as your project requires)
├── contexts/            # Global state providers
│   ├── auth-context.tsx
│   └── filter-context.tsx
├── pages/               # One directory per route/page
│   ├── login/
│   ├── reset-password/
│   └── dashboard/
│       ├── analytics/
│       ├── inventory/
│       ├── orders/
│       ├── reports/
│       ├── settings/
│       ├── users/
│       └── ... # (one directory per page/route)
├── routes/              # Route definitions and guards
│   └── index.tsx
├── services/            # API client
│   └── api.ts
├── styles/              # Global CSS and custom properties
├── types/               # Shared TypeScript interfaces
└── utils/               # Formatting, export, etc.
```

## Provider Tree

Order matters. Providers that depend on others must be nested inside their dependents.

```tsx
// app.tsx
export function App() {
  return (
    <BrowserRouter>
      <AuthProvider>
        <FilterProvider>   {/* uses useAuth() — must be inside AuthProvider */}
          <AppRoutes />
        </FilterProvider>
      </AuthProvider>
    </BrowserRouter>
  );
}
```

## Route Guards

### PrivateRoute — Authentication Barrier

```tsx
function PrivateRoute({ children }: { children: React.ReactNode }) {
  const { signed, loading } = useAuth();
  if (loading) return <Loading />;
  if (!signed) return <Navigate to="/login" replace />;
  return <>{children}</>;
}
```

### PageRoute — Authorization Barrier

```tsx
function PageRoute({ pageKey, children }: { pageKey: string; children: React.ReactNode }) {
  const { user, permissions } = useAuth();

  if (user?.type === "admin") return <>{children}</>;
  if (!permissions) return <Loading />;
  if (permissions.configured === false) return <AccessDenied />;
  if (!permissions.allowedPages.length) return <>{children}</>;
  if (!permissions.allowedPages.includes(pageKey)) return <AccessDenied />;

  return <>{children}</>;
}
```

### Composition in Routes

```tsx
<Routes>
  <Route path="/login" element={signed ? <Navigate to="/dashboard" /> : <LoginPage />} />
  <Route path="/dashboard" element={<PrivateRoute><DashboardLayout /></PrivateRoute>}>
    <Route index element={<DashboardIndex />} />
    <Route path="reports" element={<PageRoute pageKey="reports"><ReportsPage /></PageRoute>} />
    <Route path="analytics" element={<PageRoute pageKey="analytics"><AnalyticsPage /></PageRoute>} />
  </Route>
  <Route path="*" element={<Navigate to="/login" replace />} />
</Routes>
```

## Dashboard Layout Composition

The dashboard uses a nested layout pattern: a shared layout component wraps all authenticated pages, providing the sidebar, header, and content outlet.

### Structure

```
<BrowserRouter>
  <AuthProvider>                    ← global auth state
    <FilterProvider>                ← global filters (depends on auth)
      <Routes>
        <Route path="/login" />     ← public
        <Route path="/dashboard" element={<PrivateRoute><DashboardLayout /></PrivateRoute>}>
          <Route index />           ← dashboard home
          <Route path="reports" element={<PageRoute pageKey="reports">...</PageRoute>} />
          <Route path="analytics" element={<PageRoute pageKey="analytics">...</PageRoute>} />
        </Route>
      </Routes>
    </FilterProvider>
  </AuthProvider>
</BrowserRouter>
```

### DashboardLayout

The layout component renders the chrome (sidebar + header) and uses `<Outlet />` for page content:

```tsx
function DashboardLayout() {
  return (
    <div className="dashboard">
      <Sidebar items={NAV_ITEMS} />
      <Header />
      <main className="dashboard__content">
        <Outlet />  {/* active route renders here */}
      </main>
    </div>
  );
}
```

### Navigation Order (NAV_ITEMS)

The sidebar navigation is driven by an ordered array, not hardcoded JSX. This keeps the nav order declarative and easy to modify:

```typescript
// routes/index.tsx
export const NAV_ITEMS = [
  { key: "reports", label: "Reports", path: "/dashboard/reports" },
  { key: "analytics", label: "Analytics", path: "/dashboard/analytics" },
  { key: "orders", label: "Orders", path: "/dashboard/orders" },
  // ... add pages as needed
];
```

**Rules:**
1. **`key` matches the `pageKey`** passed to `PageRoute` — this is how the sidebar highlights the active page and how authorization is enforced.
2. **Order matters** — the array order is the sidebar display order.
3. **Filter by permission** — the sidebar component should filter `NAV_ITEMS` by the user's `allowedPages`, same as `PageRoute` does.

### Guard Composition

| Guard | Layer | Purpose |
|---|---|---|
| `PrivateRoute` | Wraps `DashboardLayout` | Authentication — redirects to `/login` if not signed in |
| `PageRoute` | Wraps each page component | Authorization — checks `pageKey` against `allowedPages` |
| `admin` check | Inside `PageRoute` | Admin users bypass page-level checks |

This means:
- Unauthenticated users never see the dashboard chrome.
- Authenticated users see the sidebar (filtered by permission) but may get `AccessDenied` on specific pages.
- Admin users see everything.

## Auth Context

```typescript
interface AuthContextData {
  user: IUser | null;
  signed: boolean;
  loading: boolean;
  permissions: IUserPermission | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  refreshPermissions: () => Promise<void>;
}
```

**Behaviors:**
- `user`, `token`, `permissions` persisted in `localStorage`.
- On mount: restores from `localStorage` and revalidates permissions with the server.
- Listens to `window "unauthorized"` event → auto-logout (dispatched by API client on 401).
- `refreshPermissions` reloads permissions without logout.

## Filter Context

Provides global filters (regions, categories, groups) filtered by user permissions:

```typescript
interface FilterContextData {
  regions: IRegion[];
  categories: ICategory[];
  groups: string[];
  selectedRegions: string[];
  selectedCategories: string[];
  selectedGroups: string[];
  setSelectedRegions: (v: string[]) => void;
  setSelectedCategories: (v: string[]) => void;
  setSelectedGroups: (v: string[]) => void;
  loadFilters: () => Promise<void>;
}
```

Permission-based filtering:
```typescript
const regions = useMemo(() => {
  if (!permissions?.allowedRegions?.length) return allRegions;
  return allRegions.filter(b => permissions.allowedRegions.includes(b.value));
}, [allRegions, permissions]);
```

## API Client

Located in `services/api.ts`. Wraps `fetch` with automatic behaviors:

```typescript
const API_URL = import.meta.env.VITE_API_URL;

async function request<T>(endpoint: string, options: RequestInit = {}): Promise<T> {
  const token = localStorage.getItem("token");
  const response = await fetch(`${API_URL}${endpoint}`, {
    ...options,
    credentials: "include",
    headers: {
      "Content-Type": "application/json",
      ...(token ? { authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  });

  if (!response.ok) {
    if (response.status === 401) {
      window.dispatchEvent(new Event("unauthorized"));
      throw new Error("Session expired");
    }
    const error = await response.json().catch(() => ({}));
    throw new Error(error.message || `Error ${response.status}`);
  }
  return response.json();
}

export const api = {
  get:  <T>(endpoint: string) => request<T>(endpoint, { method: "GET" }),
  post: <T>(endpoint: string, body?: unknown) => request<T>(endpoint, { method: "POST", body: JSON.stringify(body) }),
  put:  <T>(endpoint: string, body?: unknown) => request<T>(endpoint, { method: "PUT", body: JSON.stringify(body) }),
  del:  <T>(endpoint: string, body?: unknown) => request<T>(endpoint, { method: "DELETE", body: body ? JSON.stringify(body) : undefined }),
};
```

**Key behaviors:**
- `VITE_API_URL` is baked into the bundle at build-time.
- Automatic `Authorization: Bearer <token>` injection from `localStorage`.
- On 401: dispatches `unauthorized` event → `AuthContext` logs out.

## Component Conventions

### Barrel Exports

All components are re-exported through `components/index.ts`:

```typescript
export { Button } from "./button";
export { Modal } from "./modal";
// ...
```

Import with the alias:
```typescript
import { Button, Modal } from "@/components";
```

### Component Structure

Each component lives in its own directory:
```
components/button/
├── index.ts          # Barrel export
├── button.tsx        # Component implementation
└── button.css        # Styles (if not using CSS modules)
```

## Theme (CSS Custom Properties)

Design system based on CSS variables for consistency and theme switching:

```css
:root {
  --color-primary: #e11d48;  /* Replace with your brand color */
  --color-bg: #09090b;
  --color-surface: #111113;
  --color-border: #27272a;
  --color-text: #fafafa;
  --color-text-secondary: #a1a1aa;
  --radius-sm: 6px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --transition: 150ms ease;
}
```

## Excel Export

Custom OOXML implementation in `utils/export.ts`. Prefer native browser APIs over heavy libraries — justify any third-party dependency. This example generates `.xlsx` files entirely in the browser using native APIs.

## Path Alias

`@/` maps to `src/`. Configured via `tsconfig.app.json` paths and `vite-tsconfig-paths`.
