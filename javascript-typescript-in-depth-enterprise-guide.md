# JavaScript and TypeScript — In-Depth Enterprise Study Guide

**Audience:** Java full-stack developers moving into modern frontend and full-stack TypeScript development  
**Approach:** Concepts → examples → enterprise use cases → exercises → interview preparation  
**Suggested baseline:** Modern JavaScript (ES2022+) and TypeScript 5.x. Examples use browser APIs and Node.js where relevant.

---

## 1. Learning objectives

By the end, you should be able to:

- Explain JavaScript execution, scope, closures, `this`, prototypes, and the event loop.
- Write modern JavaScript using modules, classes, promises, async/await, and array methods.
- Understand TypeScript's type system and use it to make application code safer.
- Design API contracts, reusable types, and error handling for a React frontend.
- Debug asynchronous behavior and common runtime errors.
- Build a small enterprise-style product catalog frontend that integrates with Spring Boot APIs.

## 2. JavaScript and TypeScript: what is the difference?

JavaScript (JS) is the language executed by browsers and JavaScript runtimes such as Node.js. TypeScript (TS) is a typed superset of JavaScript. TypeScript source is checked by the TypeScript compiler and typically emitted as JavaScript; browsers do not execute TypeScript syntax directly.

| Area | JavaScript | TypeScript |
|---|---|---|
| Typing | Dynamic at runtime | Static checking during development/build |
| Compilation | Usually executed directly or JIT-compiled by runtime | Transpiled/compiled to JavaScript |
| Interfaces | No native TS-style interfaces | Interfaces and type aliases |
| Generics | No compile-time generic types | Generic functions, classes, and interfaces |
| Runtime behavior | Actual runtime language | Same JavaScript runtime behavior after compilation |
| Best use | Small scripts, runtime code, prototypes | Larger applications, shared contracts, refactoring |

**Important:** TypeScript does not automatically validate external JSON at runtime. Data received from an API is untrusted until validated or checked.

## 3. JavaScript fundamentals

### 3.1 Variables: `var`, `let`, and `const`

```js
var oldStyle = "function scoped";
let count = 1;                 // block scoped, can be reassigned
const applicationName = "PIM"; // block scoped, cannot be reassigned

count = 2;
// applicationName = "Store"; // TypeError
```

Prefer `const` by default, `let` when reassignment is required, and avoid `var` in modern application code.

`const` prevents reassignment of the binding; it does not freeze an object:

```js
const product = { name: "Laptop" };
product.name = "Gaming Laptop"; // allowed
// product = {};                // not allowed
```

### 3.2 Primitive and reference values

Primitive types include `string`, `number`, `bigint`, `boolean`, `undefined`, `symbol`, and `null`.

Objects, arrays, and functions are reference values.

```js
let a = 10;
let b = a;
b = 20;
console.log(a); // 10

const first = { status: "ACTIVE" };
const second = first;
second.status = "INACTIVE";
console.log(first.status); // INACTIVE
```

### 3.3 Equality and coercion

Prefer strict equality (`===` and `!==`) because it avoids implicit type conversion.

```js
0 == false;  // true
0 === false; // false
"5" == 5;    // true
"5" === 5;   // false
```

Use explicit conversion when needed: `Number(value)`, `String(value)`, `Boolean(value)`.

### 3.4 Truthy and falsy values

Falsy values include `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, and `NaN`. Empty arrays and empty objects are truthy.

```js
const items = [];
if (items) {
  // This block executes: [] is truthy.
}
```

Use nullish coalescing (`??`) when only `null` and `undefined` should trigger a default:

```js
const pageSize = userSetting ?? 20;
```

Use optional chaining (`?.`) for safe nested access:

```js
const city = customer?.address?.city;
```

### 3.5 Destructuring, spread, and rest

```js
const product = { id: 101, name: "Phone", price: 25000 };
const { id, name } = product;

const updated = { ...product, price: 23000 };
const numbers = [1, 2, 3];
const expanded = [...numbers, 4];

