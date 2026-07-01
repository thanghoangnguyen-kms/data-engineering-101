---
tags:
  - BE101
  - backend-1
  - foundations
date: 2026-06-27
status: complete
domain: "1 of 8"
track: backend
---

# B1 — Foundations & Dev Setup

**Back to:** [[Backend/00 - Backend Track Roadmap|Backend Track Roadmap]]

> [!NOTE] Domain Overview
> You'll start with a foundational understanding of how the backend works, then set up a professional Python development environment and learn the day-to-day tooling every backend engineer uses: Git, virtual environments, linters, type hints, and code quality tools.

---

## 1.0 — Backend Introduction & How the Web Works

> [!NOTE]
> A backend is the server-side of an application — the part users never see, but that does all the heavy lifting: handling requests, processing data, talking to databases, and sending responses.

**Client–Server Model**

Every web interaction follows the same pattern:
1. A **client** (browser, mobile app, another service) sends an **HTTP request**
2. The **server** (your backend) receives it, processes it, and returns an **HTTP response**

Your Python code running in FastAPI sits on the server side.

**The HTTP Request–Response Cycle**

```mermaid
sequenceDiagram
    actor Browser
    participant FastAPI as FastAPI Server
    participant DB as Database

    Browser->>+FastAPI: GET /users HTTP/1.1
    Note over FastAPI: Route matched → handler called
    FastAPI->>+DB: SELECT * FROM users
    DB-->>-FastAPI: User records
    FastAPI-->>-Browser: 200 OK + JSON body
```

Every request carries:
- **Method** — what to do (`GET`, `POST`, `PUT`, `DELETE`)
- **Path** — where to do it (`/users`, `/users/42`)
- **Headers** — metadata (content type, auth tokens)
- **Body** — data payload (for `POST`/`PUT`)

Every response carries:
- **Status code** — result (`200 OK`, `404 Not Found`, `500 Internal Server Error`)
- **Headers** — metadata about the response
- **Body** — the returned data (usually JSON)

> [!IMPORTANT] Status Code Groups
> | Range | Meaning | Examples |
> |-------|---------|---------|
> | 2xx | Success | 200 OK, 201 Created, 204 No Content |
> | 4xx | Client error | 400 Bad Request, 401 Unauthorized, 404 Not Found |
> | 5xx | Server error | 500 Internal Server Error, 503 Service Unavailable |

**Where FastAPI Fits**

FastAPI is a **web framework** — it maps incoming HTTP requests to your Python functions (routes), handles serialisation/deserialisation via Pydantic, and sends back the response. You write the logic; FastAPI handles the HTTP plumbing.

## 1.1 — Python for Backend Development

> [!NOTE]
> You don't need to be a Python expert to start. But you do need the patterns backend code actually uses: classes, error handling, and `async/await` (covered fully in [[Backend/B2 - Web & API Fundamentals|B2]]).

**Object-Oriented Patterns in Backends**

Backend code organises logic into classes called services, repositories, and models. Here's the pattern you'll see constantly:

```python
class UserService:
    def __init__(self, db: AsyncSession) -> None:
        self.db = db  # dependency injected in, not created here

    async def get_user(self, user_id: int) -> User | None:
        return await self.db.get(User, user_id)

    async def create_user(self, name: str, email: str) -> User:
        user = User(name=name, email=email)
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)
        return user
```

> [!IMPORTANT] Dependency Injection
> `UserService` receives `db` in `__init__` rather than creating it internally. This is **dependency injection** — it makes the class testable (you can pass a mock DB in tests) and decoupled. You'll use this pattern every day in FastAPI via `Depends()`.

**Error Handling**

Raise **custom exceptions** for expected failure cases. Never swallow errors silently.

```python
class UserNotFoundError(Exception):
    def __init__(self, user_id: int) -> None:
        self.user_id = user_id
        super().__init__(f"User {user_id} not found")

class DuplicateEmailError(Exception):
    pass


async def get_user_or_raise(user_id: int) -> User:
    user = await db.get(User, user_id)
    if user is None:
        raise UserNotFoundError(user_id)
    return user
```

> [!WARNING] Error Handling Anti-Patterns
> ❌ `except Exception: pass` — silently swallowing errors hides bugs
> ❌ Using generic `Exception` for all domain errors — makes caller code unreadable
> ✅ Define specific exception classes per failure type
> ✅ Let exceptions propagate to the layer that knows how to handle them

