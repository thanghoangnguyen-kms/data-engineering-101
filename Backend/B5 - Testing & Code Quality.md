---
tags:
  - BE101
  - backend-5
  - testing
date: 2026-06-27
status: complete
domain: "5 of 8"
track: backend
---

# B5 — Testing & Code Quality

**Back to:** [[Backend/00 - Backend Track Roadmap|Backend Track Roadmap]]

> [!NOTE] Domain Overview
> You'll build a complete test suite for a FastAPI + SQLAlchemy application: unit tests, integration tests, and end-to-end tests. You'll measure code coverage, enforce quality gates, and add static type checking with mypy.

---

## 5.1 — pytest Fundamentals

> [!NOTE]
> pytest is the standard Python testing framework. It discovers and runs tests automatically, has a rich fixture system, and integrates with async code via `pytest-asyncio`.

**Install dev dependencies**

```bash
uv add --dev pytest pytest-asyncio httpx pytest-mock pytest-cov
```

| Package | Purpose |
|---------|---------|
| `pytest` | Test runner and assertion engine |
| `pytest-asyncio` | Run `async def` test functions |
| `httpx` | HTTP client used by FastAPI's `AsyncClient` |
| `pytest-mock` | `mocker` fixture for patching and mocks |
| `pytest-cov` | Coverage measurement and reporting |

**Configure pytest in `pyproject.toml`**

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-v --cov=app --cov-report=html"
```

> [!IMPORTANT] `asyncio_mode = "auto"`
> With this setting, any `async def test_*` function is automatically treated as an async test — no `@pytest.mark.asyncio` decorator needed on every function. Always set this when testing FastAPI with async routes.

**Test discovery rules**

pytest finds tests automatically if you follow these naming conventions:

| Convention | Example |
|-----------|---------|
| File names | `test_security.py`, `test_users.py` |
| Function names | `def test_<scenario>_<expected_outcome>():` |

Name tests after the *scenario and expected outcome*, not the function being called:

```python
# ❌ tells you nothing when it fails
def test_login():

# ✅ tells you exactly what broke
def test_login_returns_401_for_wrong_password():
def test_login_returns_200_and_token_for_valid_credentials():
```

**The AAA Pattern**

Every test has exactly three parts — keep them visually separated:

```python
def test_verify_password_returns_false_for_wrong_input():
    # Arrange — set up the inputs
    plain = "secret123"
    hashed = hash_password(plain)

    # Act — call the thing being tested
    result = verify_password("wrong_password", hashed)

    # Assert — check the outcome
    assert result is False
```

AAA makes failures self-diagnosing: if the test breaks, you can immediately see whether the setup, the call, or the expectation is wrong.

**`conftest.py` — shared fixtures**

`conftest.py` is a special pytest file: fixtures defined in it are automatically available to all tests in the same directory and below — no import needed.

```
project/
├── conftest.py            ← fixtures available to ALL tests
├── tests/
│   ├── conftest.py        ← fixtures available to tests/ and subdirectories
│   ├── unit/
│   │   ├── conftest.py    ← fixtures available to tests/unit/ only
│   │   └── test_security.py
│   └── integration/
│       └── test_users.py
```

A fixture in `tests/unit/conftest.py` is **not** visible to `tests/integration/`. Place shared fixtures at the highest level they're needed.

**Fixtures**

Fixtures are reusable setup functions that pytest injects into tests via argument names:

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from app.main import app


@pytest.fixture
def client() -> TestClient:
    return TestClient(app)
```

```python
# tests/unit/test_users.py
def test_root_returns_200(client: TestClient) -> None:
    response = client.get("/")
    assert response.status_code == 200
```

**Fixture scope**

Scope controls how often a fixture is created and destroyed:

| Scope | Created once per… | Teardown after… |
|-------|------------------|-----------------|
| `function` (default) | Each test function | Each test function |
| `module` | Each test file | End of file |
| `session` | Entire test run | All tests complete |

