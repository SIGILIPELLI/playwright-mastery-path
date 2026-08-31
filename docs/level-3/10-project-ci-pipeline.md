# 10 · Project — Full CI E2E Pipeline

This project ties together every Level 3 module into one working pipeline:
API-seeded data, reused auth state, visual and accessibility checks, sharded
CI execution with trace artifacts, and flaky-test tolerance — the shape of a
real production E2E setup, not a single isolated technique.

## Project structure

```text
e2e/
├── conftest.py
├── auth/
│   └── setup_auth.py
├── tests/
│   ├── test_smoke.py
│   ├── test_checkout.py
│   ├── test_visual.py
│   └── test_a11y.py
├── pytest.ini
└── .github/workflows/e2e.yml
```

## `conftest.py` — shared fixtures for the whole suite

```python
# conftest.py
import uuid
import pytest
from playwright.sync_api import Playwright

STORAGE_STATE = "auth_state.json"

@pytest.fixture(scope="session")
def api_context(playwright: Playwright):
    ctx = playwright.request.new_context(base_url="https://staging.example.com")
    yield ctx
    ctx.dispose()

@pytest.fixture(scope="session")
def storage_state_path(api_context):
    login = api_context.post("/api/login", data={
        "email": "e2e-runner@example.com",
        "password": "e2e-test-password",
    })
    assert login.ok, f"seed login failed: {login.status}"
    state = api_context.storage_state()
    import json
    with open(STORAGE_STATE, "w") as f:
        json.dump(state, f)
    return STORAGE_STATE

@pytest.fixture
def page(browser, storage_state_path):
    context = browser.new_context(storage_state=storage_state_path)
    pg = context.new_page()
    yield pg
    context.close()

@pytest.fixture
def make_order(api_context):
    created = []
    def _make(item: str, qty: int = 1):
        order = api_context.post("/api/orders", data={"item": item, "qty": qty}).json()
        created.append(order["id"])
        return order
    yield _make
    for order_id in created:
        api_context.delete(f"/api/orders/{order_id}")
```

## `test_smoke.py` — the fast, must-never-flake tier

```python
from playwright.sync_api import expect

def test_dashboard_loads(page):
    page.goto("/dashboard")
    expect(page.get_by_role("heading", name="Dashboard")).to_be_visible()

def test_navigation_links_present(page):
    page.goto("/dashboard")
    for label in ["Orders", "Customers", "Settings"]:
        expect(page.get_by_role("link", name=label)).to_be_visible()
```

## `test_checkout.py` — API-seeded, UI-verified

```python
from playwright.sync_api import expect

def test_edit_order_quantity(page, make_order):
    order = make_order("Widget", qty=2)
    page.goto(f"/orders/{order['id']}/edit")
    page.get_by_label("Quantity").fill("5")
    page.get_by_role("button", name="Save").click()
    expect(page.get_by_text("Order updated")).to_be_visible()

def test_order_list_shows_seeded_orders(page, make_order):
    make_order("Widget")
    make_order("Gadget")
    page.goto("/orders")
    expect(page.get_by_role("row")).to_have_count(3)  # 2 + header
```

## `test_visual.py` — masked, tolerant screenshots

```python
from playwright.sync_api import expect

def test_dashboard_visual(page):
    page.goto("/dashboard")
    expect(page).to_have_screenshot(
        "dashboard.png",
        mask=[page.get_by_test_id("last-login-time")],
        max_diff_pixel_ratio=0.01,
        animations="disabled",
    )
```

## `test_a11y.py` — gated on the existing baseline

```python
import json, os
from axe_playwright_python.sync_playwright import Axe

def test_dashboard_a11y(page):
    page.goto("/dashboard")
    results = Axe().run(page)
    baseline = json.load(open("a11y_baseline.json")) if os.path.exists("a11y_baseline.json") else {"count": 0}
    assert results.violations_count <= baseline["count"], results.generate_report()
```

## `pytest.ini`

```ini
[pytest]
addopts = --reruns 1 --reruns-delay 1 --tracing=retain-on-failure --screenshot=only-on-failure
markers =
    flaky: mark test as known-flaky with extra retries
```

## `.github/workflows/e2e.yml`

```yaml
name: E2E Tests
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - uses: actions/cache@v4
        id: pw-cache
        with:
          path: ~/.cache/ms-playwright
          key: playwright-${{ runner.os }}-${{ hashFiles('requirements.txt') }}
      - run: playwright install --with-deps chromium
        if: steps.pw-cache.outputs.cache-hit != 'true'
      - run: playwright install-deps chromium
        if: steps.pw-cache.outputs.cache-hit == 'true'
      - run: pytest e2e/tests --shard=${{ matrix.shard }}/3 --junitxml=results-${{ matrix.shard }}.xml
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: traces-shard-${{ matrix.shard }}
          path: test-results/
          retention-days: 7
```

```text
# putting it all together: auth happens once per session via
# the API (fast, no login form), each test seeds only the data
# it needs and cleans it up, visual/a11y checks are tolerant of
# noise but still gate real regressions, and CI shards the run
# across 3 parallel jobs with traces preserved only on failure —
# this is the shape a real team's E2E suite converges on once
# it outgrows "one file of tests that all log in every time"
```

## Exercise

1. Build out this structure (or adapt it to your own app) with a real
   session-scoped `storage_state_path` fixture and confirm every test file
   shares the one login.
2. Add a `make_order`-style factory fixture and use it from at least two
   different test files, confirming cleanup runs even when a test fails
   (temporarily force a failure and check the record is still deleted).
3. Wire the whole suite into a GitHub Actions workflow with sharding and
   trace-on-failure upload; push a branch with a deliberately broken test
   and download the resulting trace artifact.
4. Add one visual test and one accessibility test to the pipeline, confirm
   both pass on a clean run, then deliberately introduce a regression in
   each (a CSS change; a missing label) and confirm CI catches both.
