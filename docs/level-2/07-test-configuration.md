# 07 · Test Configuration & Projects

## `pytest.ini` / `pyproject.toml` as the single source of truth

Command-line flags typed by hand drift between developers and CI. Baking
them into config means `pytest` alone reproduces the same run everywhere.

```ini
# pytest.ini
[pytest]
addopts =
    --browser chromium
    --headed=false
    --screenshot=only-on-failure
    --video=retain-on-failure
    --tracing=retain-on-failure
    -n auto
testpaths = tests
```

```text
# no output — every bare `pytest` invocation now implicitly
# includes all of these flags, so `pytest tests/test_login.py`
# still runs headless, traces on failure, etc.
```

`--screenshot`, `--video`, and `--tracing` are `pytest-playwright` options
(not stock pytest) — `only-on-failure`/`retain-on-failure` keep CI
artifacts small by discarding the media for tests that passed, while still
capturing full debugging evidence the moment something breaks.

## `browser_context_args` and `browser_type_launch_args` fixtures

These are the officially supported override points for how
pytest-playwright constructs the browser/context for every test.

```python
# conftest.py
import pytest

@pytest.fixture(scope="session")
def browser_type_launch_args(browser_type_launch_args):
    return {
        **browser_type_launch_args,
        "slow_mo": 50,       # ms delay between actions, for local debugging
    }

@pytest.fixture(scope="session")
def browser_context_args(browser_context_args):
    return {
        **browser_context_args,
        "viewport": {"width": 1440, "height": 900},
        "locale": "en-GB",
        "timezone_id": "Europe/London",
        "permissions": ["geolocation"],
        "geolocation": {"latitude": 51.5072, "longitude": -0.1276},
    }
```

```text
# no output — every test's context now launches at 1440x900,
# reports London as its locale/timezone/geolocation, and has
# geolocation permission pre-granted (skipping the native prompt)
```

Always spread the incoming fixture (`**browser_context_args`) rather than
returning a fresh dict — this lets other fixtures/plugins layer their own
overrides on top instead of one definition silently discarding another's.

## Base URL, so tests don't hardcode environments

```python
@pytest.fixture(scope="session")
def browser_context_args(browser_context_args):
    return {**browser_context_args, "base_url": "https://staging.example.com"}
```

```python
def test_login(page):
    page.goto("/login")   # resolves against base_url -> staging.example.com/login
```

```text
1 passed in 1.10s
```

With `base_url` set, `page.goto("/login")` and assertions like
`expect(page).to_have_url("/dashboard")` work with relative paths — this
is what makes the *same test file* runnable against staging, production,
or a local dev server, controlled entirely by config, never by editing
test code.

## Projects: running one suite in multiple configurations

The JS/TS Playwright test runner has a first-class `projects` array in
`playwright.config.ts` for this; in Python, the equivalent is combining
pytest markers with CLI-selectable fixture overrides, or running pytest
multiple times with different `--base-url`/env vars.

```python
# conftest.py
import os
import pytest

@pytest.fixture(scope="session")
def browser_context_args(browser_context_args):
    base_url = os.environ.get("BASE_URL", "https://staging.example.com")
    return {**browser_context_args, "base_url": base_url}
```

```bash
BASE_URL=https://staging.example.com pytest
BASE_URL=https://example.com pytest              # same suite, prod
```

```text
# no output for either — identical test files, environment
# variable alone decides which environment gets exercised
```

## Per-test overrides

A single test can still override context settings by requesting its own
`context`/`page` via a locally-scoped fixture rather than the global one.

```python
import pytest

@pytest.fixture
def mobile_page(browser):
    context = browser.new_context(
        viewport={"width": 390, "height": 844},
        user_agent=(
            "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) "
            "AppleWebKit/605.1.15"
        ),
        is_mobile=True,
        has_touch=True,
    )
    page = context.new_page()
    yield page
    context.close()

def test_mobile_nav_menu_is_hamburger(mobile_page):
    mobile_page.goto("https://example.com")
    from playwright.sync_api import expect
    expect(mobile_page.get_by_role("button", name="Menu")).to_be_visible()
```

```text
1 passed in 1.05s
```

Playwright ships `playwright.sync_api.playwright` device descriptors for
exact real-device emulation instead of hand-typing viewport/UA:

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    iphone_13 = p.devices["iPhone 13"]
    browser = p.chromium.launch()
    context = browser.new_context(**iphone_13)
```

```text
# no output — p.devices["iPhone 13"] expands to a full dict of
# viewport, user_agent, device_scale_factor, is_mobile, has_touch
```

## Exercise

1. Move any hardcoded `page.goto("https://...")` calls in an existing
   suite into a `base_url` context fixture, switching each call to a
   relative path.
2. Add a `pytest.ini` with `addopts` enabling `--screenshot=only-on-failure`
   and `--tracing=retain-on-failure`; deliberately break one test's
   assertion, run the suite, and locate the generated trace/screenshot
   artifact on disk.
3. Write a `mobile_context` fixture using `p.devices["Pixel 7"]` (or any
   device from `playwright.sync_api`'s device list) and duplicate one
   existing test to run against it, asserting mobile-specific UI (a
   hamburger menu, a bottom nav bar) appears.
4. Parametrize `BASE_URL` via an environment variable and run the same
   suite against two different public sites that share similar structure
   (or two branches of your own app, if available) without touching test
   code.