Use `yield` inside a fixture to run teardown code after the test:

```python
from pathlib import Path


@pytest.fixture
def temp_file(tmp_path: Path):
    path = tmp_path / "test_data.json"
    path.write_text("{}")
    yield path           # test runs here
    path.unlink()        # runs after the test, even if it fails
```

> [!WARNING] Session-Scoped Fixtures Share State
> A `session`-scoped DB fixture that isn't rolled back between tests causes **order-dependent failures** — a test that passes alone fails when run after another test that left dirty data. Unless you explicitly isolate state, use `function` scope for anything that writes data.

**Parametrize — one test, many inputs**

```python
import pytest
from app.core.security import verify_password, hash_password


@pytest.mark.parametrize("plain,expected", [
    ("correct_password", True),
    ("wrong_password", False),
    ("", False),
    ("   ", False),
])
def test_verify_password(plain: str, expected: bool) -> None:
    hashed = hash_password("correct_password")
    assert verify_password(plain, hashed) == expected
```

pytest runs this as four separate tests, each with its own pass/fail result.

---

## 5.2 — Unit Testing

> [!NOTE]
> A unit test exercises one piece of logic in complete isolation — no database, no network, no filesystem. Fast, deterministic, and run hundreds of times a day.

**What to unit test in a FastAPI project**

| Good unit test targets | Why |
|-----------------------|-----|
| Security utilities (`hash_password`, `create_access_token`) | Pure functions — no I/O, deterministic |
| Pydantic validators (`@field_validator`) | Pure business logic |
| Service functions (user creation, auth logic) | Domain logic; DB calls can be mocked |
| Helper/utility functions | Typically pure |

| Poor unit test targets | Why |
|-----------------------|-----|
| Route handlers directly (via TestClient) | That's integration/e2e testing |
| Framework behaviour (does FastAPI return 422 for missing fields?) | You're testing FastAPI, not your code |

**Testing pure functions — no mocks needed**

```python
# tests/unit/test_security.py
import pytest
import jwt
from app.core.security import (
    create_access_token,
    decode_access_token,
    hash_password,
    verify_password,
)


def test_hash_password_produces_different_hashes_each_time() -> None:
    # bcrypt uses a random salt — same input never produces the same hash
    hashed_1 = hash_password("secret")
    hashed_2 = hash_password("secret")
    assert hashed_1 != hashed_2


def test_verify_password_returns_true_for_correct_input() -> None:
    hashed = hash_password("mypassword")
    assert verify_password("mypassword", hashed) is True


def test_verify_password_returns_false_for_wrong_input() -> None:
    hashed = hash_password("mypassword")
    assert verify_password("wrong", hashed) is False


def test_create_access_token_contains_expected_claims() -> None:
    # Arrange
    data = {"sub": "42", "role": "user"}

    # Act
    token = create_access_token(data)
    decoded = decode_access_token(token)

    # Assert
    assert decoded["sub"] == "42"
    assert decoded["role"] == "user"
    assert "exp" in decoded


def test_decode_access_token_raises_for_tampered_token() -> None:
    token = create_access_token({"sub": "1"})
    tampered = token[:-5] + "XXXXX"  # corrupt the signature
    with pytest.raises(jwt.InvalidTokenError):
        decode_access_token(tampered)
```

**Mocking dependencies with `pytest-mock`**

When the code under test calls a collaborator (database, external API), replace that collaborator with a controlled mock so the test stays isolated:

```python
# tests/unit/test_auth_service.py
import pytest
from fastapi import HTTPException
from app.core.security import hash_password
from app.models.user import User
from app.routers.auth import login
from app.schemas.auth import LoginRequest


async def test_login_returns_401_when_user_not_found(mocker) -> None:
    # Arrange — mock the DB session to return no user
    mock_db = mocker.AsyncMock()
    mock_db.scalar.return_value = None  # no user found

    body = LoginRequest(email="nobody@example.com", password="pass")

    # Act + Assert
    with pytest.raises(HTTPException) as exc_info:
        await login(body, mock_db)

    assert exc_info.value.status_code == 401


async def test_login_returns_401_when_password_is_wrong(mocker) -> None:
    # Arrange — DB returns a user, but password won't match
    mock_user = mocker.MagicMock(spec=User)
    mock_user.hashed_password = hash_password("correct_password")
    mock_user.role = "user"

    mock_db = mocker.AsyncMock()
    mock_db.scalar.return_value = mock_user

    body = LoginRequest(email="alice@example.com", password="wrong_password")

    # Act + Assert
    with pytest.raises(HTTPException) as exc_info:
        await login(body, mock_db)

    assert exc_info.value.status_code == 401
```

> [!WARNING] Mock at the Right Boundary
> ❌ Mocking SQLAlchemy internals (`Session._execute`, engine internals) — your tests become coupled to the ORM's implementation details and break on upgrades
> ✅ Mock at the dependency boundary: pass a mock `AsyncSession` directly, or override `get_db` via `dependency_overrides`

**Testing protected routes with `dependency_overrides`**

You can't easily forge a real JWT in unit tests — and you shouldn't have to. FastAPI's `dependency_overrides` lets you swap any `Depends()` function with a fake for the duration of a test:

```python
# tests/unit/test_admin_routes.py
import pytest
from fastapi.testclient import TestClient
from app.main import app
from app.dependencies.auth import get_current_user


@pytest.fixture
def client_as_user() -> TestClient:
    app.dependency_overrides[get_current_user] = lambda: {"sub": "1", "role": "user"}
    yield TestClient(app)
    app.dependency_overrides.clear()  # always clean up after the test


@pytest.fixture
def client_as_admin() -> TestClient:
    app.dependency_overrides[get_current_user] = lambda: {"sub": "1", "role": "admin"}
    yield TestClient(app)
    app.dependency_overrides.clear()


def test_admin_delete_returns_403_for_non_admin(client_as_user: TestClient) -> None:
    response = client_as_user.delete("/admin/users/1")
    assert response.status_code == 403


def test_admin_delete_succeeds_for_admin(client_as_admin: TestClient) -> None:
    response = client_as_admin.delete("/admin/users/1")
    assert response.status_code == 204  # assumes the handler returns 204 No Content
```

> [!IMPORTANT] Always Clear `dependency_overrides`
> If you don't call `app.dependency_overrides.clear()` after a test, the override leaks into other tests. Always use a `yield` fixture with cleanup in the teardown section — never rely on test ordering to clean up.

> [!WARNING] Don't Test the Framework
> ❌ `assert response.status_code == 422` after sending a request with a missing required field — you're testing that Pydantic works, not your code
> ✅ Test the logic *you* wrote: your validators, your service functions, your error messages

---

## 5.3 — Integration Testing (Concepts)

> [!NOTE]
> An integration test connects two or more real components — typically your app and a real database — to verify they work together correctly. Concepts only here; hands-on setup with Docker Compose + PostgreSQL is in [[Backend/B7 - Microservices & Containers|B7]].

**The Test Pyramid**

```
        ▲ e2e (fewest — full stack, slowest, most brittle)
       ███
      ███████  integration (fewer — app + real DB, medium speed)
     ███████████
    ███████████████  unit (most — isolated, fast, high coverage)
```

Run units constantly during development. Run integration tests before merging. Run e2e tests on the full build.

**What integration tests cover that unit tests miss**

| Scenario | Unit test? | Integration test? |
|----------|-----------|-----------------|
| Service logic | ✅ | — |
| SQL query correctness | ❌ | ✅ |
| Unique constraint violations | ❌ | ✅ |
| Transaction rollback on error | ❌ | ✅ |
| ORM relationship loading | ❌ | ✅ |

**Setup pattern (concept)**

Integration tests need a test database that:
1. **Starts clean** — every test begins with known state (no leftover rows from a previous test)
2. **Isolates changes** — one test's writes don't affect another's reads
3. **Is fast to reset** — SQLite in-memory is recreated per test; PostgreSQL can use transactions that roll back