function logProduct(name, ...tags) {
  console.log(name, tags);
}
```

Spread creates a shallow copy, not a deep clone. Nested objects remain shared references.

## 4. Scope, hoisting, and closures

### 4.1 Scope

- Global scope: available throughout the relevant script/module environment.
- Function scope: variables declared inside a function.
- Block scope: variables declared with `let` and `const` inside `{}`.
- Module scope: top-level declarations in an ES module are not automatically global properties.

### 4.2 Hoisting and the temporal dead zone

Declarations are processed before execution, but `var`, `let`, `const`, and function declarations behave differently.

```js
console.log(value); // undefined
var value = 10;

// console.log(total); // ReferenceError: temporal dead zone
let total = 20;

sayHello(); // works
function sayHello() {
  console.log("Hello");
}
```

Avoid relying on hoisting; declare variables before use.

### 4.3 Closures

A closure is a function together with access to its lexical environment.

```js
function createSequence() {
  let current = 0;

  return function next() {
    current += 1;
    return current;
  };
}

const nextId = createSequence();
console.log(nextId()); // 1
console.log(nextId()); // 2
```

**Enterprise uses:** encapsulating module state, creating callbacks, debouncing search, and implementing factories.

### 4.4 Debounce example

```js
function debounce(callback, delayMs) {
  let timerId;

  return (...args) => {
    clearTimeout(timerId);
    timerId = setTimeout(() => callback(...args), delayMs);
  };
}

const searchProducts = debounce((term) => {
  console.log("Search API for:", term);
}, 300);
```

For production search, also handle request cancellation and stale responses; debouncing alone does not guarantee response order.

## 5. Functions and `this`

### 5.1 Function forms

```js
function add(a, b) {
  return a + b;
}

const multiply = function (a, b) {
  return a * b;
};

const subtract = (a, b) => a - b;
```

Arrow functions do not create their own `this`, `arguments`, or `new.target`. They capture `this` lexically.

### 5.2 Understanding `this`

For a regular function, `this` is determined by how the function is called (with strict-mode and runtime-specific rules).

```js
const user = {
  name: "Ravi",
  printName() {
    console.log(this.name);
  }
};

user.printName(); // Ravi
```

Extracting the method changes the call-site receiver:

```js
const print = user.printName;
print(); // undefined in strict mode (or global-related behavior in non-strict scripts)
```

Bind a receiver when required:

```js
const boundPrint = user.printName.bind(user);
boundPrint();
```

### 5.3 `call`, `apply`, and `bind`

```js
function introduce(city, country) {
  return `${this.name} from ${city}, ${country}`;
}

const person = { name: "Anil" };
introduce.call(person, "Hyderabad", "India");
introduce.apply(person, ["Hyderabad", "India"]);

const introduceAnil = introduce.bind(person);
introduceAnil("Hyderabad", "India");
```

## 6. Objects, prototypes, and classes

JavaScript uses prototype-based inheritance. Objects can delegate property lookup to another object through their prototype chain.

```js
const baseEntity = {
  describe() {
    return `Entity ${this.id}`;
  }
};

const product = Object.create(baseEntity);
product.id = 501;
console.log(product.describe());
```

Classes provide syntax over JavaScript's prototype system:

```js
class Product {
  constructor(id, name, price) {
    this.id = id;
    this.name = name;
    this.price = price;
  }

  applyDiscount(percent) {
    this.price = this.price * (1 - percent / 100);
  }
}

const item = new Product(1, "Laptop", 80000);
item.applyDiscount(10);
```

Inheritance:

```js
class DigitalProduct extends Product {
  constructor(id, name, price, downloadUrl) {
    super(id, name, price);
    this.downloadUrl = downloadUrl;
  }
}
```

Use composition when it keeps behavior simpler than a deep inheritance hierarchy.

## 7. Arrays and important methods

```js
const products = [
  { id: 1, name: "Laptop", price: 80000, active: true },
  { id: 2, name: "Mouse", price: 1200, active: true },
  { id: 3, name: "Old Phone", price: 5000, active: false }
];

