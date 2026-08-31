# 09 · Extending Playwright (Custom Fixtures/Plugins)

Once a team's suite has grown past a handful of files, repeated fixture
boilerplate across projects becomes worth packaging. This module covers
building reusable pytest plugins on top of `pytest-playwright`, and writing
your own custom assertions in the same idiom as Playwright's built-in
`expect()`.

## Packaging shared fixtures as an installable pytest plugin

```text
my_company_playwright_fixtures/
├── pyproject.toml
├── my_company_playwright_fixtures/
│   ├── __init__.py
│   └── plugin.py
```

```python
# my_company_playwright_fixtures/plugin.py
import pytest

@pytest.fixture(scope="session")
def api_context(playwright):
    ctx = playwright.request.new_context(base_url="https://staging.example.com")
    yield ctx
    ctx.dispose()

@pytest.fixture
def authenticated_page(browser, storage_state_path):
    context = browser.new_context(storage_state=storage_state_path)
    page = context.new_page()
    yield page
    context.close()
```

```toml
# pyproject.toml
[project.entry-points.pytest11]
my_company_fixtures = "my_company_playwright_fixtures.plugin"
```

```text
# the pytest11 entry point is what makes `pip install
# my-company-playwright-fixtures` alone enough for any team's
# conftest.py to use `authenticated_page` with zero extra
# wiring — pytest auto-discovers and loads registered plugins
# at startup, the same mechanism pytest-playwright itself uses
```

## Custom `expect()`-style assertions via `expect.extend`

```python
# assertions/custom_matchers.py
from playwright.sync_api import expect, Locator
import re

def to_have_valid_email_format(locator: Locator, **kwargs):
    text = locator.inner_text()
    matches = bool(re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", text))
    return {
        "matches": matches,
        "message": f"expected element text to be a valid email, got '{text}'" if not matches else "",
    }

expect.extend({"to_have_valid_email_format": to_have_valid_email_format})
```

```python
def test_profile_shows_valid_email(page):
    page.goto("/profile")
    expect(page.get_by_test_id("user-email")).to_have_valid_email_format()
```

```text
# expect.extend registers a new assertion in the SAME
# auto-retrying, timeout-aware machinery every built-in
# to_have_text/to_be_visible uses — this is different from just
# writing a plain `assert` helper function, because it retries
# until the timeout elapses rather than checking once instantly,
# which matters for anything checking a value that fills in
# asynchronously
```

## A domain-specific fixture: multi-tenant context switching

```python
# plugin.py
import pytest

@pytest.fixture
def tenant_page(browser, request):
    tenant = request.param
    context = browser.new_context(
        base_url=f"https://{tenant}.example.com",
        storage_state=f"auth/{tenant}.json",
    )
    page = context.new_page()
    yield page
    context.close()
```

```python
import pytest

@pytest.mark.parametrize("tenant_page", ["acme-corp", "globex"], indirect=True)
def test_dashboard_per_tenant(tenant_page):
    tenant_page.goto("/dashboard")
    expect(tenant_page.get_by_role("heading", name="Dashboard")).to_be_visible()
```

```text
# indirect=True routes the parametrize value through the
# fixture (as request.param) instead of directly into the test
# function — this is the standard pytest pattern for a fixture
# whose setup itself needs to vary per parametrized case, not
# just its return value
```

## Writing a custom pytest CLI option for environment selection

```python
# plugin.py
def pytest_addoption(parser):
    parser.addoption("--target-env", action="store", default="staging",
                      help="staging | production-readonly | local")

@pytest.fixture(scope="session")
def base_url(request):
    env = request.config.getoption("--target-env")
    urls = {
        "staging": "https://staging.example.com",
        "production-readonly": "https://example.com",
        "local": "http://localhost:3000",
    }
    return urls[env]
```

```bash
pytest tests/ --target-env=production-readonly
```

```text
# a first-class CLI flag (rather than an environment variable
# nobody remembers to set) makes environment targeting
# discoverable via `pytest --help`, and keeps it under pytest's
# normal config precedence rules (CLI > ini file > default)
```

## Full worked example: a small internal plugin package

```python
# my_company_playwright_fixtures/plugin.py
import pytest
from playwright.sync_api import expect

def pytest_addoption(parser):
    parser.addoption("--target-env", default="staging")

@pytest.fixture(scope="session")
def base_url(request):
    return {
        "staging": "https://staging.example.com",
        "production-readonly": "https://example.com",
    }[request.config.getoption("--target-env")]

@pytest.fixture
def page(browser, base_url):
    context = browser.new_context(base_url=base_url)
    pg = context.new_page()
    yield pg
    context.close()

def to_have_valid_email_format(locator, **kwargs):
    text = locator.inner_text()
    import re
    matches = bool(re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", text))
    return {"matches": matches, "message": "" if matches else f"'{text}' is not a valid email"}

expect.extend({"to_have_valid_email_format": to_have_valid_email_format})
```

## Exercise

1. Package a fixture your team reuses across multiple test files into a
   minimal installable pytest plugin with a `pytest11` entry point, and
   confirm a separate throwaway project can use it after `pip install -e`.
2. Write one custom assertion via `expect.extend` for a domain-specific
   check (a valid email, a currency format, a date range) and confirm it
   retries rather than checking once (test this by having the value appear
   after a short delay).
3. Build a `tenant_page`-style fixture using `indirect=True` parametrization
   and write a test that runs against two different simulated tenants.
4. Add a custom `--target-env` CLI option and a `base_url` fixture that
   reads it, and confirm running with two different `--target-env` values
   actually points the suite at two different base URLs.
