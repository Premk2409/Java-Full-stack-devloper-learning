# Python — In-Depth Enterprise Study Guide

**Audience:** Java full-stack developers learning Python for backend services, automation, data engineering, and AI/LLM application development  
**Approach:** Fundamentals → internals → practical code → enterprise use cases → exercises → interview preparation  
**Suggested baseline:** Python 3.11+ (examples are compatible with modern Python 3.x unless stated otherwise).

---

## 1. Learning objectives

By the end, you should be able to:

- Write readable Python using its core data types, control flow, functions, and modules.
- Understand Python object references, mutability, scope, iterators, generators, and exceptions.
- Use classes, dataclasses, type hints, and common object-oriented patterns.
- Build REST APIs with FastAPI and connect to databases.
- Write tests, package applications, manage dependencies, and configure environments.
- Use Python as an AI/ML integration layer alongside Java and Spring Boot.
- Build a small enterprise product-data API and understand how to extend it for AI workflows.

## 2. What is Python?

Python is a high-level, general-purpose programming language known for readable syntax and a broad ecosystem. It is widely used in backend development, automation, scripting, data engineering, machine learning, and AI applications.

### Java vs Python: practical comparison

| Topic | Java | Python |
|---|---|---|
| Typing | Statically typed | Dynamically typed; optional type hints |
| Compilation/execution | Compiled to JVM bytecode, run on JVM | Common implementations compile to bytecode and execute on a Python VM |
| Memory management | Garbage collection on JVM | Automatic memory management; CPython commonly uses reference counting plus cyclic GC |
| Concurrency | Threads, executors, virtual threads | Threads, processes, async I/O; CPython GIL behavior depends on build/version |
| Packaging | Maven/Gradle, JAR/WAR | `pip`, `venv`, `pyproject.toml`, wheels |
| Common backend frameworks | Spring Boot, Quarkus | FastAPI, Django, Flask |
| Common AI ecosystem | Growing | Broad Python-first ecosystem |

Python's dynamic typing does not mean applications must be unstructured. Type hints, tests, linters, clear module boundaries, and runtime validation provide strong engineering discipline.

## 3. Install Python and set up a project

Check the installed version:

```bash
python --version
# or
python3 --version
```

Create and activate a virtual environment:

```bash
mkdir product-data-service
cd product-data-service

python3 -m venv .venv

# macOS/Linux
source .venv/bin/activate

# Windows PowerShell
.venv\Scripts\Activate.ps1
```

Install packages:

```bash
python -m pip install --upgrade pip
python -m pip install fastapi uvicorn sqlalchemy pydantic
```

A virtual environment isolates project dependencies. Do not install every project dependency globally.

A simple initial structure:

```text
product-data-service/
  .venv/
  app/
    __init__.py
    main.py
  tests/
  pyproject.toml
  README.md
```

## 4. Python syntax and core data types

### 4.1 Variables and naming

Python variables are names bound to objects. You do not declare a variable's type in ordinary assignments.

```python
product_name = "Laptop"
price = 85000.0
quantity = 5
is_active = True

print(product_name, price, quantity, is_active)
```

Use `snake_case` for variables and functions, `PascalCase` for classes, and uppercase names for module-level constants by convention.

### 4.2 Built-in types

```python
name = "Laptop"                 # str
price = 85000                   # int
rating = 4.5                    # float
is_available = True             # bool
nothing = None                  # NoneType

tags = ["electronics", "pc"]    # list
dimensions = (15, 10, 2)         # tuple
categories = {"Electronics"}    # set
product = {"id": 1, "name": "Laptop"}  # dict
```

Check a type with `type(value)` or, for class-aware checks, `isinstance(value, SomeType)`.

### 4.3 Strings and formatting

```python
sku = "LAP-100"
name = "Laptop"
message = f"Product {name} has SKU {sku}"

normalized = "  laptop  ".strip().lower()
parts = "electronics,computers".split(",")
joined = " | ".join(parts)
```

Prefer f-strings for readable interpolation. Be careful when formatting untrusted values into SQL or HTML; use parameterized queries and safe rendering instead.

### 4.4 Numeric operations

```python
total = 10 + 5
remainder = 10 % 3
power = 2 ** 4
integer_division = 10 // 3
```

For financial values, avoid binary floating-point rounding surprises. Use `decimal.Decimal`:

