# 08 · Environment & Test Data Management

## The problem: secrets and environment drift

Tests need URLs, credentials, and API keys that differ between local,
staging, and CI — and must never contain real secrets committed to git.
`.env` files plus a loader, read once at session start, is the standard
Python answer.

```bash
# .env (gitignored — never committed)
BASE_URL=https://staging.example.com
TEST_USER_EMAIL=qa+staging@example.com
TEST_USER_PASSWORD=correct-horse-battery-staple
API_KEY=sk_test_abc123
```

```ini
# .gitignore
.env
.env.*
!.env.example
```

```python
# conftest.py
import os
import pytest
from dotenv import load_dotenv

load_dotenv()  # populates os.environ from .env if present

@pytest.fixture(scope="session")
def config():
    return {
        "base_url": os.environ["BASE_URL"],
        "user_email": os.environ["TEST_USER_EMAIL"],
        "user_password": os.environ["TEST_USER_PASSWORD"],
    }
```

```text
# no output — load_dotenv() is a no-op if .env is absent (e.g. in
# CI, where real env vars are injected by the pipeline directly
# instead of a file), so the same code path works both places
```

Commit a `.env.example` with the *keys* but placeholder values, so a new
teammate knows exactly what to fill in without ever seeing a real secret.

## Never hardcode credentials in test files

```python
# BAD
def test_login(page):
    page.get_by_label("Email").fill("real.person@company.com")
    page.get_by_label("Password").fill("hunter2")

# GOOD
def test_login(page, config):
    page.get_by_label("Email").fill(config["user_email"])
    page.get_by_label("Password").fill(config["user_password"])
```

```text
# no output — the good version reads no different in CI vs. local,
# and rotating the real password only means updating one secret,
# not grepping test files for where it was pasted
```

## CI secrets injection

```yaml
# .github/workflows/e2e.yml (excerpt)
env:
  BASE_URL: https://staging.example.com
  TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
  TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
steps:
  - run: pytest
```

```text
# no output — GitHub Actions injects secrets as environment
# variables at run time; os.environ["TEST_USER_PASSWORD"] in
# conftest.py sees the real value without it ever appearing in
# the workflow file or logs (Actions also auto-masks secret values
# that happen to be printed, replacing them with ***)
```

## Test data: fixtures vs. factories vs. API seeding

Static JSON fixtures are simplest for stable reference data.

```python
# fixtures/products.json
[
  {"name": "Wireless Mouse", "price": 24.99, "sku": "WM-100"},
  {"name": "Mechanical Keyboard", "price": 89.00, "sku": "MK-200"}
]
```

```python
import json
import pytest

@pytest.fixture
def sample_products():
    with open("fixtures/products.json") as f:
        return json.load(f)
```

```text
# no output — sample_products() returns a plain list of dicts
# any test can iterate over without hitting a database
```

Dynamic data that must be unique per test run (an email, an order ID)
should be generated, not fixed, so parallel/repeated runs never collide.

```python
import uuid
import pytest
from faker import Faker

fake = Faker()

@pytest.fixture
def new_user_data():
    return {
        "email": f"qa+{uuid.uuid4().hex[:8]}@example.com",
        "name": fake.name(),
        "company": fake.company(),
    }

def test_signup_creates_account(page, new_user_data):
    page.goto("/signup")
    page.get_by_label("Full name").fill(new_user_data["name"])
    page.get_by_label("Email").fill(new_user_data["email"])
    page.get_by_role("button", name="Create account").click()
```

```text
1 passed in 1.63s
```

`Faker` (the `faker` package) generates realistic-looking but fake names,
addresses, and companies — better test coverage than "Test User 1" for
catching UI truncation/formatting bugs, and the `uuid` suffix on the email
guarantees no two test runs (or two parallel workers) ever collide on a
uniqueness constraint the real backend enforces.

## Seeding data via API instead of the UI

For anything a test needs to *exist* before it starts (an order to
cancel, a document to delete), creating it through the UI wastes time and
couples an unrelated test to the signup flow's stability. Seed via the
backend's own API instead (Level 3 covers this with Playwright's
`APIRequestContext` in depth).

```python
def test_cancel_existing_order(page, api_context, config):
    order = api_context.post(
        f"{config['base_url']}/api/orders",
        data={"item": "Wireless Mouse", "qty": 1},
    ).json()

    page.goto(f"/orders/{order['id']}")
    page.get_by_role("button", name="Cancel order").click()

    from playwright.sync_api import expect
    expect(page.get_by_text("Order cancelled")).to_be_visible()
```

```text
1 passed in 0.97s
```

## Cleaning up after tests

```python
import pytest

@pytest.fixture
def temp_order(api_context, config):
    order = api_context.post(f"{config['base_url']}/api/orders", data={}).json()
    yield order
    api_context.delete(f"{config['base_url']}/api/orders/{order['id']}")
```

```text
# no output — the delete call after yield runs even if the test
# body fails an assertion, keeping the environment from
# accumulating orphaned test data across thousands of CI runs
```

## How It Actually Works

None of `.env` loading, `Faker`, or API seeding touch Playwright's
automation layer at all — they operate purely in your Python process, one
level above the browser. What's worth understanding mechanically is *why*
API seeding (rather than UI seeding) is the better default once a suite
grows: `api_context.post(...)` uses Playwright's `APIRequestContext`
(Level 3 covers it fully), which sends a plain HTTP request directly from
the Node driver process — no browser, no page, no CDP navigation or
rendering involved at all. It shares the same cookie jar as a
`BrowserContext` you create it from, though, which is why an
API-authenticated session and a UI-driven `page` in the same context see
each other's login state without any extra wiring: the session cookie set
by the API call is visible to subsequent `page.goto()` calls in that
context, and vice versa, because both are reading and writing the same
underlying context-scoped cookie store the browser process maintains.

The `yield`-based teardown pattern for `temp_order` relies on the exact same
pytest guarantee discussed in Module 2: the code after `yield` runs in a
`finally`-equivalent block pytest manages, independent of whether the test
body passed, failed an assertion, or raised — which is what makes it safe
to rely on for cleaning up real backend state (an order, a seeded resource)
rather than a plain `return` plus a manual cleanup call a developer could
forget to add to every test.

## Exercise

1. Add `python-dotenv` and `faker` to your project, create a `.env` with a
   `BASE_URL` and dummy credentials, and a matching `.env.example` with
   placeholders; confirm `.env` is gitignored.
2. Write a `config` session fixture reading from `os.environ`, and refactor
   one existing test to use it instead of a hardcoded URL/credential.
3. Write a `new_user_data` fixture using `faker` and `uuid` producing a
   unique name/email each call; use it in a signup test, running the test
   twice in a row to confirm it doesn't collide on a "email already
   registered" error.
4. Write a `temp_resource` fixture that creates something via a public
   API (e.g. `https://jsonplaceholder.typicode.com/posts`), yields the
   created object's id, and deletes it in teardown; add a print statement
   in the teardown so you can see it actually running after the test body.