**Python Package Structure**

Python treats a folder as a **package** (importable) only when it contains an `__init__.py` file. Without it, `from app.routers import users` raises `ModuleNotFoundError`.

```
app/
├── __init__.py        ← makes 'app' a package
├── main.py
└── routers/
    ├── __init__.py    ← makes 'routers' a sub-package
    └── users.py
```

```bash
touch app/__init__.py app/routers/__init__.py   # files can be empty
```

> [!WARNING] Missing `__init__.py` = `ModuleNotFoundError`
> ❌ `from app.routers import users` with no `__init__.py` in `app/` — import fails at startup
> ✅ Add an empty `__init__.py` to every folder you intend to import from

## 1.2 — Git & Version Control

> [!NOTE]
> Git is the universal version control tool. Every professional backend project lives in a Git repository. You'll use it to track changes, collaborate via branches, and ship work through pull requests.

**Core Daily Commands**

```bash
git clone <url>           # Download a repo
git status                # See what's changed
git add .                 # Stage all changes
git commit -m "feat: add user creation endpoint"
git push                  # Upload commits to remote
git pull                  # Get latest from remote
git log --oneline         # See recent commits
```

**Feature Branch Workflow**

The golden rule: **never commit directly to `main`**. Every change goes through a feature branch and a pull request.

```mermaid
gitGraph
    commit id: "Initial project setup"
    branch feature/add-user-endpoint
    checkout feature/add-user-endpoint
    commit id: "Add User model"
    commit id: "Add POST /users route"
    commit id: "Add tests"
    checkout main
    merge feature/add-user-endpoint id: "Merge PR #1"
    branch feature/add-auth
    checkout feature/add-auth
    commit id: "Add JWT middleware"
    commit id: "Protect /users route"
    checkout main
    merge feature/add-auth id: "Merge PR #2"
```

```bash
git checkout -b feature/add-user-endpoint     # Create + switch to branch
# ... make changes, commit ...
git push -u origin feature/add-user-endpoint  # Push to GitHub
# Open Pull Request → review → merge
git checkout main && git pull                  # Update local main after merge
```

**What Never Goes in Git**

```gitignore
# Python
__pycache__/
*.pyc
.venv/

# Environment — NEVER commit secrets
.env
.env.local
.env.production

# Tool caches
.mypy_cache/
.ruff_cache/
```

> [!WARNING] Committed Secrets Are Permanent
> If you commit a `.env` file with API keys or database passwords — even once — treat those credentials as compromised. Git history is permanent. Use environment variables and `.gitignore`.

**Conventional Commits**

The format `feat: add user creation endpoint` follows the **Conventional Commits** standard — a lightweight convention that makes `git log` readable and searchable.

Format: `<type>: <short description>`

| Type | Use For |
|------|---------|
| `feat` | New feature |
| `fix` | Bug fix |
| `chore` | Maintenance (deps, config, tooling) |
| `docs` | Documentation only |
| `refactor` | Restructure with no behaviour change |
| `test` | Adding or updating tests |

```bash
git commit -m "feat: add user registration endpoint"
git commit -m "fix: handle missing user_id in DELETE /users"
git commit -m "chore: update fastapi to 0.111.0"
git commit -m "docs: add README setup instructions"
```

> [!TIP] Keep descriptions lowercase and imperative
> ✅ `feat: add login endpoint`
> ❌ `feat: Added login endpoint` / `feat: Adding login endpoint`

## 1.3 — Linux CLI Basics

> [!NOTE]
> Backend engineers live in the terminal — SSH into servers, inspect logs, run scripts, manage files. These commands are your daily toolkit.

**Navigation & File Management**

```bash
pwd                          # Print current directory
ls -la                       # List files (including hidden, with details)
cd src/                      # Change directory
mkdir -p app/models          # Create nested directories
cp .env.example .env         # Copy a file
mv old_name.py new_name.py   # Rename/move file
rm -rf __pycache__           # Delete directory recursively (careful!)
```

**File Inspection**

```bash
cat pyproject.toml            # Print entire file
head -20 app/main.py          # First 20 lines
tail -f logs/app.log          # Follow a log file in real time (Ctrl+C to stop)
grep -r "UserService" src/    # Search for text across files
wc -l app/main.py             # Count lines
```