```python
from decimal import Decimal

unit_price = Decimal("19.99")
quantity = 3
total = unit_price * quantity
```

Construct `Decimal` from strings when exact decimal input matters.

## 5. Collections: list, tuple, set, and dictionary

### 5.1 Lists

Lists are ordered and mutable.

```python
products = ["Laptop", "Mouse", "Keyboard"]
products.append("Monitor")
products[0] = "Gaming Laptop"

first_two = products[:2]
```

Common operations: `append`, `extend`, `insert`, `remove`, `pop`, `sort`, `reverse`, slicing, and membership with `in`.

### 5.2 Tuples

Tuples are ordered and immutable at the container level.

```python
coordinates = (17.45, 78.50)
latitude, longitude = coordinates
```

An immutable tuple can still contain a mutable object; immutability is not recursively applied to its contents.

### 5.3 Sets

Sets hold unique hashable elements and are useful for membership tests and set operations.

```python
requested = {"SKU-1", "SKU-2", "SKU-2"}
available = {"SKU-2", "SKU-3"}

print(requested & available)  # intersection
print(requested | available)  # union
print(requested - available)  # difference
```

### 5.4 Dictionaries

Dictionaries map hashable keys to values.

```python
product = {
    "id": 101,
    "sku": "LAP-101",
    "name": "Laptop",
    "price": 85000
}

product["status"] = "ACTIVE"
price = product.get("price", 0)
```

Iterate with `.items()`, `.keys()`, and `.values()`.

### 5.5 Comprehensions

```python
prices = [100, 200, 300, 400]
discounted = [price * 0.9 for price in prices if price >= 200]

price_by_sku = {
    product["sku"]: product["price"]
    for product in [
        {"sku": "A1", "price": 100},
        {"sku": "B2", "price": 200}
    ]
}
```

Use comprehensions for simple transformations; use ordinary loops when the logic becomes difficult to read.

## 6. Mutability, identity, and copying

Names refer to objects. Assignment does not automatically copy an object.

```python
first = ["A", "B"]
second = first
second.append("C")

print(first)  # ['A', 'B', 'C']
```

A shallow copy creates a new outer container but retains references to nested objects:

```python
import copy

original = {"product": {"name": "Laptop"}, "tags": ["pc"]}
shallow = original.copy()
deep = copy.deepcopy(original)

shallow["tags"].append("gaming")
# original's tags also change

deep["product"]["name"] = "Tablet"
# original's nested product name does not change
```

Use `is` to test object identity and `==` to test value equality. Use `is None` for `None` checks.

## 7. Control flow and pattern matching

```python
status = "ACTIVE"

if status == "ACTIVE":
    print("Available")
elif status == "DRAFT":
    print("Not published")
else:
    print("Unavailable")

for number in range(1, 5):
    print(number)

while True:
    break
```

`range(start, stop, step)` excludes the stop value. Use `enumerate()` for index/value pairs and `zip()` to iterate over multiple sequences.

```python
for index, sku in enumerate(["A1", "B2"], start=1):
    print(index, sku)

for sku, price in zip(["A1", "B2"], [100, 200]):
    print(sku, price)
```

Structural pattern matching (Python 3.10+):

```python
def describe_status(status: str) -> str:
    match status:
        case "ACTIVE":
            return "Published"
        case "DRAFT":
            return "Under preparation"
        case "ARCHIVED":
            return "Archived"
        case _:
            return "Unknown"
```

## 8. Functions and argument behavior

### 8.1 Defining functions

```python
def calculate_total(unit_price: float, quantity: int) -> float:
    return unit_price * quantity

total = calculate_total(250.0, 4)
```

Type annotations are documentation and are used by static analysis tools; Python does not enforce them automatically at runtime.

### 8.2 Default and keyword arguments

```python
def search_products(query: str, page: int = 1, page_size: int = 20):
    return {"query": query, "page": page, "page_size": page_size}

search_products("laptop", page_size=50)
```

Avoid mutable default arguments:

```python
# Incorrect: the same list is reused between calls.
def add_tag(tag, tags=[]):
    tags.append(tag)
    return tags

# Correct:
def add_tag(tag, tags=None):
    if tags is None:
        tags = []
    tags.append(tag)
    return tags
```

### 8.3 `*args` and `**kwargs`

