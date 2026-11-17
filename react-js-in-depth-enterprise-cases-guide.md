# React.js — In-Depth Enterprise Guide with Real-World Cases

A practical React guide for experienced Java full-stack developers building enterprise frontends that integrate with Spring Boot microservices. It covers React fundamentals, modern hooks, component architecture, state management, routing, forms, API integration, security, performance, testing, accessibility, deployment, and realistic enterprise case studies.

> **Version note:** Examples use modern React function components and hooks, with React 18+ conventions. Some libraries and APIs evolve independently; check the versions supported by your application. React is a UI library, not a complete application framework.

## 1. What is React?

React is a JavaScript library for building user interfaces from reusable components. It uses a declarative model: describe the UI for a given state, and React updates the rendered interface when that state changes.

Core ideas:
- Components: reusable UI building blocks.
- Props: inputs passed from parent to child.
- State: data owned by a component or state layer.
- JSX: JavaScript syntax extension for describing UI.
- Hooks: APIs for state, effects, context, refs, and reusable logic.
- One-way data flow: data generally flows from parent to child; callbacks communicate changes upward.

### React vs Angular vs Vue (high level)
| React | Angular | Vue |
|---|---|---|
| UI library with a flexible ecosystem | Full framework with strong conventions | Progressive framework |
| Choose routing, data and form libraries | Many capabilities included | Core plus ecosystem |
| JSX commonly used | TypeScript/templates commonly used | Templates or JSX |

Choose based on team expertise, application needs, ecosystem, and maintainability.

## 2. JavaScript and TypeScript prerequisites

Before enterprise React, be comfortable with:
- `const`, `let`, arrow functions, destructuring.
- Template literals and modules (`import` / `export`).
- Arrays: `map`, `filter`, `reduce`, `find`, `some`.
- Objects, spread syntax, optional chaining, nullish coalescing.
- Promises, `async` / `await`, error handling.
- Event handling and DOM fundamentals.
- Closures and lexical scope.
- Immutable update patterns.
- TypeScript interfaces, types, unions, generics, and narrowing.

Example immutable update:

```javascript
const updatedProduct = {
  ...product,
  name: "Updated name",
  attributes: product.attributes.map(attribute =>
    attribute.code === "COLOR"
      ? { ...attribute, value: "Blue" }
      : attribute
  )
};
```

Avoid mutating state objects or arrays directly.

## 3. Project setup and structure

A common modern setup uses Vite:

```bash
npm create vite@latest enterprise-portal -- --template react-ts
cd enterprise-portal
npm install
npm run dev
```

Common scripts:
```bash
npm run dev
npm run build
npm run preview
npm run lint
```

### Enterprise folder structure

```text
src/
  app/
    App.tsx
    router.tsx
    providers.tsx
  features/
    products/
      api/
      components/
      hooks/
      pages/
      types/
    categories/
    pricing/
    imports/
    users/
  shared/
    components/
    hooks/
    utils/
    types/
  services/
    http/
    auth/
  layouts/
  styles/
  test/
  main.tsx
```

Organize by business feature rather than creating huge global folders for every component type. Keep shared code genuinely reusable; do not move feature-specific logic into `shared` prematurely.

## 4. Components, props and composition

### Function component

```tsx
type ProductCardProps = {
  sku: string;
  name: string;
  status: "DRAFT" | "ACTIVE" | "DISCONTINUED";
  onOpen: (sku: string) => void;
};

export function ProductCard({
  sku,
  name,
  status,
  onOpen
}: ProductCardProps) {
  return (
    <article>
      <h2>{name}</h2>
      <p>{sku}</p>
      <span>{status}</span>
      <button onClick={() => onOpen(sku)}>View</button>
    </article>
  );
}
```

### Props vs state
- Props are passed into a component by its parent.
- State is managed by the component or a state-management layer.
- Do not copy props into state unless you intentionally need an independent editable snapshot.

### Composition
Prefer composing small components over building a component with dozens of boolean flags.

```tsx
function PageLayout({ children }: React.PropsWithChildren) {
  return <main className="page-layout">{children}</main>;
}
```