**Processes & Permissions**

```bash
ps aux | grep uvicorn   # Find a running process
kill 12345              # Stop a process by PID
chmod +x scripts/run.sh # Make a script executable
./scripts/run.sh        # Run it
```

> [!TIP] Shell Aliases for Speed
> Add shortcuts to `~/.zshrc` (macOS) or `~/.bashrc` (Linux):
> ```bash
> alias ll="ls -la"
> alias gst="git status"
> ```
> Run `source ~/.zshrc` to apply.

## 1.4 — Virtual Environments & Dependency Management (pip, venv, uv)

> [!NOTE]
> A virtual environment isolates your project's Python packages from your system Python and from other projects. Without it, installing packages for one project can break another.

**Why Virtual Environments Exist**

```
System Python
├── Project A needs requests==2.28
└── Project B needs requests==2.31  ← conflict without isolation
```

Each venv has its own `site-packages` folder — no conflicts.

**Option A: `venv` + `pip` (built-in)**

```bash
python -m venv .venv              # Create venv
source .venv/bin/activate         # Activate (Mac/Linux)
.venv\Scripts\activate            # Activate (Windows)

pip install fastapi uvicorn       # Install packages
pip freeze > requirements.txt     # Save exact versions
deactivate                        # Leave the venv
```

**Option B: `uv` (Recommended)**

`uv` replaces `pip`, `venv`, and `pip-tools` in one tool. It's 10–100× faster than `pip`.

```bash
uv init my-project          # New project with pyproject.toml
cd my-project

uv add fastapi uvicorn             # Add runtime dependencies
uv add --dev pytest ruff mypy      # Add dev-only dependencies

uv run python main.py       # Run inside venv (no manual activate needed)
uv run pytest               # Same for any command
uv sync                     # Install all deps from pyproject.toml (for onboarding)
```

**`requirements.txt` vs `pyproject.toml`**

| | `requirements.txt` | `pyproject.toml` |
|--|--|--|
| Used by | `pip` | `uv`, `poetry`, `hatch` |
| Dev dependencies | Separate file needed | Built-in groups |
| Project metadata | ❌ | ✅ |
| Modern standard | Legacy | ✅ Preferred |

> [!WARNING] Never Install Packages Globally
> ❌ `pip install fastapi` (outside a venv) — pollutes system Python, causes conflicts
> ✅ Always activate a venv first, or use `uv run`

**Understanding `pyproject.toml`**

`uv init` generates a `pyproject.toml`. Here's what each section means:

```toml
[project]
name = "my-project"
version = "0.1.0"
description = "Backend API"
requires-python = ">=3.11"

# Runtime dependencies — uv add <pkg> appends here automatically
dependencies = [
    "fastapi>=0.111.0",
    "uvicorn>=0.29.0",
]

[project.optional-dependencies]
# Dev-only — uv add --dev <pkg> appends here
dev = [
    "pytest>=8.0.0",
    "ruff>=0.4.0",
    "mypy>=1.9.0",
]

[tool.ruff]
# ... ruff config (shown in 1.5)

[tool.mypy]
# ... mypy config (shown in 1.6)
```

> [!TIP] You rarely edit this manually
> `uv add fastapi` appends to `dependencies` automatically. `uv add --dev pytest` appends to `dev`. Direct edits are only needed to tighten version constraints.

**`.env` Files — Managing Configuration**

A `.env` file stores environment-specific values (DB URLs, secrets, feature flags) as key-value pairs. It lives in the project root and is **never committed**.

```bash
# .env
DATABASE_URL=postgresql+asyncpg://user:password@localhost/mydb
SECRET_KEY=super-secret-key-change-in-production
DEBUG=true
```

Read values in Python with `os.getenv()`:

```python
import os

database_url = os.getenv("DATABASE_URL")
secret_key = os.getenv("SECRET_KEY", "fallback-default")  # second arg = default
debug = os.getenv("DEBUG", "false").lower() == "true"
```

For development, `python-dotenv` loads `.env` automatically:

```bash
uv add --dev python-dotenv
```

```python
from dotenv import load_dotenv
load_dotenv()  # reads .env into os.environ — call this before anything else
```