const names = products.map(p => p.name);
const activeProducts = products.filter(p => p.active);
const laptop = products.find(p => p.id === 1);
const total = products.reduce((sum, p) => sum + p.price, 0);
const hasInactive = products.some(p => !p.active);
const allPriced = products.every(p => p.price > 0);
```

- `map`: transform each element.
- `filter`: keep matching elements.
- `find`: return first matching element or `undefined`.
- `some`: true if at least one element matches.
- `every`: true if all elements match.
- `reduce`: accumulate a result.
- `forEach`: perform a side effect; it does not return a transformed array.

Do not mutate shared state accidentally. Methods such as `sort`, `reverse`, `push`, and `splice` mutate the original array.

```js
const sorted = [...products].sort((a, b) => a.price - b.price);
```

## 8. Asynchronous JavaScript

### 8.1 Call stack and event loop

JavaScript commonly runs application code on a single main thread per execution context. The runtime coordinates the call stack with queues and host facilities such as timers, network I/O, and rendering.

A simplified mental model:
1. Synchronous code runs on the call stack.
2. Asynchronous work is handled by the host runtime.
3. Promise reactions are queued as microtasks.
4. Tasks such as timer callbacks run from the task queue.
5. The runtime processes microtasks at defined checkpoints.

```js
console.log("A");

setTimeout(() => console.log("timer"), 0);

Promise.resolve().then(() => console.log("promise"));

console.log("B");

// A
// B
// promise
// timer
```

The exact runtime has more detail, but this ordering is a useful starting point.

### 8.2 Callback pattern

```js
function loadProduct(id, callback) {
  setTimeout(() => {
    callback(null, { id, name: "Laptop" });
  }, 100);
}

loadProduct(10, (error, product) => {
  if (error) {
    console.error(error);
    return;
  }
  console.log(product);
});
```

Nested callbacks can become difficult to read and maintain.

### 8.3 Promises

A Promise represents the eventual completion or failure of an asynchronous operation.

```js
function fetchProduct(id) {
  return fetch(`/api/products/${id}`).then(response => {
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return response.json();
  });
}

fetchProduct(10)
  .then(product => console.log(product))
  .catch(error => console.error("Product request failed", error));