Use clear component boundaries:
- Page/container component: coordinates data and page-level state.
- Presentational component: renders data and emits user actions.
- Feature component: owns a coherent business UI capability.

## 5. JSX and rendering

JSX supports expressions inside braces:

```tsx
function ProductStatus({ active }: { active: boolean }) {
  return <span>{active ? "Active" : "Inactive"}</span>;
}
```

Conditional rendering:
```tsx
{isLoading ? <Spinner /> : error ? <ErrorMessage /> : <ProductTable />}
```

List rendering:
```tsx
{products.map(product => (
  <ProductRow key={product.id} product={product} />
))}
```

Use stable unique keys. Avoid array indexes as keys when rows can be inserted, removed, filtered, or reordered.

## 6. State management fundamentals

### useState
```tsx
const [search, setSearch] = useState("");

<input value={search} onChange={event => setSearch(event.target.value)} />
```

When new state depends on previous state, use a functional update:

```tsx
setPage(currentPage => currentPage + 1);
```

### State should be minimal
Do not store values that can be derived from other state.

```tsx
const activeProducts = products.filter(product => product.status === "ACTIVE");
```

Avoid storing both `products` and `activeProducts` unless there is a clear caching or synchronization reason.

### Lift state up
If sibling components need the same state, move it to their closest common parent and pass values/callbacks through props. For deeply shared state, consider Context or a state library.

## 7. React Hooks in depth

### useEffect
Use effects to synchronize with external systems: subscriptions, browser APIs, timers, or imperative integrations. Do not use an effect for every calculation or to mirror props into state.

```tsx
useEffect(() => {
  const controller = new AbortController();

  async function loadProduct() {
    try {
      const response = await fetch(`/api/products/${productId}`, {
        signal: controller.signal
      });
      if (!response.ok) throw new Error("Unable to load product");
      setProduct(await response.json());
    } catch (error) {
      if ((error as Error).name !== "AbortError") {
        setError("Product could not be loaded");
      }
    }
  }

  loadProduct();
  return () => controller.abort();
}, [productId]);
```

Important:
- Include all reactive dependencies.
- Clean up subscriptions and external resources.
- Handle stale requests and race conditions.
- Understand Strict Mode development behavior.
- Prefer a query/data library for complex server-state fetching.

### useMemo
Memoizes a computed value when recalculation is expensive or referential stability matters.

```tsx
const filteredProducts = useMemo(
  () => products.filter(product => product.name.toLowerCase().includes(query.toLowerCase())),
  [products, query]
);
```

Do not add `useMemo` everywhere. Measure first.

### useCallback
Memoizes a function reference, commonly useful when passing callbacks to memoized children or as dependencies.

```tsx
const openProduct = useCallback((id: number) => {
  navigate(`/products/${id}`);
}, [navigate]);
```

### useRef
Stores a mutable value that does not trigger rendering, or references a DOM element.

```tsx
const searchInputRef = useRef<HTMLInputElement>(null);

<button onClick={() => searchInputRef.current?.focus()}>Focus search</button>
<input ref={searchInputRef} />
```

### useContext
Shares values across a component subtree, such as theme or authenticated-user context. Avoid placing high-frequency, frequently changing global state into one broad context if it causes unnecessary rerenders.

### Custom hooks
Extract reusable behavior:

```tsx
function useProduct(productId: number) {
  const [product, setProduct] = useState<Product | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    async function load() {
      setLoading(true);
      setError(null);
      try {
        const response = await fetch(`/api/products/${productId}`, {
          signal: controller.signal
        });
        if (!response.ok) throw new Error("Request failed");
        setProduct(await response.json());
      } catch (e) {
        if ((e as Error).name !== "AbortError") setError("Unable to load product");
      } finally {
        if (!controller.signal.aborted) setLoading(false);
      }
    }

    load();
    return () => controller.abort();
  }, [productId]);

  return { product, loading, error };
}
```

For enterprise applications, use a shared API/data layer rather than duplicating fetch logic in many hooks.