Always commit a `.env.example` documenting what's required, without real values:

```bash
# .env.example  ← committed; shows what config is needed
DATABASE_URL=postgresql+asyncpg://user:password@localhost/mydb
SECRET_KEY=replace-me
DEBUG=false
```

> [!TIP] Pydantic Settings in B3
> In [[Backend/B3 - Databases & ORM|B3]] you'll use `pydantic-settings` to load and validate config with full type safety — no more raw `os.getenv()` strings.

## 1.5 — Code Quality: Linters, Formatters & Pre-commit Hooks

> [!NOTE]
> Consistent code style reduces cognitive load in code review, prevents common bugs, and keeps import order clean. `ruff` handles all of this in one tool.

**Ruff — The Modern Standard**

`ruff` replaces `flake8` (linter), `black` (formatter), and `isort` (import sorter) in a single fast tool. Configure it in `pyproject.toml`:

```toml
[tool.ruff]
target-version = "py311"
line-length = 88

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes (undefined names, unused imports)
    "I",   # isort (import ordering)
]

[tool.ruff.format]
quote-style = "double"
```

```bash
ruff check .          # Lint
ruff check --fix .    # Auto-fix fixable issues
ruff format .         # Format
```

> [!TIP] Older Codebases
> You may encounter projects using `black` (formatter) and `isort` (import sorter) separately — they predate `ruff`. They do the same job as `ruff format` and `ruff check --select I`. When joining an existing project, follow what's already configured rather than switching tools mid-project.

**Pre-commit Hooks**

Pre-commit hooks run checks automatically before each `git commit`. If a check fails, the commit is blocked.

```bash
uv add --dev pre-commit
pre-commit install          # Wire the hook into .git/
```

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

> [!EXAMPLE] Your First Commit with Pre-commit
> ```bash
> git add .
> git commit -m "feat: add user model"
> # ruff runs automatically
> # If issues found → commit blocked, files auto-fixed
> # git add . again → commit succeeds
> ```

## 1.6 — Type Hints & Static Analysis

> [!NOTE]
> Type hints tell Python (and your IDE) what types a function expects and returns. They don't change runtime behaviour — but they let `mypy` catch bugs before code runs, and make your codebase self-documenting.

**Basic Annotations**

```python
# Before: no annotations — ambiguous, IDE can't help
def create_user(name, age, roles):
    ...

# After: fully annotated — clear contract, IDE + mypy can verify callers
def create_user(name: str, age: int, roles: list[str]) -> dict[str, object]:
    ...
```

Common types you'll use constantly:

```python
user_id: int = 42
username: str = "alice"
tags: list[str] = ["admin", "staff"]
config: dict[str, str] = {"env": "production"}

# Optional (can be None) — two equivalent styles:
def find_user(user_id: int) -> str | None: ...        # Python 3.10+, preferred
def find_user(user_id: int) -> Optional[str]: ...     # older style, still valid
```

**Running `mypy`**

```bash
mypy app/           # Type-check all files
mypy app/main.py    # Type-check one file
```

Configure in `pyproject.toml`:

```toml
[tool.mypy]
python_version = "3.11"
strict = false             # Good starting point; enable strict later
ignore_missing_imports = true
```

> [!WARNING] The `Any` Trap
> ❌ Overusing `Any` defeats the purpose:
> ```python
> def process(data: Any) -> Any:  # type checker gives up entirely
>     ...
> ```
> ✅ Be specific. Reserve `Any` for genuine interop with untyped third-party libraries.

> [!TIP] IDE Integration
> VS Code with **Pylance** gives real-time type errors as you type — no need to run `mypy` manually during development. Add `mypy` to your pre-commit hooks for CI enforcement.

## 1.7 — IDE Setup

> [!NOTE]
> Your IDE is where you spend 90% of your time. A properly configured VS Code gives you real-time type errors, auto-formatting on save, and import suggestions — without running any commands manually.

**Recommended Extensions**

| Extension | VS Code ID | Purpose |
|-----------|-----------|---------|
| Python | `ms-python.python` | Language support, debugger, test runner |
| Pylance | `ms-python.vscode-pylance` | Fast type checking, autocomplete |
| Ruff | `charliermarsh.ruff` | Lint and format on save |

Install from the VS Code Extensions panel (`Cmd+Shift+X`) or via CLI:

```bash
code --install-extension ms-python.python
code --install-extension ms-python.vscode-pylance
code --install-extension charliermarsh.ruff
```

**Project Settings**

Create `.vscode/settings.json` in your project root to apply settings per-project:

```json
{
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  },
  "python.analysis.typeCheckingMode": "basic"
}
```

This makes VS Code:
- Auto-format your file every time you save (via `ruff format`)
- Auto-fix lint issues on save (via `ruff check --fix`)
- Show type errors inline (via Pylance in `basic` mode)

> [!IMPORTANT] Select the right Python interpreter
> Press `Cmd+Shift+P` → **"Python: Select Interpreter"** → choose the `.venv` inside your project folder.
> Without this, VS Code uses system Python and won't see your installed packages — all imports will show as errors.

> [!TIP] Commit `.vscode/settings.json`
> This file is project-specific config, not personal preference. Committing it ensures every team member gets the same formatter and type checker settings automatically.

---

## 1.8 — SOLID Principles

> [!NOTE] What is SOLID?
> SOLID is five design principles that make code easier to understand, test, and change. They were named by Robert C. Martin and appear constantly in professional Python codebases. You don't need to memorise the acronym — you need to recognise the problems each principle prevents. You will see all five applied across B2–B7: `Depends()` in B2, repository interfaces in B3, fake implementations in B5, and the full Clean Architecture wiring in B7.

The principles are introduced here in the order you'll encounter them in this track, not alphabetically.

---

### D — Dependency Inversion Principle

> [!IMPORTANT] The Most Important One for Backend Work
> High-level modules (your business logic) should not depend on low-level modules (SQLAlchemy, Redis, HTTP clients). Both should depend on **abstractions** — abstract interfaces that the low-level modules implement.

This is what enables FastAPI's `Depends()`, repository interfaces in B3, and the IoC container in B7. It's worth understanding first.

```python
# ❌ UserService depends on SQLAlchemy directly — can't test without a real database
from sqlalchemy.ext.asyncio import AsyncSession

class UserService:
    def __init__(self, db: AsyncSession) -> None:
        self.db = db

    async def get_user(self, user_id: int) -> User | None:
        return await self.db.get(User, user_id)  # tied to SQLAlchemy forever


# ✅ UserService depends on an abstract interface — SQLAlchemy is a swappable detail
from abc import ABC, abstractmethod

class IUserRepository(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: int) -> User | None: ...

class UserService:
    def __init__(self, user_repo: IUserRepository) -> None:
        self._repo = user_repo  # any implementation works — real DB, in-memory fake, mock

    async def get_user(self, user_id: int) -> User | None:
        return await self._repo.get_by_id(user_id)
```

In tests (B5), you pass a fake `IUserRepository`. In production, you pass the SQLAlchemy one. `UserService` never changes.

---

### S — Single Responsibility Principle

One class, one reason to change. If your `UserService` also handles authentication and sends emails, three unrelated reasons can force you to touch the same file.

```python
# ❌ One service doing three jobs — auth, user management, and email
class UserService:
    async def create_user(self, data: UserCreate) -> User: ...
    async def login(self, email: str, password: str) -> str: ...      # auth concern
    async def send_welcome_email(self, user: User) -> None: ...       # email concern


# ✅ Each service owns exactly one concern
class UserService:
    async def create_user(self, data: UserCreate) -> User: ...

class AuthService:
    async def login(self, email: str, password: str) -> str: ...

class EmailService:
    async def send_welcome_email(self, user: User) -> None: ...
```

> [!TIP]
> In Clean Architecture (B7), SRP maps directly to the layer model: controllers handle HTTP, orchestration services coordinate use cases, domain logic holds business rules, infrastructure handles persistence. One layer, one responsibility.

---

### O — Open/Closed Principle

Classes should be **open for extension** (new behaviour via new classes) but **closed for modification** (existing code untouched). Adding a new notification channel should not require editing existing code.

