# 02 · Fixtures & Test Hooks

## The problem fixtures solve

Every test in Module 1's suite repeated `LoginPage(page)` and often
`login.goto()`. Multiply that by fifty tests across ten page objects, and
setup code dominates the file. Pytest fixtures let you declare "here is
something this test needs" as a function parameter, and pytest builds it,
hands it to the test, and tears it down afterward — without the test ever
calling a constructor itself.

```python
# conftest.py
import pytest
from playwright.sync_api import Page
from pages.login_page import LoginPage

@pytest.fixture
def login_page(page: Page) -> LoginPage:
    lp = LoginPage(page)
    lp.goto()
    return lp
```

```python
# tests/test_login.py
def test_invalid_login_shows_error(login_page):
    login_page.login("user@example.com", "wrong-password")
    from playwright.sync_api import expect
    expect(login_page.error).to_be_visible()
```

```text
1 passed in 1.02s
```

`page` here is itself a fixture — it comes from `pytest-playwright`, the
official plugin, which is why `pip install pytest-playwright` plus
`playwright install` gives you `page`, `browser`, `context`, and
`browser_name` for free in every test without writing any of them yourself.

## Fixture scope

By default a fixture runs fresh for every test (`function` scope, the
default). Some setup is expensive enough that you want to do it once per
test file, or once per whole run — a database seed, a browser instance
you don't want to relaunch fifty times.

```python
import pytest
from playwright.sync_api import sync_playwright, Browser

@pytest.fixture(scope="session")
def browser() -> Browser:
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        yield browser
        browser.close()

@pytest.fixture(scope="function")
def context(browser: Browser):
    ctx = browser.new_context()
    yield ctx
    ctx.close()

@pytest.fixture(scope="function")
def page(context):
    pg = context.new_page()
    yield pg
    pg.close()
```

```text
# no output — this is effectively what pytest-playwright provides
# out of the box; shown here so the scoping is visible, not opaque
```

The `yield` splits the fixture into setup (before `yield`) and teardown
(after) — pytest runs the code after `yield` once the test using it is
done, guaranteeing cleanup even if the test fails, because pytest wraps it
in the equivalent of a `try/finally`.

Scoping `browser` to `session` but `context` to `function` is the standard
Playwright pattern: launching a browser process is slow (hundreds of ms to
seconds) and safe to reuse; a `BrowserContext` is cheap to create and gives
each test an isolated cookie jar/storage, so tests never leak session state
into each other.

## Hooks: `autouse` and `conftest.py` reach

```python
# conftest.py
import pytest

@pytest.fixture(autouse=True)
def slow_test_warning(request):
    yield
    # runs after every test in this directory and subdirectories, no
    # test needs to request it by name
    duration = request.node.rep_call.duration if hasattr(request.node, "rep_call") else None
```

```python
@pytest.fixture(autouse=True)
def reset_test_data(page):
    """Runs before AND after every test automatically."""
    print("seeding fresh test data")
    yield
    print("cleaning up test data")
```

```text
seeding fresh test data
.
cleaning up test data
```

`autouse=True` fixtures apply to every test in the file (or, if placed in
`conftest.py`, every test in that directory and below) without being named
as a parameter. Use them sparingly — for genuinely universal setup like
resetting a database or clearing localStorage — because a test's fixture
list is otherwise your best documentation of what it actually depends on;
hidden autouse magic can make failures harder to diagnose.

## Fixture composition and parametrization

Fixtures can depend on other fixtures, and can be parametrized so the same
test runs multiple times with different fixture values.

```python
import pytest

@pytest.fixture(params=["chromium", "firefox", "webkit"])
def browser_type_name(request):
    return request.param

@pytest.fixture
def page_in_browser(playwright, browser_type_name):
    browser = getattr(playwright, browser_type_name).launch()
    context = browser.new_context()
    page = context.new_page()
    yield page
    context.close()
    browser.close()

def test_homepage_loads(page_in_browser):
    page_in_browser.goto("https://example.com")
    assert page_in_browser.title() != ""
```

```text
test_homepage_loads[chromium] PASSED
test_homepage_loads[firefox] PASSED
test_homepage_loads[webkit] PASSED
```

pytest runs `test_homepage_loads` three times, once per `params` value,
labeling each in the output — a cleaner way to achieve cross-browser
coverage than duplicating test functions (Level 4 covers the built-in,
more idiomatic `--browser` CLI flag pytest-playwright provides for this
exact purpose).

## `pytest.ini` / marker-based hooks

```ini
# pytest.ini
[pytest]
markers =
    smoke: quick tests run on every commit
    slow: full regression suite
```

```python
import pytest

@pytest.mark.smoke
def test_homepage_loads(page):
    page.goto("https://example.com")
    assert page.title()
```

```bash
pytest -m smoke
```

```text
1 passed in 0.91s
```

Registering markers in `pytest.ini` avoids `PytestUnknownMarkWarning` and
documents, in one place, every category of test your suite recognizes —
useful once you have hundreds of tests and want to run only a subset in a
pre-commit hook versus nightly CI.

## Exercise

1. In `conftest.py`, write a `session`-scoped fixture `api_base_url` that
   returns a string constant, and a `function`-scoped fixture `authed_page`
   that depends on `page`, navigates to a login page, logs in with a
   hardcoded test account, and yields the now-authenticated `page`.
2. Write a test that uses `authed_page` and asserts a dashboard-only
   element is visible — notice the test never mentions the login flow.
3. Add an `autouse=True` fixture that takes a screenshot named after the
   test (`request.node.name`) into a `screenshots/` folder after every
   test, whether it passes or fails, using `page.screenshot(path=...)`.
4. Convert one existing test into a parametrized fixture-driven test that
   runs against two different demo accounts (`params=["free_user",
   "pro_user"]`), asserting each account sees the pricing tier it should.