The typical pattern in `conftest.py`:

```
create async engine (test DB)
  └── create all tables
      └── open AsyncSession
          └── yield session to test  ← test runs here
          └── rollback transaction   ← changes vanish
      └── close session
  └── drop all tables
```

**SQLite in-memory vs PostgreSQL for tests**

| | SQLite in-memory | PostgreSQL test DB |
|--|------------------|--------------------|
| Setup | Zero-config | Requires Docker or a test server |
| Speed | Very fast | Fast (with connection pooling) |
| Postgres-specific features | ❌ (RETURNING, arrays, JSONB) | ✅ |
| Suitable for | Learning, simple queries | Production parity |

> [!TIP] Start Simple, Then Add Parity
> Use SQLite in-memory for early integration tests to avoid Docker dependency. Add a real PostgreSQL test container in B7 when you need production parity. `pydantic-settings` makes switching just a `TEST_DATABASE_URL` env var change.

---

## 5.4 — End-to-End Testing (Concepts)

> [!NOTE]
> An end-to-end (e2e) test sends real HTTP requests through the full FastAPI application stack — routing, middleware, validation, dependencies, and handlers — and checks the HTTP response. Concepts only; hands-on e2e test writing follows naturally from the integration setup in [[Backend/B7 - Microservices & Containers|B7]].

**TestClient vs AsyncClient**

FastAPI provides two ways to make test HTTP requests:

| | `TestClient` | `AsyncClient` |
|--|--------------|---------------|
| From | `fastapi.testclient` | `httpx.AsyncClient` |
| How | Synchronous wrapper around httpx | Async — use with `async def test_*` |
| Best for | Simple route tests, dependency override tests | Async handlers, concurrent request testing |
| Import | `from fastapi.testclient import TestClient` | `from httpx import AsyncClient` |

```python
# Sync — simplest form
from fastapi.testclient import TestClient
client = TestClient(app)
response = client.get("/me")

# Async — when you need async in the test itself
from httpx import AsyncClient, ASGITransport
async def test_me():
    async with AsyncClient(
        transport=ASGITransport(app=app), base_url="http://test"
    ) as ac:
        response = await ac.get("/me")
```

**What e2e tests cover**

E2e tests verify complete user journeys — scenarios that span multiple endpoints:

- Register → Login → Access a protected resource
- Create a resource → Retrieve it → Delete it → Verify it's gone
- Submit invalid data → Verify the correct error shape is returned

**Cost vs value**

E2e tests are the most expensive tests to write and maintain:
- **Slow** — every test boots the full app stack
- **Brittle** — a middleware change can break dozens of unrelated e2e tests
- **Valuable** — they're the only tests that catch integration gaps between layers

> [!NOTE] "E2E" in Test Scope
> At intern level, e2e means the full HTTP layer — request in, response out — still using the test DB override, not necessarily a production database. "True" e2e against a full production-like environment is a CI/CD concern covered in B7.

**Rule of thumb:** have one e2e test per critical user flow (login works, register works, protected route rejects unauthenticated requests). Everything else is unit or integration.

---

## 5.5 — Coverage & Quality Gates

> [!NOTE]
> Coverage measures which lines of your code are executed during tests. A quality gate is an automated check that fails a build when coverage drops below a threshold — keeping test debt from accumulating.

**Generating coverage reports**

With `pytest-cov` installed, add `--cov` to your test run:

```bash
# Run tests with coverage, output to terminal
pytest --cov=app --cov-report=term-missing

# Run tests with HTML report (open htmlcov/index.html in browser)
pytest --cov=app --cov-report=html
```

`--cov-report=term-missing` shows line numbers that weren't executed — useful for finding exactly what's untested.

**Configure in `pyproject.toml`**

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-v --cov=app --cov-report=html"