```

States: pending, fulfilled, rejected. A settled Promise does not change state again.

### 8.4 `async` / `await`

```js
async function loadProduct(id) {
  try {
    const response = await fetch(`/api/products/${id}`);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    return await response.json();
  } catch (error) {
    console.error("Unable to load product", error);
    throw error;
  }
}
```

`async` functions always return a Promise. `await` pauses that async function, not the entire JavaScript thread.

### 8.5 Sequential vs parallel requests

Sequential when the second request depends on the first:

```js
const product = await getProduct(id);
const inventory = await getInventory(product.sku);
```

Parallel when independent:

```js
const [product, categories] = await Promise.all([
  getProduct(id),
  getCategories()
]);
```

Useful Promise methods:
- `Promise.all`: rejects as soon as one input rejects.
- `Promise.allSettled`: waits for all inputs to settle.
- `Promise.race`: settles with the first settled input.
- `Promise.any`: fulfills with the first fulfilled input; rejects if all reject.

### 8.6 Fetch, cancellation, and HTTP errors

`fetch` rejects for network-level failures, but an HTTP 404 or 500 normally still fulfills with a `Response`. Check `response.ok`.

```js
async function getJson(url, signal) {
  const response = await fetch(url, { signal });

  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`);
  }

  return response.json();
}

const controller = new AbortController();
getJson("/api/products", controller.signal);
// controller.abort(); // cancel when no longer needed
```

## 9. Modules and browser fundamentals

Use ES modules to organize code:

```js
// productService.js
export async function getProducts() {
  const response = await fetch("/api/products");
  if (!response.ok) throw new Error("Could not load products");
  return response.json();
}

// productPage.js
import { getProducts } from "./productService.js";
```

Browser concepts to understand:
- DOM: document object model.
- Events: click, input, submit, change.
- Event bubbling and capturing.
- Event delegation.
- `localStorage` and `sessionStorage` (not suitable for secrets).
- Cookies and their security attributes.
- CORS: browser-enforced cross-origin access rules.
- Same-origin policy.
- XSS and safe rendering.
- Network tab, console, sources, performance and application panels in browser DevTools.

Event delegation example:

```js
document.querySelector("#product-list").addEventListener("click", event => {
  const button = event.target.closest("[data-action='delete']");
  if (!button) return;

  const productId = button.dataset.productId;
  console.log("Delete product:", productId);
});
```

## 10. TypeScript foundations

### 10.1 Basic types and annotations

```ts
let productName: string = "Laptop";
let price: number = 80000;
let isActive: boolean = true;
let tags: string[] = ["electronics", "computer"];

let scores: Array<number> = [80, 90, 95];
```

TypeScript can infer types, so annotations are not needed everywhere:

```ts
const quantity = 5; // inferred as number
```

### 10.2 `any`, `unknown`, and `never`

- `any`: disables most checking; use sparingly.
- `unknown`: accepts any value, but requires narrowing before use.
- `never`: represents a value that cannot occur, often used for exhaustive checks.

```ts
function parseExternalValue(value: unknown): string {
  if (typeof value === "string") {
    return value.trim();
  }
  throw new Error("Expected a string");
}
```

### 10.3 Interfaces and type aliases

```ts
interface Product {
  id: number;
  name: string;
  price: number;
  active: boolean;
}

type ProductId = number;
type ProductStatus = "DRAFT" | "ACTIVE" | "ARCHIVED";
```

Interfaces are useful for object contracts and can be extended/declaration-merged. Type aliases can express unions, intersections, primitives, tuples, and mapped/conditional types. Both are widely used for object shapes.

### 10.4 Optional and readonly properties

```ts
interface ProductSummary {
  readonly id: number;
  name: string;
  description?: string;
}
```

`readonly` is a compile-time restriction, not runtime immutability. Optional properties may be absent; check before using them.

### 10.5 Union and intersection types

```ts
type ProductStatus = "DRAFT" | "ACTIVE" | "ARCHIVED";

type Identified = { id: number };
type Audited = { createdAt: string; updatedAt: string };
type AuditedProduct = Identified & Audited & { name: string };
```

### 10.6 Literal types and discriminated unions

```ts
type ApiResult<T> =
  | { status: "success"; data: T }
  | { status: "error"; message: string };

function showResult<T>(result: ApiResult<T>) {
  if (result.status === "success") {
    console.log(result.data);
  } else {
    console.error(result.message);
  }
}
```

Discriminated unions make state handling explicit and help prevent invalid combinations.

### 10.7 Type narrowing and guards

```ts
function formatValue(value: string | number) {
  if (typeof value === "number") {
    return value.toFixed(2);
  }
  return value.toUpperCase();
}

interface Supplier {
  supplierCode: string;
}

function isSupplier(value: unknown): value is Supplier {
  return typeof value === "object"
    && value !== null
    && "supplierCode" in value
    && typeof (value as { supplierCode: unknown }).supplierCode === "string";
}
```

A type assertion (`as`) tells the compiler to trust you; it does not validate the value at runtime.

## 11. TypeScript functions, generics, and utility types

### 11.1 Function types

```ts
type DiscountCalculator = (price: number, percent: number) => number;

const calculateDiscount: DiscountCalculator = (price, percent) =>
  price * (1 - percent / 100);
```

### 11.2 Generics

Generics preserve the relationship between input and output types.

```ts
function identity<T>(value: T): T {
  return value;
}

function first<T>(items: T[]): T | undefined {
  return items[0];
}

interface ApiResponse<T> {
  data: T;
  message: string;
  timestamp: string;
}

const response: ApiResponse<Product[]> = {
  data: [],
  message: "Loaded",
  timestamp: new Date().toISOString()
};
```

Constrained generic:

```ts
function getById<T extends { id: number }>(items: T[], id: number) {
  return items.find(item => item.id === id);
}
```

### 11.3 Utility types

```ts
type ProductCreate = Omit<Product, "id">;
type ProductUpdate = Partial<ProductCreate>;
type ProductView = Pick<Product, "id" | "name" | "price">;
type ReadonlyProduct = Readonly<Product>;
type ProductMap = Record<number, Product>;
```

Common utilities: `Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record`, `Exclude`, `Extract`, `NonNullable`, `ReturnType`, `Parameters`, `Awaited`.

### 11.4 Enums vs string unions

```ts
type Status = "DRAFT" | "ACTIVE" | "ARCHIVED";
```

String unions are often straightforward for API contracts. Enums are also available, but remember that regular enums can generate runtime JavaScript. Choose consistently with the team's conventions.

## 12. TypeScript classes and object-oriented design

```ts
interface Discountable {
  applyDiscount(percent: number): void;
}

class CatalogProduct implements Discountable {
  constructor(
    public readonly id: number,
    public name: string,
    private price: number
  ) {}

  applyDiscount(percent: number): void {
    if (percent < 0 || percent > 100) {
      throw new Error("Invalid discount");
    }
    this.price *= 1 - percent / 100;
  }

  getPrice(): number {
    return this.price;
  }
}
```

Access modifiers: `public`, `private`, `protected`. TypeScript's `private` is primarily compile-time enforced; ECMAScript `#private` fields provide runtime privacy.

## 13. TypeScript project configuration

A practical starting `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noImplicitAny": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist"
  },
  "include": ["src"]
}
```

For a browser app built with Vite, use a compatible Vite TypeScript configuration and scripts. For Node.js, select module settings that match the chosen Node module system. Do not copy one module configuration blindly across browser and server projects.

Example `package.json` scripts:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit"
  }
}
```

Typical commands:

```bash
npm install
npm run dev
npm run typecheck
npm run build
```

## 14. Integrating TypeScript frontend with Spring Boot

Assume Spring Boot exposes:

```http
GET /api/products
GET /api/products/{id}
POST /api/products
PUT /api/products/{id}
DELETE /api/products/{id}
```

Define frontend contracts:

```ts
export interface ProductDto {
  id: number;
  sku: string;
  name: string;
  price: number;
  status: "DRAFT" | "ACTIVE" | "ARCHIVED";
}

