# 02 · Authentication State Reuse

Logging in through the UI at the start of every test is slow and, worse,
makes every single test depend on the login form working — a bug in login
takes down your entire suite instead of just the login tests. Playwright lets
you log in once, save the resulting browser storage state, and reuse it
across tests and workers.

## Saving storage state after a UI login

```python
# auth_setup.py — run once to produce storage_state.json
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    page.goto("https://example.com/login")
    page.get_by_label("Email").fill("qa-user@example.com")
    page.get_by_label("Password").fill("correct-horse-battery-staple")
    page.get_by_role("button", name="Sign in").click()
    page.wait_for_url("**/dashboard")
    page.context.storage_state(path="storage_state.json")
    browser.close()
```

```json
// storage_state.json (shape)
{
  "cookies": [
    {"name": "session_id", "value": "…", "domain": "example.com", "path": "/", "expires": 1735689600}
  ],
  "origins": [
    {"origin": "https://example.com", "localStorage": [{"name": "auth_token", "value": "…"}]}
  ]
}
```

`storage_state()` captures both cookies *and* `localStorage`/`sessionStorage`
per origin, which matters because many SPAs keep the actual auth token in
`localStorage`, not a cookie.

## Reusing it as a pytest fixture

```python
# conftest.py
import pytest
from playwright.sync_api import Browser

@pytest.fixture
def authenticated_page(browser: Browser):
    context = browser.new_context(storage_state="storage_state.json")
    page = context.new_page()
    yield page
    context.close()
```

```python
# test_dashboard.py
def test_dashboard_shows_username(authenticated_page):
    authenticated_page.goto("/dashboard")
    expect(authenticated_page.get_by_text("Welcome, qa-user")).to_be_visible()
```

```text
# no login form is ever touched — new_context(storage_state=...)
# seeds the browser context's cookie jar and origin storage
# before the first page is even created, so the app believes
# a real login already happened
```

## A session-scoped fixture to log in once per test run

Logging in via the UI still has a cost even at "once per test file" — do it
once for the whole run instead, then hand every test a context built from
that saved state:

```python
# conftest.py
import pytest
from playwright.sync_api import Playwright, sync_playwright

@pytest.fixture(scope="session")
def storage_state_path(tmp_path_factory):
    path = tmp_path_factory.mktemp("auth") / "state.json"
    with sync_playwright() as p:
        browser = p.chromium.launch()
        page = browser.new_page()
        page.goto("https://example.com/login")
        page.get_by_label("Email").fill("qa-user@example.com")
        page.get_by_label("Password").fill("correct-horse-battery-staple")
        page.get_by_role("button", name="Sign in").click()
        page.wait_for_url("**/dashboard")
        page.context.storage_state(path=str(path))
        browser.close()
    return path

@pytest.fixture
def page(browser, storage_state_path):
    context = browser.new_context(storage_state=str(storage_state_path))
    page = context.new_page()
    yield page
    context.close()
```

```text
# login runs exactly once for the entire pytest session
# (scope="session"), and every test's `page` fixture builds
# a fresh context from the resulting file — tests stay
# isolated from each other (separate contexts) while sharing
# the one-time login cost
```

## Multiple roles: admin vs. regular user

Real apps need more than one identity. Parameterize the saved-state fixture
by role instead of hard-coding one login:

```python
import pytest

ROLES = {
    "admin": ("admin@example.com", "admin-pass"),
    "member": ("member@example.com", "member-pass"),
}

@pytest.fixture(scope="session")
def storage_state_for(tmp_path_factory, playwright):
    cache = {}
    def _get(role: str):
        if role in cache:
            return cache[role]
        email, password = ROLES[role]
        browser = playwright.chromium.launch()
        page = browser.new_page()
        page.goto("/login")
        page.get_by_label("Email").fill(email)
        page.get_by_label("Password").fill(password)
        page.get_by_role("button", name="Sign in").click()
        page.wait_for_url("**/dashboard")
        path = tmp_path_factory.mktemp("auth") / f"{role}.json"
        page.context.storage_state(path=str(path))
        browser.close()
        cache[role] = str(path)
        return cache[role]
    return _get

def test_admin_sees_settings(browser, storage_state_for):
    context = browser.new_context(storage_state=storage_state_for("admin"))
    page = context.new_page()
    page.goto("/settings")
    expect(page.get_by_role("heading", name="Admin Settings")).to_be_visible()
    context.close()

def test_member_cannot_see_settings(browser, storage_state_for):
    context = browser.new_context(storage_state=storage_state_for("member"))
    page = context.new_page()
    page.goto("/settings")
    expect(page.get_by_text("You don't have access")).to_be_visible()
    context.close()
```

## Handling expiring sessions

A saved state with a short-lived session cookie will start failing tests
hours after it was captured. Regenerate it on a schedule rather than trusting
a stale file indefinitely:

```python
import json, time

def is_state_fresh(path: str, max_age_seconds: int = 3600) -> bool:
    import os
    return os.path.exists(path) and (time.time() - os.path.getmtime(path)) < max_age_seconds
```

```text
# a session-scoped fixture can check is_state_fresh() first
# and only re-run the UI login when the cached file is
# missing or older than the app's actual token lifetime
```

## How It Actually Works

`context.storage_state()` works by querying two separate CDP surfaces and
combining them into one JSON document: cookies come from
`Network.getCookies` (or `Storage.getCookies`, scoped to the whole browser
context rather than one page), while `localStorage`/`sessionStorage` per
origin come from `DOMStorage.getDOMStorageItems`, called once for every
origin the context has ever navigated to. Because both are captured at the
`BrowserContext` level rather than per-page, a multi-tab login flow's state
is captured completely in one call.

Feeding that file back in via `browser.new_context(storage_state=...)`
replays it in the opposite direction *before any page exists in the new
context*: cookies are set via `Network.setCookie` calls issued against the
freshly created context, and each origin's storage entries are written
through `DOMStorage.setDOMStorageItem` calls scheduled to run the instant
that origin is first navigated to — Chromium doesn't let you write
`localStorage` for an origin with no document loaded yet, so Playwright
defers those writes and injects them via an init script that runs before
the page's own JavaScript on first load. This ordering is exactly why the
app "believes a real login already happened" the moment `page.goto()`
resolves: by the time any of your test's own JavaScript or React code runs,
both the cookie jar and the storage entries a real login would have
produced are already in place.

## Exercise

1. Write a script that logs in through your app's real login form and saves
   `storage_state.json`.
2. Write a fixture that builds a `BrowserContext` from that file and confirm
   a test can load an authenticated page with zero login steps of its own.
3. Extend the fixture to support two roles (e.g. admin/member) using
   separate saved state files, and write one test per role asserting
   role-specific UI.
4. Add an `is_state_fresh()`-style check (or equivalent) that forces
   regeneration when the saved file is older than your app's session
   lifetime, and explain in a comment why blindly trusting an old file
   would eventually make every test fail for a reason unrelated to the
   code being tested.