[tool.coverage.run]
omit = [
    "*/alembic/*",     # migrations are not unit-testable
    "*/__init__.py",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",  # type-only imports don't need to run
]
fail_under = 80
```

`fail_under = 80` in `[tool.coverage.report]` is the authoritative gate — coverage.py exits non-zero when coverage drops below 80%, which fails any CI job running `pytest`. Defining it here is sufficient; you don't need to duplicate it in `addopts`.

> [!TIP] Skip Coverage During Fast Iteration
> With `--cov` in `addopts`, every `pytest` call generates a coverage report — even a quick single-file run. Use `pytest --no-cov` to skip coverage when you're iterating rapidly on a specific test. Run the full coverage report before committing.

**Coverage in CI**

The quality gate only matters if it's enforced automatically. In GitHub Actions:

```yaml
# .github/workflows/test.yml
- name: Run tests
  run: pytest  # fails if coverage < 80% (from fail_under config)
```

Because `fail_under` is in `pyproject.toml`, the CI job inherits it without any extra flags. A PR that drops coverage below 80% fails the check and cannot be merged.

> [!WARNING] 100% Coverage ≠ Good Tests
> A test that calls a function but makes no assertions is 100% coverage and 0% useful:
> ```python
> def test_login():
>     login(body, db)  # no assert — this covers the lines but proves nothing
> ```
> Coverage measures *what ran*, not *what was verified*. A 90% covered suite with meaningful assertions beats a 100% covered suite full of empty tests.

**The quality gate pipeline**

Enforcement happens in layers, from fastest to slowest:

```
pre-commit (on every git commit)
    └── ruff check --fix        ← lint + auto-fix
    └── ruff format             ← formatting
    └── mypy app/               ← type check
CI (on every push / PR)
    └── pytest --cov            ← all tests + coverage gate
```

pre-commit gives fast local feedback. CI is the authoritative gate that blocks bad merges. The `pre-commit` setup is in [[Backend/B1 - Foundations & Dev Setup|B1]] — B5 just adds `mypy` and `pytest -x` to the hook list:

```yaml
# .pre-commit-config.yaml (add to existing config from B1)
- repo: local
  hooks:
    - id: mypy
      name: mypy
      entry: mypy app/
      language: system
      types: [python]
    - id: pytest-unit
      name: pytest (unit tests only)
      entry: pytest tests/unit/ -x --no-header -q
      language: system
      pass_filenames: false
```

> [!TIP] Keep Pre-commit Fast
> Only run the unit test suite (`tests/unit/`) in pre-commit hooks. Unit tests should complete in under 5 seconds. Integration and e2e tests run in CI (on every push/PR) where longer runtimes are acceptable. Running the full suite on every commit trains developers to use `git commit --no-verify`, which defeats the purpose of the hooks.

---

## 5.6 — Type Checking with mypy

> [!NOTE]
> mypy reads your Python source code without running it and reports type errors — wrong argument types, unhandled `None` values, missing return annotations. It catches bugs that tests might miss because the bug only manifests with specific input types.

**Install and configure**

mypy is already in your dev dependencies from B1. Configure it in `pyproject.toml`:

```toml
[tool.mypy]
python_version = "3.11"
strict = true
plugins = ["sqlalchemy.ext.mypy.plugin"]

# Third-party libraries that don't ship type stubs
[[tool.mypy.overrides]]
module = [
    "passlib.*",
    "alembic.*",
]
ignore_missing_imports = true
```

> [!IMPORTANT] SQLAlchemy 2.0 Needs the mypy Plugin
> Without `plugins = ["sqlalchemy.ext.mypy.plugin"]`, running `mypy --strict` on code that uses `Mapped[]` annotations (from B3) produces dozens of false errors. Add the plugin and install `sqlalchemy[mypy]`:
> ```bash
> uv add --dev sqlalchemy[mypy]
> ```

**What `strict = true` enables**

`strict` is a shorthand that turns on the most important mypy flags at once:

| Flag | What it enforces |
|------|-----------------|
| `disallow_untyped_defs` | Every function must have type annotations |
| `disallow_any_generics` | No bare `list` or `dict` — must be `list[str]`, `dict[str, int]` |
| `warn_return_any` | Functions can't silently return `Any` |
| `no_implicit_optional` | `def f(x: str = None)` must be `def f(x: str | None = None)` |

**Most common errors interns hit**

```python
# Error 1: Missing return type annotation
def get_user(user_id):                          # ❌ missing annotation
    ...