export interface ProductCreateRequest {
  sku: string;
  name: string;
  price: number;
}
```

Create an API client:

```ts
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL;

async function request<T>(path: string, init?: RequestInit): Promise<T> {
  const response = await fetch(`${API_BASE_URL}${path}`, {
    ...init,
    headers: {
      "Content-Type": "application/json",
      ...init?.headers
    }
  });

  if (!response.ok) {
    const message = await response.text();
    throw new Error(message || `HTTP ${response.status}`);
  }

  if (response.status === 204) {
    return undefined as T;
  }

  return response.json() as Promise<T>;
}

export const productApi = {
  getAll: () => request<ProductDto[]>("/api/products"),
  getById: (id: number) => request<ProductDto>(`/api/products/${id}`),
  create: (body: ProductCreateRequest) =>
    request<ProductDto>("/api/products", {
      method: "POST",
      body: JSON.stringify(body)
    }),
  update: (id: number, body: ProductCreateRequest) =>
    request<ProductDto>(`/api/products/${id}`, {
      method: "PUT",
      body: JSON.stringify(body)
    }),
  delete: (id: number) =>
    request<void>(`/api/products/${id}`, { method: "DELETE" })
};
```

**Production note:** The generic `request<T>` function assumes the server response matches `T`; it does not validate JSON. Add runtime schema validation (for example, with Zod) at trust boundaries. Also handle authentication, refresh flows, correlation IDs, timeouts, and structured API errors according to the application architecture.

## 15. Enterprise case study: Product Information Management (PIM)

### Requirements

Build a catalog screen where users can:
- View products with SKU, name, price, status, and category.
- Search by SKU/name.
- Filter by status and category.
- Create and edit products.
- Display API loading, empty, and error states.
- Prevent invalid form submissions.

### Suggested folder structure

```text
src/
  app/
    App.tsx
    router.tsx
  components/
    DataTable.tsx
    SearchInput.tsx
    StatusBadge.tsx
  features/
    products/
      api/
        productApi.ts
      components/
        ProductForm.tsx
        ProductTable.tsx
      hooks/
        useProducts.ts
      pages/
        ProductListPage.tsx
      types/
        product.types.ts
  shared/
    components/
    utils/
  main.tsx