## 8. Forms and validation

Forms commonly need:
- Controlled fields.
- Client-side validation for usability.
- Server-side validation as the source of truth.
- Field-level and form-level errors.
- Dirty/touched state.
- Submission/loading state.
- Protection against duplicate submissions.

Simple controlled form:

```tsx
type CreateProductForm = {
  sku: string;
  name: string;
};

function CreateProduct() {
  const [form, setForm] = useState<CreateProductForm>({ sku: "", name: "" });
  const [error, setError] = useState<string | null>(null);

  async function submit(event: React.FormEvent) {
    event.preventDefault();
    setError(null);

    if (!form.sku.trim() || !form.name.trim()) {
      setError("SKU and product name are required");
      return;
    }

    // Call the feature API and handle server validation errors.
  }

  return (
    <form onSubmit={submit}>
      <label>
        SKU
        <input
          value={form.sku}
          onChange={e => setForm(current => ({ ...current, sku: e.target.value }))}
        />
      </label>
      <label>
        Product name
        <input
          value={form.name}
          onChange={e => setForm(current => ({ ...current, name: e.target.value }))}
        />
      </label>
      {error && <p role="alert">{error}</p>}
      <button type="submit">Create product</button>
    </form>
  );
}
```

For complex forms, consider React Hook Form with a schema validator such as Zod. Validate again on the backend; client validation is not a security boundary.

## 9. Routing and layouts

React Router is a common client-side routing option.

Typical routes:
```text
/login
/products
/products/new
/products/:productId
/categories
/imports
/imports/:jobId
/admin/users
```

Example:
```tsx
const router = createBrowserRouter([
  {
    path: "/",
    element: <AppLayout />,
    children: [
      { path: "products", element: <ProductListPage /> },
      { path: "products/:productId", element: <ProductDetailsPage /> },
      { path: "imports/:jobId", element: <ImportJobPage /> }
    ]
  }
]);
```

Route protection improves navigation and user experience, but the backend must enforce authorization for every protected operation.

Use nested layouts for navigation/sidebar/header. Add not-found routes and error boundaries. For large applications, lazy-load route modules.

## 10. API integration with Spring Boot

### Recommended separation

```text
React Page
   |
Feature Hook / Query
   |
Feature API Module
   |
Shared HTTP Client
   |
API Gateway / Load Balancer
   |
Spring Boot REST API
   |
Service Layer
   |
Database / Kafka / Other Services
```

### Shared API client example
```tsx
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL;

export async function apiGet<T>(path: string): Promise<T> {
  const response = await fetch(`${API_BASE_URL}${path}`, {
    credentials: "include",
    headers: { Accept: "application/json" }
  });

  if (!response.ok) {
    throw new Error(`API request failed: ${response.status}`);
  }

  return response.json() as Promise<T>;
}
```

Do not assume all APIs return JSON (204 responses, downloads, and error bodies need handling). Centralize timeout, authentication, correlation IDs, error mapping, and serialization behavior.

### Server state vs client state
- Server state: products, prices, users, import job status from APIs.
- Client/UI state: open modal, selected tab, unsaved input, expanded row.

Use a query library such as TanStack Query for server state when appropriate. It provides caching, refetching, invalidation, loading/error state, and mutation workflows.

Example concept:
```tsx
const { data, isPending, error } = useQuery({
  queryKey: ["products", filters],
  queryFn: () => productApi.search(filters)
});
```

After a successful mutation, invalidate or update the relevant query cache. Define cache freshness and stale-data behavior deliberately.

## 11. State management choices

| Tool | Typical role | When it fits |
|---|---|---|
| useState/useReducer | Local state | Component or page state |
| Context | Shared subtree state | Theme, locale, auth session metadata |
| Redux Toolkit | Central client state | Complex cross-feature workflows and predictable event/state updates |
| Zustand | Lightweight shared state | Simple shared client state |
| TanStack Query | Server state | API caching, fetching, invalidation and mutations |
| URL/search params | Shareable navigation state | Filters, sort, pagination, selected resource |