def get_user(user_id: int) -> User | None:      # ✅
    ...


# Error 2: Unhandled Optional — payload.get() returns str | None
async def get_current_user(token: str) -> dict:
    payload = decode_access_token(token)
    user_id = payload.get("sub")               # type: str | None
    user = await db.get(User, int(user_id))    # ❌ int(None) would crash

    # ✅ handle the None case first
    if user_id is None:
        raise HTTPException(status_code=401, detail="Invalid token")
    user = await db.get(User, int(user_id))    # now user_id is str


# Error 3: Bare generic container
users: list = []                               # ❌ mypy infers list[Any]
users: list[User] = []                         # ✅
```

**Stub packages**

Some libraries don't include type information. mypy reports `error: Library stubs not found`. Fix by installing the stub package:

```bash
uv add --dev types-passlib
```

Common stubs for this project:

| Library | Stub package |
|---------|-------------|
| `passlib` | `types-passlib` |
| `PyYAML` | `types-PyYAML` |
| `redis` | `types-redis` |

**`# type: ignore` — when it's OK vs a red flag**

```python
# ✅ Acceptable — genuinely untyped third-party code with no stubs available
result = some_untyped_library.call()  # type: ignore[no-untyped-call]

# ❌ Red flag — hiding a real type error in your own code
user_id = payload.get("sub")
db.get(User, int(user_id))           # type: ignore  ← masking a real None risk
```

If you find yourself writing `# type: ignore` on your own code, it's a signal to fix the type rather than silence the warning.

> [!TIP] Adopt mypy Incrementally
> If you inherit a codebase with no types, start with:
> ```bash
> mypy app/ --ignore-missing-imports
> ```
> Fix the errors it reports. Then remove `--ignore-missing-imports` and add stubs. Then enable `strict = true`. Trying to go from zero to strict in one step is overwhelming.

**Running mypy**

```bash
# Check the entire app
mypy app/

# Check a single file
mypy app/core/security.py

# Show error codes (useful for targeted type: ignore comments)
mypy app/ --show-error-codes
```

---

## 🎯 What You Learned

You can now:

- **Write unit tests with pytest** — Arrange/Act/Assert structure, fixtures with appropriate scope, `@pytest.mark.parametrize` for data-driven cases, and `pytest.raises` for error paths
- **Mock external dependencies cleanly** — `unittest.mock.patch`, `MagicMock`/`AsyncMock`, and FastAPI's `dependency_overrides` to swap real DB sessions and auth for fakes in tests
- **Measure and enforce coverage** — `pytest --cov` with `fail_under = 80` in `pyproject.toml`, and why 100% coverage does not mean bug-free code
- **Run static analysis** — `mypy --strict` catches type errors before runtime; `Ruff` enforces code style; `pre-commit` runs both automatically on every commit
- **Understand the test pyramid** — unit tests for logic (fast, no I/O), integration tests for DB/API contracts (slower, real infrastructure), and why the split matters for CI speed

---

## ✅ Practice Checklist