```

### Search and filter logic

```ts
function filterProducts(
  products: ProductDto[],
  search: string,
  status?: ProductDto["status"]
): ProductDto[] {
  const normalizedSearch = search.trim().toLowerCase();

  return products.filter(product => {
    const matchesSearch =
      product.name.toLowerCase().includes(normalizedSearch) ||
      product.sku.toLowerCase().includes(normalizedSearch);

    const matchesStatus = status === undefined || product.status === status;
    return matchesSearch && matchesStatus;
  });
}
```

For large datasets, prefer server-side pagination, filtering, and sorting instead of downloading every record.

### API state model

```ts
type LoadState<T> =
  | { state: "loading" }
  | { state: "success"; data: T }
  | { state: "error"; message: string };

const initialState: LoadState<ProductDto[]> = { state: "loading" };
```

A discriminated union prevents contradictory states such as `loading = true` while also holding an unrelated error and stale data without an explicit policy.

## 16. JavaScript and TypeScript security essentials

- Never put passwords, API secrets, or private keys in frontend bundles.
- Treat API responses, URL parameters, local storage, and user input as untrusted.
- Avoid `eval` and unsafe HTML injection.
- Prefer framework-safe rendering and context-appropriate output encoding.
- Use HTTPS.
- Configure CORS on the server; CORS is not an authentication mechanism.
- Use secure cookie attributes where cookie-based sessions are used: `HttpOnly`, `Secure`, and an appropriate `SameSite`.
- Avoid storing long-lived sensitive tokens in `localStorage` unless the security architecture explicitly accepts the XSS exposure.
- Validate authorization on the backend; hiding a button in the UI is not access control.
- Apply dependency scanning and keep packages updated.

## 17. Testing and code quality

Recommended layers:
- Unit tests for pure functions and utilities.
- Component tests for UI behavior.
- API tests for integration contracts.
- End-to-end tests for critical user journeys.
- Type checking and linting in CI.

Example pure-function test (Vitest):

```ts
import { describe, expect, it } from "vitest";