```python
def log_event(event_name, *values, **metadata):
    print(event_name, values, metadata)

log_event("PRODUCT_UPDATED", 101, "LAP-101", source="admin", retry=1)
```

- `*args` collects extra positional arguments into a tuple.
- `**kwargs` collects extra keyword arguments into a dictionary.

### 8.4 Lambda functions

```python
products = [
    {"name": "Mouse", "price": 1000},
    {"name": "Laptop", "price": 80000}
]

products.sort(key=lambda product: product["price"])
```

Use named functions when behavior needs a meaningful name or becomes complex.

## 9. Scope, namespaces, and closures

Python resolves names using the LEGB rule: Local, Enclosing, Global, Built-in.

```python
tax_rate = 0.18

def calculate_tax(amount):
    return amount * tax_rate
```

A closure retains access to variables from its enclosing function:

```python
def make_counter():
    count = 0

    def next_value():
        nonlocal count
        count += 1
        return count

    return next_value

counter = make_counter()
print(counter())  # 1
print(counter())  # 2
```

Use `global` only when genuinely necessary; passing dependencies explicitly often makes code easier to test.

## 10. Object-oriented programming

### 10.1 Classes and objects

```python
class Product:
    def __init__(self, product_id: int, name: str, price: float):
        self.product_id = product_id
        self.name = name
        self.price = price

    def apply_discount(self, percent: float) -> None:
        if not 0 <= percent <= 100:
            raise ValueError("Discount must be between 0 and 100")
        self.price *= 1 - percent / 100


product = Product(101, "Laptop", 80000)
product.apply_discount(10)
```

`self` is the conventional name for the instance parameter. It is not a reserved keyword.

### 10.2 Inheritance and composition

```python
class DigitalProduct(Product):
    def __init__(self, product_id, name, price, download_url):
        super().__init__(product_id, name, price)
        self.download_url = download_url
```

Inheritance models an “is-a” relationship. Composition models a “has-a” relationship and is often more flexible for enterprise systems.

### 10.3 Properties

```python
class InventoryItem:
    def __init__(self, quantity: int):
        self._quantity = quantity

    @property
    def quantity(self) -> int:
        return self._quantity

    @quantity.setter
    def quantity(self, value: int) -> None:
        if value < 0:
            raise ValueError("Quantity cannot be negative")
        self._quantity = value
```

### 10.4 Class methods and static methods

```python
class ProductCode:
    prefix = "SKU"

    def __init__(self, value: str):
        self.value = value

    @classmethod
    def from_number(cls, number: int):
        return cls(f"{cls.prefix}-{number}")

    @staticmethod
    def is_valid(value: str) -> bool:
        return value.startswith("SKU-")
```

### 10.5 Dataclasses

Dataclasses reduce boilerplate for data-focused classes:

```python
from dataclasses import dataclass
from decimal import Decimal

@dataclass
class ProductPrice:
    sku: str
    amount: Decimal
    currency: str = "INR"

price = ProductPrice("LAP-101", Decimal("85000.00"))
```

Use `frozen=True` for a frozen dataclass when immutable-style instances are desired. Dataclasses do not automatically validate all business rules.

## 11. Exceptions and resource management

### 11.1 Exception handling

```python
def get_product(product_id: int):
    if product_id <= 0:
        raise ValueError("Product ID must be positive")
    return {"id": product_id, "name": "Laptop"}

try:
    product = get_product(-1)
except ValueError as exc:
    print(f"Invalid request: {exc}")
else:
    print(product)
finally:
    print("Operation finished")
```

Catch specific exceptions. Do not silently swallow errors with a broad `except Exception` unless you log, translate, or deliberately handle them.

### 11.2 Custom exceptions

```python
class ProductNotFoundError(Exception):
    pass

def find_product(product_id: int):
    product = None
    if product is None:
        raise ProductNotFoundError(f"Product {product_id} was not found")
```

### 11.3 Context managers

Use `with` to ensure resources are cleaned up:

```python
with open("products.txt", "r", encoding="utf-8") as file:
    content = file.read()
```

Context managers are commonly used for files, locks, database sessions, and transactions.

## 12. Iterators and generators

An iterable can produce an iterator. An iterator provides values one at a time. A generator is a convenient way to create an iterator using `yield`.

```python
def read_product_ids(limit: int):
    for product_id in range(1, limit + 1):
        yield product_id

for product_id in read_product_ids(3):
    print(product_id)
```