Do not put every API response into Redux by default. Separate server state from UI/client state.

## 12. Enterprise component architecture

### Design system
Create reusable primitives and patterns:
- Buttons, inputs, selects, modals, alerts.
- Tables, pagination, filters.
- Form fields and validation messages.
- Loading, empty, and error states.
- Accessibility and keyboard behavior.
- Theme, typography, spacing, and tokens.

Prefer an established accessible component library or a well-governed internal design system. Avoid copying slightly different versions of the same component across features.

### Component responsibility
A useful feature component should have:
- One clear business/UI responsibility.
- Explicit props and event callbacks.
- Predictable state ownership.
- Testable behavior.
- Minimal knowledge of unrelated features.

### Feature module example
```text
features/products/
  api/productApi.ts
  components/ProductTable.tsx
  components/ProductFilters.tsx
  components/ProductForm.tsx
  hooks/useProducts.ts
  pages/ProductListPage.tsx
  pages/ProductDetailsPage.tsx
  types/product.ts
```

## 13. Performance optimization

### Common causes
- Large component rerenders.
- Unstable keys.
- Large lists/tables.
- Expensive derived computations.
- Too much global context.
- Large JavaScript bundles.
- Excessive API requests.
- Unoptimized images.
- Slow backend endpoints.

### Tools and techniques
- React DevTools Profiler.
- `React.memo` for measured rerender hotspots.
- `useMemo` / `useCallback` when they provide real value.
- Route-level code splitting with `React.lazy` and `Suspense`.
- List virtualization for large datasets.
- Debounced search.
- Server-side pagination/filtering.
- Image resizing and modern formats.
- Bundle analysis.
- Query caching and request deduplication.

### Debounced search
```tsx
function useDebouncedValue<T>(value: T, delayMs: number) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = window.setTimeout(() => setDebounced(value), delayMs);
    return () => window.clearTimeout(timer);
  }, [value, delayMs]);

  return debounced;
}
```

For thousands of product rows, use server-side filtering/pagination and virtualization rather than loading all rows into the browser.

## 14. Security for enterprise React applications

Important:
- Use a trusted identity provider and established OIDC/OAuth2 flow.
- Prefer Authorization Code with PKCE for browser-based public clients.
- Understand token storage trade-offs; avoid storing sensitive long-lived tokens in localStorage where possible.
- Consider secure, HttpOnly, SameSite cookies with an appropriate CSRF strategy.
- Enforce authorization on the backend, not only in React routes/buttons.
- Escape/render untrusted content safely; avoid unsafe HTML injection.
- Protect against XSS, CSRF, clickjacking, and dependency vulnerabilities.
- Do not put secrets in `VITE_*` variables; frontend build variables are visible to users.
- Validate file uploads on the server and restrict file size/type/content.
- Avoid logging tokens and sensitive personal/customer data.

A hidden button is not access control. A user can call the API directly.

## 15. Error handling and user experience

Enterprise UI should deliberately handle:
- Loading.
- Empty results.
- Validation errors.
- Authentication expiry.
- Authorization failure.
- Network timeout/offline state.
- Server errors.
- Conflict responses (e.g. stale update).
- Partial success in bulk operations.

Use error boundaries for render-time errors, and explicit handling for API and event-handler errors. Provide a correlation/request ID in support diagnostics without exposing internal stack traces.

## 16. Accessibility and responsive design

Accessibility is part of production quality:
- Use semantic HTML.
- Associate labels with form controls.
- Support keyboard navigation.
- Maintain visible focus indicators.
- Use correct heading order.
- Provide accessible names for icon buttons.
- Announce important async status updates appropriately.
- Ensure contrast and zoom support.
- Do not communicate status only through color.
- Test with keyboard and assistive technology.

Responsive layouts should support the real target devices and workflows, not just shrink a desktop table. Use mobile-friendly detail views or responsive table strategies.

## 17. Testing React applications