describe("filterProducts", () => {
  it("filters by SKU or name", () => {
    const products = [
      { id: 1, sku: "LAP-1", name: "Laptop", price: 100, status: "ACTIVE" },
      { id: 2, sku: "MOU-1", name: "Mouse", price: 20, status: "DRAFT" }
    ] satisfies ProductDto[];

    expect(filterProducts(products, "lap")).toHaveLength(1);
  });
});
```

For component testing, learn React Testing Library. For browser workflows, learn Playwright. Test behavior and outcomes rather than implementation details.

## 18. Common mistakes and how to avoid them

| Mistake | Better approach |
|---|---|
| Using `any` everywhere | Prefer precise types and `unknown` at untrusted boundaries |
| Assuming TS validates API JSON | Validate runtime data |
| Forgetting `fetch` HTTP status checks | Check `response.ok` |
| Mutating arrays used by UI state | Use immutable transformations |
| Using `==` without a reason | Prefer `===` |
| Unhandled Promise rejection | Handle or propagate errors |
| Calling independent APIs sequentially | Use `Promise.all` when appropriate |
| Using `as` to silence every type error | Narrow and validate values |
| Putting secrets in frontend code | Keep secrets on backend |
| Ignoring loading/empty/error states | Model all UI states explicitly |
| Creating huge components | Split by feature and responsibility |
| Fetching all rows for a large table | Use server-side pagination/filtering |

## 19. Interview questions

### JavaScript fundamentals
1. Difference between `var`, `let`, and `const`?
2. What is hoisting?
3. What is the temporal dead zone?
4. Explain closure with a practical example.
5. How does `this` work in regular and arrow functions?
6. Difference between `==` and `===`?
7. Explain prototype inheritance.
8. Difference between `map`, `filter`, and `reduce`?
9. What is event bubbling?
10. What is event delegation?

### Asynchronous JavaScript
11. Explain the event loop.
12. What is the difference between a task and a microtask?
13. Difference between callbacks, Promises, and async/await?
14. What happens when one Promise in `Promise.all` rejects?
15. Difference between `Promise.all` and `Promise.allSettled`?
16. Why does `fetch` not reject for HTTP 404?
17. How do you cancel a fetch request?
18. How do you avoid stale search responses?

### TypeScript
19. Difference between `interface` and `type`?
20. Difference between `any` and `unknown`?
21. What is `never`?
22. What are union and intersection types?
23. What is type narrowing?
24. What are generics and why are they useful?
25. Explain `Partial`, `Pick`, `Omit`, and `Record`.
26. What is a discriminated union?
27. Does a type assertion validate runtime data?
28. What does `strict` mode do?

### Enterprise frontend
29. How would you structure a large TypeScript frontend?
30. How do you handle API errors consistently?
31. How do you model loading, success, empty, and error states?
32. How do you secure a frontend application?
33. How would you handle pagination for millions of products?
34. How do you prevent duplicate form submissions?
35. How do you test a product search and filter feature?
36. How do you manage API contract changes between Spring Boot and TypeScript?

## 20. Hands-on exercises

1. Write a function to group products by status.
2. Calculate total catalog value using `reduce`.
3. Sort products by price without mutating the original array.
4. Implement a debounce utility with TypeScript generics.
5. Write a typed API client with reusable error handling.
6. Model API success and failure using a discriminated union.
7. Create a type guard for an unknown API response.
8. Fetch product and category data concurrently.
9. Cancel a search request when the user changes the query.
10. Implement client-side search and status filtering.
11. Add server-side pagination and sorting.
12. Write tests for discount calculation and product filtering.
13. Build a create-product form with validation.
14. Display loading, empty, error, and success states.
15. Add role-based UI visibility while enforcing authorization on the backend.

## 21. 30-day learning plan

| Days | Focus | Deliverable |
|---|---|---|
| 1–3 | JS syntax, variables, types, operators | Small scripts and exercises |
| 4–6 | Scope, closures, functions, `this` | Debounce and utility functions |
| 7–9 | Objects, prototypes, classes, arrays | Product transformation exercises |
| 10–12 | Promises, async/await, event loop, fetch | API integration exercises |
| 13–15 | Modules, browser APIs, DOM, events | Searchable browser page |
| 16–18 | TS types, interfaces, unions, narrowing | Typed domain models |
| 19–21 | Generics, utility types, classes, tsconfig | Reusable API client |
| 22–24 | Spring Boot API integration and error states | Product list and detail pages |
| 25–27 | Testing, security, performance | Tests and security review |
| 28–30 | Capstone and interview revision | Working PIM catalog UI |

## 22. Capstone: Enterprise Product Catalog UI

Build a TypeScript frontend connected to a Spring Boot REST API.

**Features**
- Product list with server-side pagination.
- Search by SKU and product name.
- Filter by category and lifecycle status.
- Create, edit, view, and archive products.
- Form validation and clear API error messages.
- Loading, empty, and retry states.
- Role-aware actions.
- Unit and component tests.
- Environment-specific API configuration.

**Suggested implementation order**
1. Define TypeScript DTOs and API contracts.
2. Build the typed API client.
3. Implement list, search, and pagination.
4. Add create/edit forms and validation.
5. Add error and loading states.
6. Add tests and accessibility checks.
7. Integrate authentication and authorization.
8. Build and deploy the frontend.

## 23. Recommended mastery order

For a Java full-stack developer, use this order:

1. JavaScript fundamentals — understand runtime behavior, not just syntax.
2. Asynchronous JavaScript — Promises, event loop, fetch, cancellation.
3. TypeScript — strict typing, unions, generics, utility types.
4. Browser and HTTP fundamentals — DOM, events, CORS, cookies, security.
5. React with TypeScript — components, hooks, forms, routing, state and data fetching.
6. Enterprise practices — testing, performance, accessibility, CI/CD and API contracts.

**Practical target:** Be able to independently build a typed React screen that consumes a Spring Boot API, handles errors and pagination, and has meaningful tests. That is a stronger job-ready outcome than memorizing language definitions.