Generators are useful for streaming large files or processing records without loading the entire dataset into memory.

```python
def active_products(records):
    for record in records:
        if record.get("status") == "ACTIVE":
            yield record
```

A generator is generally consumed as it is iterated. Create a new generator when you need to iterate again.

## 13. Modules, packages, and imports

A module is a Python file. A package groups modules.

```text
app/
  __init__.py
  main.py
  api/
    __init__.py
    products.py
  services/
    __init__.py
    product_service.py
  repositories/
    __init__.py
    product_repository.py
  schemas/
    __init__.py
    product_schema.py
```

Example imports:

```python
from app.services.product_service import ProductService
```

Avoid circular imports by keeping dependencies flowing in a clear direction. Put shared contracts in a suitable lower-level module rather than importing service modules into each other.

## 14. Type hints and modern Python typing

```python
from typing import Optional

def find_name(product_id: int) -> Optional[str]:
    if product_id == 1:
        return "Laptop"
    return None
```

Modern syntax (Python 3.10+):

```python
def find_name(product_id: int) -> str | None:
    ...
```

Collections:

```python
def total_prices(prices: list[float]) -> float:
    return sum(prices)

def product_index(products: list[dict[str, object]]) -> dict[int, dict[str, object]]:
    return {int(product["id"]): product for product in products}
```

Use `TypedDict`, dataclasses, or Pydantic models when a structured dictionary or validated data model is needed.

Generic type example:

```python
from typing import TypeVar

T = TypeVar("T")

def first_or_none(items: list[T]) -> T | None:
    return items[0] if items else None
```

Static checking tools such as mypy and Pyright can detect many type errors before runtime.

## 15. Working with files, JSON, CSV, and dates

### 15.1 JSON

```python
import json

product = {"id": 101, "name": "Laptop", "active": True}
json_text = json.dumps(product)
restored = json.loads(json_text)
```

### 15.2 CSV

```python
import csv

with open("products.csv", "r", newline="", encoding="utf-8") as file:
    reader = csv.DictReader(file)
    for row in reader:
        print(row["sku"], row["name"])
```

For large imports, process rows incrementally and validate each row. Record rejected rows with a reason instead of silently dropping them.

### 15.3 Dates and time zones

```python
from datetime import datetime, timezone

now_utc = datetime.now(timezone.utc)
print(now_utc.isoformat())
```

Use timezone-aware datetimes for timestamps representing real instants. Store and exchange timestamps in UTC where practical, then convert for display.

## 16. Concurrency and asynchronous programming

### 16.1 Threads

Threads are useful for I/O-bound work such as network calls and file operations.

```python
from concurrent.futures import ThreadPoolExecutor

def check_supplier(supplier_id: int) -> str:
    return f"Checked {supplier_id}"

with ThreadPoolExecutor(max_workers=5) as executor:
    results = list(executor.map(check_supplier, [1, 2, 3, 4]))
```

In standard GIL-enabled CPython builds, the Global Interpreter Lock affects parallel execution of Python bytecode across threads. Threads can still improve I/O concurrency. Python versions/builds are evolving, so check the runtime configuration when CPU parallelism matters.

### 16.2 Processes

Processes can run CPU-bound work in parallel using separate interpreter processes:

```python
from concurrent.futures import ProcessPoolExecutor

def calculate(value: int) -> int:
    return value * value

if __name__ == "__main__":
    with ProcessPoolExecutor() as executor:
        results = list(executor.map(calculate, range(10)))
```

The `if __name__ == "__main__"` guard is important for portable multiprocessing startup behavior.

### 16.3 Asyncio

Asyncio supports concurrent I/O using an event loop and coroutines.

```python
import asyncio

async def fetch_supplier(supplier_id: int) -> str:
    await asyncio.sleep(0.1)  # represents asynchronous I/O
    return f"Supplier {supplier_id}"

async def main():
    results = await asyncio.gather(
        fetch_supplier(1),
        fetch_supplier(2),
        fetch_supplier(3)
    )
    print(results)

asyncio.run(main())
```

Do not call blocking I/O or long CPU work directly on the event loop. Use async-compatible libraries or move blocking/CPU work to an executor or worker process.

## 17. Database access and SQLAlchemy concepts

In enterprise applications, separate API, service, and repository responsibilities. SQLAlchemy is a common Python ORM and SQL toolkit.