```python
# ❌ Every new channel means editing this function — risks breaking existing channels
def send_notification(user: User, channel: str) -> None:
    if channel == "email":
        send_email(user.email)
    elif channel == "sms":
        send_sms(user.phone)
    # adding "push" means opening this file again


# ✅ New channels are new classes — the function never changes
from abc import ABC, abstractmethod

class NotificationChannel(ABC):
    @abstractmethod
    async def send(self, user: User, message: str) -> None: ...

class EmailChannel(NotificationChannel):
    async def send(self, user: User, message: str) -> None:
        await send_email(user.email, message)

class SMSChannel(NotificationChannel):
    async def send(self, user: User, message: str) -> None:
        await send_sms(user.phone, message)

# Adding push notifications = new PushChannel class. Zero existing code touched.
```

---

### I — Interface Segregation Principle

No class should be forced to implement methods it doesn't use. Split fat interfaces into small, focused ones.

```python
# ❌ Every implementor must handle export_to_csv — even a simple in-memory fake
class IUserRepository(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: int) -> User | None: ...
    @abstractmethod
    async def save(self, user: User) -> User: ...
    @abstractmethod
    async def delete(self, user_id: int) -> None: ...
    @abstractmethod
    async def export_all_to_csv(self) -> bytes: ...  # unrelated to most use cases


# ✅ Focused interfaces — each class implements only what it needs
class IUserReader(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: int) -> User | None: ...

class IUserWriter(ABC):
    @abstractmethod
    async def save(self, user: User) -> User: ...
    @abstractmethod
    async def delete(self, user_id: int) -> None: ...
```

> [!TIP]
> You'll see this applied in B3: each repository interface is scoped to one entity. `IUserRepository` does not carry post or order operations.

---

### L — Liskov Substitution Principle

Any subtype must work wherever its parent type is expected — without breaking the caller. If a subclass raises `NotImplementedError` on a method the parent defines, it violates the contract.

```python
# ❌ ReadOnlyRepository claims to be an IUserRepository but breaks on save()
class IUserRepository(ABC):
    @abstractmethod
    async def save(self, user: User) -> User: ...

class ReadOnlyRepository(IUserRepository):
    async def save(self, user: User) -> User:
        raise NotImplementedError("This repo is read-only")  # caller doesn't know this


# ✅ Don't inherit from a contract you can't honour — use a narrower interface instead
class IUserReader(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: int) -> User | None: ...

class ReadOnlyRepository(IUserReader):  # only promises reads — no save() to violate
    async def get_by_id(self, user_id: int) -> User | None: ...
```

> [!TIP]
> LSP becomes practical in B5 (testing): your fake `InMemoryUserRepository` must honour every method signature of `IUserRepository` — otherwise the tests using it don't reflect real behaviour.

---

### SOLID at a Glance

| Principle | Rule in One Line | Where You'll See It |
|-----------|-----------------|---------------------|
| **D** — Dependency Inversion | Depend on abstractions, not concretions | `Depends()` in B2 · `IUserRepository` in B3 · IoC container in B7 |
| **S** — Single Responsibility | One class, one reason to change | Service split in B3 · Layer model in B7 |
| **O** — Open/Closed | Extend by adding classes, not editing existing ones | New channel handlers in B6 · New repository implementations in B3 |
| **I** — Interface Segregation | Small, focused interfaces over fat ones | `IUserReader` / `IUserWriter` in B3 |
| **L** — Liskov Substitution | Subtypes honour the parent contract | Fake repositories in B5 tests |

---

## 🎯 What You Learned

You can now:

- **Set up a production-grade Python dev environment** — `uv` for dependency management, `pyproject.toml` for project configuration, `.env` for secrets, and `pre-commit` to enforce quality automatically on every commit
- **Write clean, type-safe Python** — type hints on all functions, `mypy --strict` for static checking, and `Ruff` handling linting + formatting in a single tool
- **Use Git professionally** — Conventional Commits format, feature branch workflow, and `.gitignore` patterns to keep secrets out of version control
- **Navigate the Linux command line** — file operations, permissions, environment variables, process management, and the shell patterns you'll use every day in a backend role
- **Structure a Python project** — packages vs modules, `__init__.py`, relative imports, and the `src/` layout that scales to real projects

---

## ✅ Practice Checklist