### Testing layers
- Unit tests: pure utilities and reducers.
- Component tests: user interactions and visible behavior.
- Integration tests: feature workflows with mocked network boundaries.
- End-to-end tests: critical user journeys in a browser.
- Accessibility tests: automated checks plus manual keyboard/screen-reader review.

Common tools:
- Vitest or Jest.
- React Testing Library.
- MSW (Mock Service Worker) for API mocking.
- Playwright or Cypress for end-to-end testing.

Example component test:
```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { ProductCard } from "./ProductCard";

test("opens selected product", async () => {
  const user = userEvent.setup();
  const onOpen = vi.fn();

  render(
    <ProductCard
      sku="SKU-1001"
      name="Cotton Shirt"
      status="ACTIVE"
      onOpen={onOpen}
    />
  );

  await user.click(screen.getByRole("button", { name: "View" }));
  expect(onOpen).toHaveBeenCalledWith("SKU-1001");
});
```

Test what users can observe rather than implementation details. Use accessible roles and labels in tests.

## 18. Build and deployment

### Build output
Vite produces static assets for production. A common deployment model:
- Build React application in CI.
- Upload static files to S3.
- Deliver globally through CloudFront.
- Use Route 53 for DNS.
- Use TLS certificate management.
- Configure SPA fallback so client-side routes resolve to the application entry point.
- Cache hashed assets aggressively and keep HTML cache policy appropriate for releases.

Alternative: serve static assets from a web server/container, but object storage + CDN is common for static frontends.

### Example AWS deployment
```text
Browser
  |
Route 53
  |
CloudFront + WAF (optional)
  |
S3 static website origin / private S3 origin with OAC
  |
React build assets

API requests:
Browser
  |
api.example.com
  |
ALB / API Gateway
  |
Spring Boot services on ECS/EKS/EC2
```

For production, prefer a private S3 origin with CloudFront Origin Access Control (OAC) rather than making the bucket publicly readable. Configure SPA route fallback, security headers, TLS, cache invalidation/versioning, and API CORS/cookie behavior.

### Environment configuration
```text
.env.development
.env.production
```

Build-time frontend environment variables are not secrets. Use them only for public configuration such as API base URL or feature configuration. Sensitive secrets belong on the server.

## 19. Enterprise case study A — Retail PIM / Product Catalog

### Business scenario
A merchandising platform manages products, SKUs, brands, categories, dynamic attributes, prices, imports, and channel publishing.

### Screens
1. Product search/list page.
2. Product details page.
3. Product create/edit form.
4. Dynamic attribute editor based on product type.
5. Category assignment screen.
6. Bulk import upload and progress page.
7. Product approval/publishing screen.
8. Audit history view.

### Example API contracts
```text
GET    /api/v1/products?query=shirt&categoryId=10&page=0&size=25
GET    /api/v1/products/{id}
POST   /api/v1/products
PATCH  /api/v1/products/{id}
GET    /api/v1/categories/tree
GET    /api/v1/product-types/{typeId}/attributes
POST   /api/v1/import-jobs
GET    /api/v1/import-jobs/{jobId}
POST   /api/v1/products/{id}/publish
```

### UI design considerations
- Use server-side pagination and filtering.
- Debounce free-text search.
- Load attribute definitions based on product type.
- Keep form draft state separate from server state.
- Display optimistic-lock conflicts when another user changes the product.
- Show row-level import errors and allow downloading an error report.
- Display channel publication status independently per channel.
- Use permission-aware UI, while enforcing all permissions in Spring Boot.