- [ ] Configure `pyproject.toml` with `asyncio_mode = "auto"`, `testpaths`, and `addopts`
- [ ] Write a parametrized test covering at least 3 input cases for one function
- [ ] Write unit tests for `hash_password`, `verify_password`, `create_access_token`, and `decode_access_token`
- [ ] Write a test that uses `mocker.AsyncMock()` to mock a database call
- [ ] Use `app.dependency_overrides` to test both a `user` and an `admin` role on a protected route — clean up with `app.dependency_overrides.clear()` in teardown
- [ ] Generate an HTML coverage report with `--cov-report=html` and find at least one untested code path
- [ ] Set `fail_under = 80` in `pyproject.toml` and verify the test run exits non-zero below that threshold
- [ ] Add `mypy` to your pre-commit hooks and run `mypy app/` — fix all errors it reports
- [ ] Fix the three most common mypy errors: missing return type, unhandled `Optional`, and bare generic container
- [ ] Explain the difference between unit and integration testing to a colleague without referring to notes

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| https://docs.pytest.org | pytest — full reference |
| https://pytest-asyncio.readthedocs.io | pytest-asyncio — async test configuration |
| https://pytest-mock.readthedocs.io | pytest-mock — mocker fixture reference |
| https://coverage.readthedocs.io | coverage.py — report configuration |
| https://mypy.readthedocs.io | mypy — type checker documentation |
| https://mypy.readthedocs.io/en/stable/config_file.html | mypy `pyproject.toml` config options |
| https://docs.sqlalchemy.org/en/20/orm/extensions/mypy.html | SQLAlchemy mypy plugin setup |
| https://fastapi.tiangolo.com/tutorial/testing/ | FastAPI testing docs — TestClient and AsyncClient |

## 🃏 Quick-Reference Flash Cards

**Q:** What is the AAA pattern in testing?
**A:** Arrange (set up inputs and state) → Act (call the code under test) → Assert (verify the outcome). Every test has exactly these three sections.

**Q:** What does `asyncio_mode = "auto"` do in `pyproject.toml`?
**A:** Makes pytest-asyncio treat every `async def test_*` function as an async test automatically — no `@pytest.mark.asyncio` decorator needed per function.

**Q:** How do you test a protected FastAPI route without a real JWT?
**A:** Use `app.dependency_overrides[get_current_user] = lambda: {"sub": "1", "role": "admin"}`. Always clear it with `app.dependency_overrides.clear()` in fixture teardown.

**Q:** What is a fixture scope and what is the main risk of `session` scope?
**A:** Scope controls how often a fixture is created (`function` = per test, `module` = per file, `session` = once per run). Session scope risks shared mutable state — tests that write data can corrupt the state for tests that run after them, causing flaky order-dependent failures.

**Q:** What determines whether a `conftest.py` fixture is visible to a test?
**A:** Directory containment — a fixture in `conftest.py` is visible to all tests in the same directory and its subdirectories. A fixture in `tests/unit/conftest.py` is not visible to `tests/integration/`.

**Q:** What is the difference between unit and integration testing?
**A:** Unit tests exercise one piece of logic in isolation — no I/O, dependencies are mocked. Integration tests connect two or more real components (e.g. app + database) to verify they work together correctly.

**Q:** What does `--cov-fail-under=80` (or `fail_under = 80` in config) do?
**A:** Causes pytest-cov to exit with a non-zero code if total coverage is below 80%, which causes CI jobs to fail. It's the mechanism that enforces the coverage quality gate.

**Q:** Why is 100% coverage not the same as good tests?
**A:** Coverage measures which lines were *executed*, not which behaviours were *verified*. A test that calls a function with no assertions covers the lines but proves nothing. Meaningful assertions matter more than line counts.

**Q:** Why does `strict = true` in mypy require the SQLAlchemy plugin?
**A:** `Mapped[]` annotations in SQLAlchemy 2.0 use runtime descriptors that mypy can't understand without help. The plugin teaches mypy how to interpret them so it doesn't produce false errors on every column definition.

**Q:** When is `# type: ignore` acceptable?
**A:** When suppressing errors from a genuinely untyped third-party library that has no stubs available. It is a red flag when used on your own code to silence a real type error — fix the type instead.

*Checkpoint: [[Backend/Checkpoints/CB5 - Test Suite Complete|CB5]]*

*Previous: [[Backend/B4 - Authentication & Security|B4]] | Next: [[Backend/B6 - Async, Queues & Background Jobs|B6]]*