- [ ] Set up a Python virtual environment with `venv` and install packages with `pip`
- [ ] Set up the same project using `uv` (`uv init`, `uv add`, `uv run`, `uv sync`)
- [ ] Understand every section of the generated `pyproject.toml`
- [ ] Create a `.env` file, add it to `.gitignore`, and commit a `.env.example`
- [ ] Configure `ruff` in a `pyproject.toml` and set up a pre-commit hook that runs it on commit
- [ ] Write a Python script with full type annotations that passes `mypy`
- [ ] Push a project to GitHub following the feature branch workflow (branch → commit → PR)
- [ ] Create an `app/` package with correct `__init__.py` files and import across modules without errors
- [ ] Install the recommended VS Code extensions and confirm format-on-save works
- [ ] Look at the `UserService` example in section 1.1 — identify which SOLID principle `db` being passed into `__init__` (instead of created inside the class) demonstrates
- [ ] Name which SOLID principle FastAPI's `Depends()` system enforces and write one sentence explaining why

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| https://docs.python.org/3/ | Python language reference |
| https://docs.astral.sh/ruff/ | Ruff linter documentation |
| https://mypy.readthedocs.io | mypy type checker |
| https://pre-commit.com | Pre-commit hooks |
| https://docs.astral.sh/uv/ | uv — fast Python package manager |
| https://www.conventionalcommits.org | Conventional Commits specification |
| https://code.visualstudio.com/docs/python/python-tutorial | VS Code Python setup guide |

## 🃏 Quick-Reference Flash Cards

**Q:** What does a 4xx HTTP status code mean?
**A:** A client error — the request was invalid or the resource wasn't found. Common: 400 Bad Request, 401 Unauthorized, 404 Not Found.

**Q:** What is dependency injection and why use it?
**A:** Passing dependencies (like a DB session) into a class rather than creating them inside it. Makes code testable — you can inject a mock in tests.

**Q:** What's the feature branch workflow?
**A:** Create a branch per feature (`git checkout -b feature/x`), commit there, push, open a PR for review, merge to `main`. Never commit directly to `main`.

**Q:** Why use a virtual environment?
**A:** To isolate project dependencies from the system Python and from other projects, preventing version conflicts.

**Q:** What does `ruff` replace?
**A:** `flake8` (linting), `black` (formatting), and `isort` (import sorting) — all in one faster tool.

**Q:** What's the difference between `str | None` and `Optional[str]`?
**A:** They're equivalent. `str | None` is the Python 3.10+ union syntax and is now preferred.

**Q:** What does `pre-commit install` do?
**A:** Wires hooks into `.git/hooks/pre-commit` so they run automatically before every `git commit`.

**Q:** What makes a Python folder an importable package?
**A:** An `__init__.py` file inside it. Without it, `from app.routers import users` raises `ModuleNotFoundError`.

**Q:** What is Conventional Commits?
**A:** A commit message convention: `<type>: <description>`. Common types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`. Keeps `git log` readable.

**Q:** What does `os.getenv("KEY", "default")` do?
**A:** Reads the `KEY` environment variable. Returns `"default"` if the variable isn't set. Use `python-dotenv` to load `.env` automatically in development.

**Q:** Why should `.vscode/settings.json` be committed?
**A:** It sets project-specific formatter and type checker config. Committing it ensures all team members get the same settings automatically.

**Q:** What does the Single Responsibility Principle say?
**A:** A class should have one reason to change — it does one thing. If your `UserService` also handles auth and sends email, split it into three classes.

**Q:** What does the Open/Closed Principle say?
**A:** Classes should be open for extension (new behaviour via new classes) but closed for modification (existing code untouched). Add a new notification channel by creating `PushChannel`, not by editing `send_notification()`.

**Q:** What does the Liskov Substitution Principle say?
**A:** Any subtype must work wherever the parent type is expected without breaking the caller. If `ReadOnlyRepository` raises on `save()`, it violates the `IUserRepository` contract.

**Q:** What does the Interface Segregation Principle say?
**A:** Don't force classes to implement methods they don't use. Split fat interfaces into focused ones — `IUserReader` and `IUserWriter` instead of one interface with 8 unrelated methods.

**Q:** What does the Dependency Inversion Principle say?
**A:** High-level modules should depend on abstractions, not concretions. `UserService` should accept `IUserRepository`, not `AsyncSession`. This is what makes `Depends()`, repository interfaces in B3, and the IoC container in B7 all work.

*Checkpoint: [[Backend/Checkpoints/CB1 - Dev Environment Ready|CB1]]*

*Next: [[Backend/B2 - Web & API Fundamentals|B2]]*