### Example dynamic attribute model
```tsx
type AttributeDefinition = {
  code: string;
  label: string;
  type: "TEXT" | "NUMBER" | "BOOLEAN" | "SELECT";
  required: boolean;
  options?: string[];
};

function AttributeField({
  definition,
  value,
  onChange
}: {
  definition: AttributeDefinition;
  value: string | number | boolean;
  onChange: (value: string | number | boolean) => void;
}) {
  if (definition.type === "BOOLEAN") {
    return (
      <label>
        <input
          type="checkbox"
          checked={Boolean(value)}
          onChange={e => onChange(e.target.checked)}
        />
        {definition.label}
      </label>
    );
  }

  if (definition.type === "SELECT") {
    return (
      <label>
        {definition.label}
        <select
          value={String(value)}
          onChange={e => onChange(e.target.value)}
        >
          <option value="">Select</option>
          {definition.options?.map(option => (
            <option key={option} value={option}>{option}</option>
          ))}
        </select>
      </label>
    );
  }

  return (
    <label>
      {definition.label}
      <input
        type={definition.type === "NUMBER" ? "number" : "text"}
        value={String(value)}
        onChange={e => onChange(
          definition.type === "NUMBER" ? Number(e.target.value) : e.target.value
        )}
      />
    </label>
  );
}
```

In production, also handle empty numeric values, decimal precision, locale, validation rules, controlled value types, and server-side validation.

## 20. Enterprise case study B — Bulk supplier import

### Workflow
1. User selects supplier and uploads file.
2. Frontend validates basic file size/type for usability.
3. File uploads to backend or a controlled S3 presigned upload flow.
4. Backend creates import job.
5. Worker processes rows asynchronously.
6. Frontend polls or subscribes to job status.
7. User sees progress, counts, and row-level errors.
8. User downloads error report and retries corrected rows.

UI states:
- `UPLOADING`
- `QUEUED`
- `VALIDATING`
- `PROCESSING`
- `COMPLETED`
- `PARTIALLY_FAILED`
- `FAILED`
- `CANCELLED`

Do not treat a successful file upload as a successful import. Display job status separately.

## 21. Enterprise case study C — Order management dashboard

Screens:
- Order search with date/status/customer filters.
- Order details and item lines.
- Payment state.
- Inventory reservation state.
- Shipment status.
- Audit timeline.

Design:
- Server-side filtering and pagination.
- Poll or subscribe to status updates at a controlled interval.
- Distinguish pending, failed, and confirmed operations.
- Use idempotency-aware retry actions.
- Show a safe error when a downstream system is unavailable.
- Enforce role/tenant permissions on the backend.

## 22. Enterprise case study D — Admin and access management

Features:
- User search.
- Role assignment.
- Permission matrix.
- Account activation/deactivation.
- Audit history.

Security:
- Require appropriate authorization for sensitive actions.
- Confirm destructive changes.
- Do not rely on UI role checks alone.
- Avoid exposing secrets or internal authorization claims unnecessarily.
- Log administrative actions on the server.

## 23. Common React enterprise mistakes

1. One giant component for an entire page.
2. Too many global state variables.
3. Duplicating server state in Redux and query cache.
4. Fetching data independently in many child components.
5. Missing loading, empty, and error states.
6. Unstable list keys.
7. Mutating arrays/objects in state.
8. Using `useEffect` for derived values.
9. Missing effect cleanup and request cancellation.
10. Putting secrets in frontend environment variables.
11. Treating hidden buttons as authorization.
12. Loading thousands of table rows at once.
13. No API contract/error standard.
14. No component or end-to-end tests.
15. Ignoring accessibility.
16. Overusing memoization without profiling.
17. Duplicating design-system components.
18. No route fallback/cache strategy in static hosting.

## 24. React interview questions

### Fundamentals
1. What is React and how does declarative rendering work?
2. Props vs state?
3. Controlled vs uncontrolled components?
4. Why are keys important?
5. What is reconciliation?
6. What causes a component to rerender?
7. Why should state be treated immutably?
8. What is lifting state up?

### Hooks
9. Rules of Hooks?
10. useState vs useReducer?
11. What is useEffect for?
12. How do effect dependencies work?
13. How do you prevent stale network responses?
14. useMemo vs useCallback?
15. useRef vs useState?
16. When should you create a custom hook?
17. What is Context and what are its performance trade-offs?

### Architecture and integration
18. How do you structure a large React application?
19. How do you separate server state from UI state?
20. How do you integrate React with Spring Boot?
21. How do you handle API errors globally?
22. How do you implement route-level authorization UX?
23. How do you handle token refresh/session expiry?
24. How do you design a dynamic form from backend metadata?
25. How do you prevent duplicate form submission?