Conceptual SQLAlchemy 2.x model:

```python
from sqlalchemy import String, Numeric
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class ProductEntity(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    sku: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(200))
    price: Mapped[float] = mapped_column(Numeric(12, 2))
```

Important concepts:
- Engine and connection pool.
- Session lifecycle and transaction boundaries.
- Commit and rollback.
- Parameterized queries.
- Migrations (commonly Alembic).
- Avoiding N+1 query patterns.
- Explicit session cleanup.

Never concatenate untrusted values into SQL. Use ORM expressions or bound parameters.

## 18. Build a REST API with FastAPI

FastAPI is a Python framework for building APIs using type hints and ASGI.

Install:

```bash
python -m pip install fastapi uvicorn
```

Minimal API:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

app = FastAPI(title="Product Data Service")

class ProductCreate(BaseModel):
    sku: str = Field(min_length=2, max_length=50)
    name: str = Field(min_length=2, max_length=200)
    price: float = Field(gt=0)

class ProductResponse(ProductCreate):
    id: int
    status: str

products: dict[int, ProductResponse] = {}
next_id = 1

@app.get("/api/products", response_model=list[ProductResponse])
def list_products():
    return list(products.values())

@app.get("/api/products/{product_id}", response_model=ProductResponse)
def get_product(product_id: int):
    product = products.get(product_id)
    if product is None:
        raise HTTPException(status_code=404, detail="Product not found")
    return product

@app.post("/api/products", response_model=ProductResponse, status_code=201)
def create_product(request: ProductCreate):
    global next_id
    product = ProductResponse(
        id=next_id,
        status="DRAFT",
        **request.model_dump()
    )
    products[next_id] = product
    next_id += 1
    return product
```

Run locally:

```bash
uvicorn app.main:app --reload
```

Open `/docs` on the local server to explore the generated OpenAPI documentation.

This in-memory example is for learning only. A production service needs a database, concurrency-safe persistence, authentication/authorization, structured logging, configuration, migrations, and tests.

## 19. Pydantic validation and API contracts

Pydantic validates and parses input data at runtime.

```python
from pydantic import BaseModel, Field, ConfigDict

class ProductCreateRequest(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True, extra="forbid")

    sku: str = Field(min_length=2, max_length=50)
    name: str = Field(min_length=2, max_length=200)
    price: float = Field(gt=0)
```

Distinguish:
- Type hints: static information for developers and tools.
- Pydantic models: runtime parsing and validation.
- Database constraints: persistence-level integrity.
- Business rules: application-specific logic that may involve multiple records or services.

Do not assume a frontend TypeScript type or a Python annotation alone validates incoming HTTP data.

## 20. Testing with pytest

Install:

```bash
python -m pip install pytest
```

Example:

```python
def calculate_discount(price: float, percent: float) -> float:
    if price < 0:
        raise ValueError("Price cannot be negative")
    if not 0 <= percent <= 100:
        raise ValueError("Invalid discount")
    return price * (1 - percent / 100)
```

Test:

```python
import pytest

def test_calculate_discount():
    assert calculate_discount(100, 10) == 90

def test_invalid_discount():
    with pytest.raises(ValueError):
        calculate_discount(100, 150)
```

Testing layers:
- Unit tests for functions and business rules.
- API tests for routes, validation, and status codes.
- Integration tests for database and external services.
- Contract tests for service boundaries.
- End-to-end tests for critical workflows.

Use fixtures for reusable test setup. Mock external boundaries, but do not mock so much that tests stop checking real integration behavior.

## 21. Logging, configuration, and observability

Use the `logging` module rather than scattered `print()` statements in services.

```python
import logging

logger = logging.getLogger(__name__)

def publish_product(product_id: int) -> None:
    logger.info("Publishing product", extra={"product_id": product_id})
```

Configure logging centrally. In production, use structured logs with request/correlation IDs and avoid logging secrets or sensitive data.

Configuration should come from environment variables or a managed configuration system, not hardcoded credentials.

```python
import os

database_url = os.environ["DATABASE_URL"]
environment = os.getenv("APP_ENV", "local")
```

Operational basics:
- Health and readiness endpoints.
- Metrics for latency, throughput, and errors.
- Distributed tracing across service calls.
- Correlation IDs across Java and Python services.
- Alerts based on user impact and service-level objectives.

## 22. Packaging, dependencies, and code quality

Modern Python projects commonly use `pyproject.toml` to describe project metadata and tool configuration.

Example minimal configuration:

```toml
[project]
name = "product-data-service"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
  "fastapi",
  "uvicorn",
  "pydantic"
]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

