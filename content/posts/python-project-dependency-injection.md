---
title: "Python Project Dependency Injection: Best Practices with python-inject"
date: 2026-09-09T21:55:00+03:00
tags: ["Python", "Dependency Injection", "FastAPI", "Typer", "Architecture", "SOLID"]
draft: false
---

If you struggle to follow SOLID principles in Python projects—especially the **Dependency Inversion Principle (DIP)**—it is time to embrace Dependency Injection (DI). In enterprise software engineering, DI is an established industry standard. Frameworks in almost every major ecosystem make DI a first-class citizen: **Spring** in Java, **Symfony** in PHP, **Fx** in Go, **Angular** in TypeScript/JavaScript, and the native DI container in **.NET (C#)**. Yet in the Python community, DI is often met with skepticism or regarded as "un-Pythonic." Because Python is dynamic and allows monkeypatching, many developers assume they do not need DI—until their codebase grows, circular imports emerge, and testing becomes a tangled web of fragile mocks.

Over the past six years, I have worked extensively with medium- and large-scale Python projects powered by **FastAPI** and the lightweight DI library [`inject`](https://github.com/ivankorobkov/python-inject). While `python-inject` is minimalist and high-performing, the library documentation alone does not explain how to structure a real-world enterprise project. By bringing battle-tested patterns from mature ecosystems like Symfony and Spring into Python, I was able to answer recurring questions from teammates on how to organize modules, decouple services, and write genuine object-oriented code. A solid grasp of Dependency Injection is also the cornerstone of understanding Robert C. Martin's **Package Principles** (Cohesion and Coupling) and structuring maintainable Python applications.

When developers begin structuring a new Python project, guidance is surprisingly sparse. The [Twelve-Factor App](https://12factor.net/) methodology provides great cloud-native operational principles, but it is too generic to guide internal application and package design. Even ubiquitous web frameworks like Django lack built-in Dependency Injection, and their documentation offers little insight into clean OOP package boundaries. Proper module and package design strictly depends on SOLID and Package Principles (such as the Common Closure Principle and Stable Dependencies Principle). The single biggest obstacle to adhering to these principles in Python is attempting to do so without Dependency Injection.

Let’s examine how this architectural dilemma presents itself, how DI solves it, and the best practices for integrating `python-inject` into your applications.

---

## The Problem: How Dependencies Creep into Constructors

In many Python projects, services instantiate their own dependencies directly inside `__init__`:

```python
class UserService:
    def __init__(self):
        self.db = PostgresDatabase()
        self.email = EmailClient()
```

### What is wrong with this approach?

1. **Tight Coupling:** `UserService` is tightly bound to `PostgresDatabase` and `EmailClient`. It cannot work with MySQL, SQLite, or an in-memory database without modifying `UserService` itself.
2. **Violation of the Dependency Inversion Principle:** High-level policy (`UserService`) directly depends on low-level implementation details (`PostgresDatabase`, `EmailClient`), rather than depending on abstractions or interfaces.
3. **Untestable Code:** You cannot unit test `UserService` in isolation without connecting to a real PostgreSQL database and SMTP server, unless you resort to messy runtime monkeypatching (`unittest.mock.patch`).
4. **Hidden Side Effects:** Instantiating a class triggers hidden side effects (opening connection pools, reading environment variables, connecting to remote sockets), making the instantiation order fragile.

---

## Step 1: Constructor Injection (Manual Inversion of Control)

The textbook fix is **Inversion of Control (IoC)** via constructor injection:

```python
from typing import Protocol

class Database(Protocol):
    def fetch_user(self, user_id: int): ...

class EmailClient(Protocol):
    def send(self, recipient: str, subject: str, body: str): ...

class UserService:
    def __init__(self, db: Database, email: EmailClient):
        self.db = db
        self.email = email
```

`UserService` no longer cares *how* `Database` or `EmailClient` are constructed. It only cares that the passed objects fulfill the contract.

### The New Dilemma: Instantiation Boilerplate

While the class is now clean and testable, consumer code must now wire everything up manually:

```python
service = UserService(
    db=PostgresDatabase(settings.DB_URL),
    email=EmailClient(settings.SMTP_HOST),
)
```

If you have 20 endpoints, 5 CLI commands, and 3 background tasks that need `UserService`, you must duplicate this wiring snippet across 28 call sites.

### The Factory Pattern and Its Limits

A natural step is to extract this construction logic into a reusable factory:

```python
def create_user_service() -> UserService:
    return UserService(
        db=PostgresDatabase(settings.DB_URL),
        email=EmailClient(settings.SMTP_HOST),
    )
```

While factories help reduce duplication, they quickly become unwieldy as your dependency graph grows. If `PostgresDatabase` depends on a connection pool, which depends on a configuration manager, which depends on secrets management, your factories become deeply nested, manual dependency trees.

Can we encapsulate class instantiation and lifecycle management into a reusable engine? This is where **Dependency Injection containers** step in.

---

## Introducing `python-inject`

[`python-inject`](https://github.com/ivankorobkov/python-inject) is a lightweight, thread-safe DI library for Python. Unlike heavy frameworks that wrap your entire runtime, `python-inject` provides a clean binding API and automatic parameter resolution (**autoparams**).

Having contributed to `python-inject` to advance its autowiring and binding capabilities, I have seen firsthand how it drastically simplifies project architecture when used properly.

However, using a DI library without architectural guidelines can lead to new anti-patterns. Below are the key best practices collected from production systems.

---

## Best Practice 1: Keep Container Configuration Centralized in `di.py` (Composition Root)

Always configure your DI container in a single, dedicated module (typically named `di.py` or `core/di.py`). This serves as the **Composition Root** of your application—the single location where the entire object graph is composed.

In `di.py`, use `@inject.autoparams` **only** on provider functions that construct your dependencies. This keeps your actual domain and service classes free of framework decorators, while allowing the container to automatically resolve constructor arguments for provider functions.

```python
# app/di.py
import inject

from app.infrastructure.db import Database, PostgresDatabase
from app.infrastructure.email import EmailClient, SmtpEmailClient
from app.services.user_service import UserService
from app.core.config import Settings, get_settings


@inject.autoparams()
def provide_database(settings: Settings) -> Database:
    return PostgresDatabase(settings.DB_URL)


@inject.autoparams()
def provide_email_client(settings: Settings) -> EmailClient:
    return SmtpEmailClient(settings.SMTP_HOST)


@inject.autoparams()
def provide_user_service(db: Database, email: EmailClient) -> UserService:
    return UserService(db=db, email=email)


def configure(binder: inject.Binder) -> None:
    # 1. Bind configuration / value objects
    binder.bind_to_constructor(Settings, get_settings)

    # 2. Bind interfaces and services to their providers
    binder.bind_to_constructor(Database, provide_database)
    binder.bind_to_constructor(EmailClient, provide_email_client)
    binder.bind_to_constructor(UserService, provide_user_service)


# Configure container on module import
inject.configure(configure, once=True, bind_in_runtime=False)
```

### Why this design is so effective:
- **`@inject.autoparams` only on providers:** The provider functions handle wiring via type hints, while `UserService`, `PostgresDatabase`, and `SmtpEmailClient` remain pure, decoupled classes.
- **`bind_in_runtime=False`:** Disables uncontrolled, implicit runtime auto-binding for unregistered types, ensuring your container fail-fast if a binding is missing.
- **`once=True`:** Ensures that multiple imports or reloading don't raise configuration errors.
- **Self-initializing module:** By running `inject.configure(...)` at the module level and re-exporting `inject`, consumers simply import `inject` from `app.di`, guaranteeing the container is always ready:

```python
from app.di import inject

user_service = inject.instance(UserService)
```

---

## Best Practice 2: Never Use DI Decorators Inside Target Service Classes

`python-inject` provides decorators like `@inject.autoparams()` and `@inject.params()`. A common mistake is decorating domain services directly:

```python
# ❌ ANTI-PATTERN: Polluting domain logic with DI decorators
import inject

class UserService:
    @inject.autoparams()  # DO NOT DO THIS
    def __init__(self, db: Database, email: EmailClient):
        self.db = db
        self.email = email
```

### Why avoid decorators on service classes?
1. **Framework Decoupling:** Your business logic should be composed of **POPOs (Plain Old Python Objects)**. If you ever switch DI libraries or run code in a standalone script, your domain classes shouldn't fail because `inject` is imported.
2. **Seamless Unit Testing:** A pure constructor allows you to instantiate `UserService(mock_db, mock_email)` in tests without initializing or resetting any DI container.
3. **Explicit Contracts:** Keep constructors standard and predictable. Let the container provider functions in `di.py` handle wiring.

---

## Best Practice 3: Understanding the Service Locator Pattern and Its Boundaries

In `python-inject`, you request an instance using `inject.instance(Cls)`:

```python
from app.di import inject

user_service = inject.instance(UserService)
```

From an architectural standpoint, calling `inject.instance()` directly inside arbitrary business logic is the **Service Locator pattern**, which is widely considered an anti-pattern. Why? Because it conceals dependencies: instead of looking at a class signature to see what it requires, dependencies are fetched behind the scenes via a global registry.

### Why do Symfony and Angular avoid this?
In **Symfony (PHP)** and **Angular (TypeScript)**, the framework manages the entire component lifecycle. Symfony's compiled container inspects controller constructor arguments and automatically injects them during request dispatching. Angular's dependency injector resolves tokens declared in component constructors. Neither requires you to write `container.get(Service)` inside standard application code.

### The Python Reality
Python does not have a global runtime compiler or an omnipresent kernel managing all class instantiations. Neither the Python interpreter nor ASGI frameworks automatically resolve full constructor graphs for arbitrary classes.

Because `python-inject` is non-intrusive, calling `inject.instance()` at the boundary is practically unavoidable.

### The Rule: Confine `inject.instance` Strictly to Entry Points
The golden rule is: **Never call `inject.instance()` inside services or domain models.** Restrict `inject.instance()` solely to the **edges/entry points** of your application:
- HTTP route controllers (e.g., FastAPI endpoints)
- CLI command handlers (e.g., Typer commands)
- Message queue workers / task consumers (e.g., Celery, RQ)

---

## Best Practice 4: FastAPI Integration

In FastAPI, routes act as the entry point into your application layer. Always import `inject` from `app.di` to ensure the container is initialized:

### Option A: Directly in Route Handlers

```python
# app/api/routers/users.py
from fastapi import APIRouter, HTTPException, status
from app.di import inject
from app.services.user_service import UserService
from app.api.schemas import UserCreate, UserResponse

router = APIRouter(prefix="/users", tags=["Users"])

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
def create_user(payload: UserCreate):
    # Retrieve service at the entry point boundary
    service = inject.instance(UserService)
    return service.register_user(payload.email, payload.password)
```

### Option B: Bridging with FastAPI's `Depends`

If you want to maintain FastAPI's idiomatic dependency declarations in OpenAPI docs and support FastAPI's native `app.dependency_overrides`, write a thin provider helper:

```python
# app/api/dependencies.py
from fastapi import Depends
from app.di import inject
from app.services.user_service import UserService

def get_user_service() -> UserService:
    return inject.instance(UserService)

# app/api/routers/users.py
@router.post("/", response_model=UserResponse)
def create_user(payload: UserCreate, service: UserService = Depends(get_user_service)):
    return service.register_user(payload.email, payload.password)
```

Both patterns keep `UserService` completely unaware of FastAPI, HTTP headers, request objects, or status codes.

---

## Best Practice 5: CLI Integration with Typer

A major strength of this architecture is that your domain services can be reused across completely different entry points without modification.

Consider a CLI tool built with [Typer](https://typer.tiangolo.com/):

```python
# app/cli/commands.py
import typer
from app.di import inject
from app.services.user_service import UserService

cli = typer.Typer()

@cli.command()
def activate_user(user_id: int):
    """CLI entry point: resolve UserService and perform the action."""
    service = inject.instance(UserService)
    service.activate(user_id)
    typer.echo(f"User {user_id} successfully activated.")

@cli.command()
def sync_users():
    """Bulk synchronization CLI command."""
    service = inject.instance(UserService)
    count = service.sync_all()
    typer.echo(f"Synchronized {count} users.")
```

Notice how clean the command handler is. Importing `from app.di import inject` automatically guarantees container configuration, and commands simply request their top-level service from the container.

---

## Best Practice 6: Multiple Instances of the Same Class and Custom Scalars with `typing.NewType`

Because Python DI containers map dependencies by type annotations, a common challenge arises when your application requires **multiple instances of the exact same class** or needs to inject **primitive scalar values**.

### The Problem: Ambiguous Type Keys
Consider an application using **SQLAlchemy** with a master-replica setup:
- A primary database engine for writes (`create_engine(...)`)
- A read-only replica engine for heavy queries and reports (`create_engine(...)`)

Both instances share the same type: `sqlalchemy.Engine`. Similarly, if you want to inject primitive scalar configuration parameters—such as a database URL (`str`), an API secret key (`str`), or a cache timeout in seconds (`int`)—relying on raw built-in types like `str` or `int` makes it impossible for `inject` to differentiate between them.

Falling back to magic string keys (e.g., `@inject.params(db="replica_db")`) sacrifices type safety, breaks IDE autocomplete, and easily leads to runtime errors.

### The Solution: `typing.NewType`
The Python standard library provides `typing.NewType`. It creates distinct, semantic types with zero runtime overhead while providing a unique identity token for static type checkers (Mypy, Pyright) and runtime DI containers alike.

```python
from typing import NewType
from sqlalchemy import Engine

# Distinct types for multiple instances of SQLAlchemy Engine
PrimaryEngine = NewType("PrimaryEngine", Engine)
ReplicaEngine = NewType("ReplicaEngine", Engine)

# Distinct types for scalar configuration parameters
PrimaryDbUrl = NewType("PrimaryDbUrl", str)
ReplicaDbUrl = NewType("ReplicaDbUrl", str)
CacheTimeoutSeconds = NewType("CacheTimeoutSeconds", int)
```

### Wiring Multiple Instances in `di.py`

Here is how you cleanly wire multiple SQLAlchemy engines and custom scalars in `di.py`:

```python
# app/di.py
from typing import NewType
from sqlalchemy import Engine, create_engine
import inject

from app.core.config import Settings, get_settings

# 1. Define unique types
PrimaryEngine = NewType("PrimaryEngine", Engine)
ReplicaEngine = NewType("ReplicaEngine", Engine)


# 2. Define providers for each specific instance
@inject.autoparams()
def provide_primary_engine(settings: Settings) -> PrimaryEngine:
    engine = create_engine(settings.PRIMARY_DB_URL, pool_size=10, pool_pre_ping=True)
    return PrimaryEngine(engine)


@inject.autoparams()
def provide_replica_engine(settings: Settings) -> ReplicaEngine:
    engine = create_engine(settings.REPLICA_DB_URL, pool_size=30, pool_pre_ping=True)
    return ReplicaEngine(engine)


# 3. Target service requiring both specific connections
class AnalyticsReportService:
    def __init__(self, write_engine: PrimaryEngine, read_engine: ReplicaEngine):
        self.write_engine = write_engine
        self.read_engine = read_engine

    def generate_report(self) -> dict:
        # Read from replica, write results to primary
        ...


@inject.autoparams()
def provide_analytics_service(
    primary: PrimaryEngine,
    replica: ReplicaEngine,
) -> AnalyticsReportService:
    return AnalyticsReportService(write_engine=primary, read_engine=replica)


def configure(binder: inject.Binder) -> None:
    binder.bind_to_constructor(Settings, get_settings)
    binder.bind_to_constructor(PrimaryEngine, provide_primary_engine)
    binder.bind_to_constructor(ReplicaEngine, provide_replica_engine)
    binder.bind_to_constructor(AnalyticsReportService, provide_analytics_service)


inject.configure(configure, once=True, bind_in_runtime=False)
```

### Why this is a game changer:
1. **Zero Runtime Overhead:** At runtime, `NewType(name, BaseClass)` behaves essentially like `BaseClass` (the helper returns identity at runtime).
2. **Complete Static Type Safety:** If a developer inadvertently passes a `ReplicaEngine` where a `PrimaryEngine` is expected, Mypy and IDEs will immediately flag a type mismatch before the code ever runs.
3. **Flawless Autoparams Resolution:** Because `PrimaryEngine` and `ReplicaEngine` have distinct identities (`id(PrimaryEngine) != id(ReplicaEngine)`), `@inject.autoparams()` unambiguously identifies and injects the exact intended database instance.

---

## Best Practice 7: Frictionless Testing

The real reward of following these practices comes when writing tests.

### 1. Pure Unit Tests (Zero DI Overhead)
Because services use pure constructor injection without decorators, your unit tests do not need `inject` at all:

```python
# tests/unit/test_user_service.py
from unittest.mock import MagicMock
from app.services.user_service import UserService
from app.infrastructure.db import Database
from app.infrastructure.email import EmailClient

def test_register_user_sends_email():
    mock_db = MagicMock(spec=Database)
    mock_email = MagicMock(spec=EmailClient)
    
    # Pure instantiation with mock doubles
    service = UserService(db=mock_db, email=mock_email)
    service.register_user("test@example.com", "password123")
    
    mock_db.create_user.assert_called_once()
    mock_email.send.assert_called_once_with(
        "test@example.com", "Welcome!", "Thank you for signing up."
    )
```

### 2. Integration / E2E Tests (Rebinding the Container)
For end-to-end or integration tests where FastAPI routes or Typer commands are executed, you can rebind dependencies to in-memory test doubles using `inject.clear_and_configure()`:

```python
# tests/conftest.py
import pytest
import inject
from app.infrastructure.db import Database
from app.infrastructure.email import EmailClient
from tests.doubles import InMemoryDatabase, FakeEmailClient

@pytest.fixture(autouse=True)
def configure_test_di():
    def test_bindings(binder: inject.Binder):
        binder.bind(Database, InMemoryDatabase())
        binder.bind(EmailClient, FakeEmailClient())
    
    # Reset and configure test doubles
    inject.clear_and_configure(test_bindings)
    yield
    inject.clear()
```

---

## When You Do NOT Need Dependency Injection

Dependency Injection is indispensable for managing complex, stateful, or infrastructural object graphs in **end applications**—wiring business services, repositories, API clients, and use case handlers. However, applying DI indiscriminately across every layer of your codebase is an anti-pattern. 

There are two major areas where you should **never** use Dependency Injection:

### 1. Data-Carrying Objects: Models, Entities, and DTOs
In modern Python architectures (especially when using FastAPI with Pydantic and ORMs like SQLAlchemy, SQLModel, or Tortoise), you frequently work with Pydantic schemas, DTOs (Data Transfer Objects), and database models.
- **Instantiation is the Framework's Responsibility:** The lifecycle of these objects belongs entirely to the ORM or the API framework. When FastAPI deserializes an incoming HTTP JSON payload into a `UserCreate` schema, or SQLAlchemy hydrates rows from a database query into `User` models, the framework handles the instantiation. Attempting to shoehorn a DI container into this pipeline is unnecessary and fights the framework.
- **Data vs. Business Logic:** Models and DTOs represent state and data contracts, not operations. They **must not** depend on business logic classes, repositories, or external services. If your `User` model or Pydantic schema needs to inject an `EmailClient` or `UserService`, you have an architectural code smell. Keep data structures pure, and let higher-level domain services orchestrate the business operations.

### 2. Libraries and Reusable Packages (Leaf Nodes)
Another critical boundary: **never embed or enforce a DI container inside a reusable library.**
- **Libraries are Leaves in the Dependency Graph:** In an application's architecture, third-party libraries sit at the bottom of the graph as leaf nodes. They are building blocks that provide agnostic classes, functions, and utilities.
- **Preserve Developer Control:** Application developers must be free to compose their own dependency graph. If a library brings its own internal DI container, it imposes rigid assumptions about object lifecycles, bloats the package with unwanted transitive dependencies, and rarely aligns with how the consuming application is structured.
- **Avoid Framework Lock-In:** If a library depends on a specific DI framework (such as `inject`, `dependency-injector`, or `injector`), developers using a different DI tool—or no DI tool at all—are either blocked or forced into ugly workarounds. DI is fundamentally application-specific.
- **The Solution: Constructors and Factories:** Reusable libraries should expose standard constructors with sensible defaults, accompanied by factory functions or builder patterns where complex setup is required. These factory methods can then be trivially plugged into the application's own DI configuration in `di.py`.

### The Architectural Exception: Framework Integration Glue (e.g., Symfony Bundles)
Is there an exception to the "no DI in libraries" rule? Yes: **framework integration plugins**.

A prime example is found in the PHP ecosystem with **Symfony Bundles** (and similarly, Spring Boot Auto-Configurations or Django apps). In Symfony, a Bundle often serves as a dedicated adapter that bridges an underlying, DI-agnostic library with Symfony's service container. The core library itself remains completely free of any container logic. The bundle merely provides the container extensions, configuration schemas, and service definitions necessary to wire the library seamlessly into the host application's dependency injection ecosystem.

---

## Conclusion

Dependency Injection is not an anti-pattern in Python—it is an underutilized superpower. When building serious applications with FastAPI, Typer, or any modern Python framework:

1. **Centralize bindings** in a single `di.py` module (Composition Root).
2. **Keep domain services pure POPOs**; avoid framework decorators like `@inject.autoparams()` in business logic.
3. **Confine `inject.instance()`** strictly to application entry points (routers, CLI commands, queue consumers).
4. **Use `typing.NewType`** to cleanly disambiguate multiple instances of the same class (like primary vs. replica SQLAlchemy engines) or special scalars.
5. **Exclude data models, DTOs, and reusable libraries** from DI containers; reserve DI for end-application service graphs.
6. **Leverage pure constructors** for effortless unit testing, and rebind test doubles in integration fixtures.

Following these practices brings the architectural rigor of Spring, Symfony, and .NET to Python, giving you a clean, maintainable, and thoroughly testable codebase.