### Performance, testing and security
26. How do you diagnose unnecessary rerenders?
27. How do you render a table with 100,000 records?
28. What is code splitting?
29. How do you test user interactions?
30. How do you mock APIs in integration tests?
31. How do you prevent XSS?
32. What is CSRF and when does it matter?
33. Why are frontend environment variables not secret?
34. How do you deploy a React SPA to S3 and CloudFront?
35. How do you handle cache invalidation after a release?

## 25. Hands-on exercises

### Core React
1. Build a product card with typed props.
2. Build a searchable product list.
3. Add filtering and sorting.
4. Add pagination.
5. Build a controlled product form.
6. Create a custom hook for a reusable input/search behavior.
7. Add route navigation and nested layouts.
8. Add a reusable modal and confirmation dialog.

### Enterprise workflows
9. Build a product list connected to a Spring Boot API.
10. Build product create/edit with server validation.
11. Build dynamic attributes based on product type.
12. Build a category tree selector.
13. Build a bulk import upload and job-status page.
14. Build a role-aware admin screen.
15. Add optimistic-lock conflict handling.
16. Add a channel publication status view.

### Quality and deployment
17. Add React Testing Library component tests.
18. Add MSW API integration tests.
19. Add Playwright critical user journeys.
20. Profile and optimize a large table.
21. Add keyboard accessibility.
22. Build the production bundle.
23. Deploy to private S3 + CloudFront.
24. Configure SPA fallback, cache policies, HTTPS, and API routing.

## 26. Suggested 30-day React learning plan

| Days | Focus | Deliverable |
|---|---|---|
| 1–3 | JavaScript/TypeScript refresh, JSX, components | Typed component library |
| 4–6 | Props, state, events, rendering | Product list |
| 7–10 | Hooks, effects, custom hooks | API-backed feature |
| 11–13 | Forms and validation | Product create/edit |
| 14–16 | Routing and layouts | Multi-page portal |
| 17–19 | API integration and server state | Spring Boot integration |
| 20–21 | Context/state management | Shared auth/UI state |
| 22–24 | Performance and accessibility | Large table and accessible forms |
| 25–26 | Testing | Component and API tests |
| 27–28 | Security and deployment | Production-ready build |
| 29–30 | Capstone and interview practice | Enterprise PIM UI |

## 27. Capstone — Enterprise PIM React Application

### Technology
- React + TypeScript + Vite.
- React Router.
- TanStack Query for server state.
- React Hook Form + Zod for complex forms (optional).
- A consistent component library/design system.
- Vitest + React Testing Library.
- MSW for API mocks.
- Playwright for end-to-end tests.
- Spring Boot REST APIs.
- AWS S3 + CloudFront for static frontend hosting.

### Features
- Login/session integration with an identity provider.
- Product search, filters, sort, pagination.
- Product create/edit with dynamic attributes.
- Category assignment.
- Product approval and publishing status.
- Supplier import upload and progress.
- Audit history.
- Role-aware navigation and actions.
- Responsive layout and accessible controls.

### Definition of done
- API data is cached and invalidated intentionally.
- Forms show client and server validation errors.
- Large lists are paginated/virtualized.
- Stale edits are handled with a conflict message.
- Loading, empty, error, and success states exist.
- Authorization is enforced by backend APIs.
- Tests cover critical user workflows.
- Production assets deploy through CI/CD to private S3 + CloudFront.
- No secrets are exposed in the frontend bundle.

---

## Official references
- React documentation: https://react.dev/
- React Learn: https://react.dev/learn
- TypeScript handbook: https://www.typescriptlang.org/docs/
- Vite documentation: https://vite.dev/guide/
- React Router: https://reactrouter.com/
- TanStack Query: https://tanstack.com/query/latest/docs/framework/react/overview
- React Hook Form: https://react-hook-form.com/
- Testing Library: https://testing-library.com/docs/react-testing-library/intro/
- Playwright: https://playwright.dev/docs/intro
- AWS CloudFront: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html