Useful tools:
- `venv`: isolated environments.
- `pip`: package installer.
- `pytest`: testing.
- `ruff`: linting and formatting.
- `mypy` or `pyright`: static type checking.
- `pre-commit`: local quality checks.
- `build`: building distribution artifacts.
- `pip-audit`: dependency vulnerability checks.

Pin and manage dependencies deliberately for reproducible builds. For larger projects, use a lockfile workflow supported by the chosen dependency manager.

## 23. Docker deployment basics

Example `Dockerfile`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Production considerations:
- Run as a non-root user.
- Use a small trusted base image and scan it.
- Pin dependencies.
- Keep secrets out of the image.
- Configure CPU/memory limits and health checks.
- Use a production process strategy appropriate to the deployment platform.
- Send logs to the platform logging system.

## 24. Python in an enterprise Java + AI architecture

A practical division of responsibilities can be:

```text
React + TypeScript UI
        |
        v
Spring Boot application API
  - Authentication and authorization
  - Core business transactions
  - Existing enterprise integrations
        |
        +----------------------+
        |                      |
        v                      v
Relational database        Kafka events
                               |
                               v
                       Python AI service
                       - Data extraction
                       - Embeddings
                       - Retrieval / RAG
                       - Model orchestration
                       - Evaluation jobs
                               |
                               v
                       Vector database / LLM
```

This is one possible design, not a universal rule. Keep core business ownership clear and use well-defined APIs or events between services.

### Example: product enrichment workflow

1. Supplier data arrives through an upload or event.
2. Spring Boot validates core business rules and records the import job.
3. An event announces that a product needs enrichment.
4. A Python worker extracts descriptions and attributes.
5. The worker may retrieve relevant catalog standards or reference documents.
6. It returns proposed attributes with confidence and evidence.
7. The business workflow reviews or approves the proposals.
8. Approved data is persisted and published by the owning service.

AI-generated output should be treated as a proposal until validated. Include provenance, confidence, validation, audit history, and human review for consequential catalog changes.

## 25. Python for AI and data work: what to learn next

Python is widely used for AI because of its ecosystem. Learn the foundations before jumping directly into agent frameworks.

Suggested progression:
1. Python language and packaging.
2. NumPy and pandas for numerical and tabular data.
3. Basic statistics and machine learning concepts.
4. scikit-learn for classical ML workflows.
5. PyTorch for deep learning foundations.
6. Embeddings and vector search.
7. RAG pipeline design and evaluation.
8. LLM APIs, structured outputs, and tool calling.
9. LangChain or LangGraph after understanding the underlying workflow.
10. Production concerns: latency, cost, retries, observability, privacy, and evaluation.

Do not treat a framework tutorial as a substitute for understanding Python, HTTP, data structures, async execution, and testing.

## 26. Common mistakes

| Mistake | Better approach |
|---|---|
| Using mutable default arguments | Use `None` and create the object inside the function |
| Confusing `is` and `==` | Use `==` for values and `is None` for None checks |
| Catching every exception silently | Catch specific exceptions and log or propagate appropriately |
| Using global mutable state | Pass dependencies explicitly or encapsulate state |
| Loading huge files into memory | Stream records with iterators/generators |
| Blocking an async event loop | Use async libraries or offload blocking work |
| Assuming type hints validate data | Use runtime validation at trust boundaries |
| Using float for exact money | Use `Decimal` |
| Hardcoding credentials | Use environment/configuration and secret management |
| Sharing one database session across concurrent tasks | Follow the ORM's session/thread/task safety rules |
| Skipping tests because Python is dynamic | Add unit, integration, and contract tests |

## 27. Interview questions

### Python fundamentals
1. What are Python's main built-in data types?
2. Difference between list and tuple?
3. Difference between set and dictionary?
4. Difference between mutable and immutable objects?
5. Difference between `is` and `==`?
6. Explain shallow copy and deep copy.
7. What is list comprehension?
8. Explain LEGB scope resolution.
9. What are `*args` and `**kwargs`?
10. Why should mutable default arguments be avoided?

### OOP and language features
11. What is the purpose of `self`?
12. Difference between instance, class, and static methods?
13. What is inheritance? When is composition better?
14. What is a decorator?
15. What is a generator?
16. Difference between iterable and iterator?
17. How does a context manager work?
18. What are dataclasses?

### Runtime and concurrency
19. What is the Python GIL?
20. When would you use threads vs processes?
21. What is asyncio?
22. What happens when you `await` a coroutine?
23. Why should blocking operations be avoided in an async endpoint?

### Backend and enterprise
24. What is FastAPI and how does it use type hints?
25. Difference between Pydantic validation and static type checking?
26. How do you manage database transactions?
27. How do you handle exceptions consistently in REST APIs?
28. How do you test FastAPI endpoints?
29. How do you manage configuration and secrets?
30. How would you integrate a Python AI service with Spring Boot and Kafka?

## 28. Hands-on exercises

1. Build a function to calculate product price after discount.
2. Remove duplicate SKUs from a list while preserving order.
3. Count products by status using a dictionary.
4. Group products by category.
5. Sort products by price without changing the original list.
6. Read a large CSV file one row at a time.
7. Validate product data and collect row-level errors.
8. Create a `Product` class with validation.
9. Convert a class to a dataclass.
10. Write a generator that yields batches of records.
11. Create custom exceptions for invalid and missing products.
12. Build a typed service/repository structure.
13. Create FastAPI endpoints for product CRUD.
14. Add Pydantic validation and consistent error responses.
15. Connect the API to PostgreSQL using SQLAlchemy.
16. Add database migrations with Alembic.
17. Write pytest unit and API tests.
18. Add structured logging and correlation IDs.
19. Process independent I/O calls concurrently.
20. Build a worker that consumes a product-enrichment event and returns a validated proposal.

## 29. 30-day learning plan

| Days | Focus | Deliverable |
|---|---|---|
| 1–3 | Syntax, variables, types, strings, operators | Small scripts |
| 4–6 | Collections, comprehensions, control flow | Product data transformations |
| 7–9 | Functions, scope, closures, exceptions | Reusable utility module |
| 10–12 | OOP, dataclasses, modules, typing | Product domain model |
| 13–15 | Files, JSON, CSV, dates | Supplier CSV import utility |
| 16–18 | Iterators, generators, threads, processes, asyncio | Batch processing exercise |
| 19–21 | FastAPI, Pydantic, REST design | Product CRUD API |
| 22–24 | SQLAlchemy, PostgreSQL, migrations | Persistent product API |
| 25–27 | pytest, logging, configuration, Docker | Tested containerized service |
| 28–30 | Java integration and AI workflow design | Product enrichment service prototype |

## 30. Capstone: Product Data Import and Enrichment Service

Build a Python service that accepts supplier product data and produces validated product records.

**Core features**
- Upload and parse CSV/JSON product data.
- Validate SKU, name, price, category, and required attributes.
- Generate row-level validation errors.
- Persist valid records through a repository layer.
- Expose REST endpoints using FastAPI.
- Add pagination and filtering.
- Publish or consume events through Kafka as a later enhancement.
- Add a Python AI enrichment worker as a separate module or service.
- Include tests, structured logs, Docker packaging, and configuration.

**Suggested milestones**
1. Implement parsing and validation as pure Python functions.
2. Add typed domain models and tests.
3. Expose endpoints through FastAPI.
4. Add database persistence and migrations.
5. Add batch processing and retry/error handling.
6. Integrate with a Spring Boot service through REST or Kafka.
7. Add AI enrichment behind a clear service boundary.
8. Add monitoring, deployment, and operational documentation.

## 31. Recommended mastery order for a Java developer

1. Python syntax and collections — become comfortable writing idiomatic code.
2. Functions, OOP, modules, and exceptions — build maintainable application structure.
3. Type hints, dataclasses, and Pydantic — make contracts explicit.
4. Iterators, generators, and asyncio — understand efficient data and I/O processing.
5. FastAPI and SQLAlchemy — build production-style backend services.
6. pytest, logging, Docker, and CI — deliver and operate the service.
7. NumPy, pandas, ML, embeddings, and RAG — move into AI engineering.

**Practical target:** Build one tested FastAPI service that accepts supplier product data, validates it, stores it, and integrates with a Spring Boot application. Then extend it with an AI enrichment workflow.
