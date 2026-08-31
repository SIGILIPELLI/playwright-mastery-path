# 10 · Capstone — Production-Grade E2E Framework

The capstone assembles everything from Levels 1-4 into one cohesive
framework: a shared fixtures plugin, tiered browser/device coverage, custom
reporting, ownership metadata, and a CI pipeline that scales — the kind of
setup a real platform/QA team maintains for a company's E2E suite.

## Framework layout

```text
e2e-framework/
├── pyproject.toml                    # installable shared plugin (Level 4.9)
├── e2e_framework/
│   ├── plugin.py                     # fixtures: api_context, page, base_url
│   ├── assertions.py                 # expect.extend custom matchers (4.9)
│   └── reporters/
│       ├── json_reporter.py          # structured run output (4.3)
│       └── slack_reporter.py         # failure notifications (4.3)
├── tests/
│   ├── smoke/                        # tier 1, every PR, every browser
│   ├── checkout/                     # owned by checkout team
│   ├── search/                       # owned by search team
│   └── shared/                       # cross-cutting
├── .github/CODEOWNERS                # ownership (4.7)
└── .github/workflows/
    ├── pr.yml                        # smoke (multi-browser) + feature (chromium)
    └── nightly.yml                   # full matrix + a11y + visual (Level 3)
```

## `e2e_framework/plugin.py` — the shared foundation

```python
import pytest
from playwright.sync_api import Playwright

def pytest_addoption(parser):
    parser.addoption("--target-env", default="staging")

@pytest.fixture(scope="session")
def base_url(request):
    return {
        "staging": "https://staging.example.com",
        "production-readonly": "https://example.com",
    }[request.config.getoption("--target-env")]

@pytest.fixture(scope="session")
def api_context(playwright: Playwright, base_url):
    ctx = playwright.request.new_context(base_url=base_url)
    yield ctx
    ctx.dispose()

@pytest.fixture(scope="session")
def storage_state_path(api_context, tmp_path_factory):
    login = api_context.post("/api/login", data={
        "email": "e2e-runner@example.com", "password": "e2e-test-password",
    })
    assert login.ok
    path = tmp_path_factory.mktemp("auth") / "state.json"
    import json
    json.dump(api_context.storage_state(), open(path, "w"))
    return str(path)

@pytest.fixture
def page(browser, storage_state_path, base_url):
    context = browser.new_context(storage_state=storage_state_path, base_url=base_url)
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
    for oid in created:
        api_context.delete(f"/api/orders/{oid}")
```

```toml
# pyproject.toml
[project.entry-points.pytest11]
e2e_framework = "e2e_framework.plugin"
```

## `assertions.py` — a domain matcher used across every team's tests

```python
from playwright.sync_api import expect

def to_show_currency(locator, expected: float, **kwargs):
    text = locator.inner_text().strip()
    actual = float(text.replace("$", "").replace(",", ""))
    matches = abs(actual - expected) < 0.01
    return {"matches": matches, "message": "" if matches else f"expected ${expected}, got {text}"}

expect.extend({"to_show_currency": to_show_currency})
```

## `tests/smoke/test_critical_paths.py`

```python
import pytest
from playwright.sync_api import expect

@pytest.mark.smoke
def test_login_and_dashboard(page):
    page.goto("/dashboard")
    expect(page.get_by_role("heading", name="Dashboard")).to_be_visible()

@pytest.mark.smoke
def test_checkout_happy_path(page, make_order):
    order = make_order("Widget", qty=1)
    page.goto(f"/orders/{order['id']}")
    expect(page.get_by_test_id("order-total")).to_show_currency(9.99)
```

## `.github/CODEOWNERS`

```text
/tests/checkout/    @checkout-team
/tests/search/      @search-team
/tests/shared/      @qa-platform-team
/e2e_framework/      @qa-platform-team
```

## `.github/workflows/pr.yml`

```yaml
name: E2E (PR)
on: pull_request
jobs:
  smoke:
    strategy:
      matrix: { browser: [chromium, firefox, webkit] }
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e . && playwright install --with-deps
      - run: pytest -m smoke --browser ${{ matrix.browser }} --tracing=retain-on-failure
      - uses: actions/upload-artifact@v4
        if: failure()
        with: { name: traces-${{ matrix.browser }}, path: test-results/ }

  feature:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e . && playwright install --with-deps chromium
      - run: pytest -m "not smoke" --browser chromium --reruns 1
```

## `.github/workflows/nightly.yml`

```yaml
name: E2E (Nightly Full Matrix)
on:
  schedule:
    - cron: "0 3 * * *"
jobs:
  full-matrix:
    strategy:
      matrix:
        browser: [chromium, firefox, webkit]
        device: ["Desktop", "iPhone 13"]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e . && playwright install --with-deps
      - run: pytest --browser ${{ matrix.browser }} --device "${{ matrix.device }}"
      - run: pytest tests/shared/test_visual.py tests/shared/test_a11y.py --browser chromium
```

```text
# this is the whole picture assembled: an installable shared
# plugin (fixtures + custom assertions) so every team's tests
# start from the same foundation; tiered CI (fast smoke on
# every browser per PR, full matrix + visual/a11y nightly);
# ownership encoded in CODEOWNERS so failures route to the
# right team; and every technique from Levels 1-3 (auth reuse,
# API seeding, tracing, retries, visual/a11y gating) composed
# together rather than left as separate one-off examples
```

## Exercise

1. Assemble this structure (or an adapted version) in a real repository:
   an installable `e2e_framework` plugin package with `base_url`,
   `api_context`, `storage_state_path`, and `page` fixtures.
2. Add one custom `expect.extend` matcher relevant to your domain and use it
   in at least two tests across two different directories.
3. Set up the two-workflow CI split (PR: smoke multi-browser + feature
   chromium-only; nightly: full matrix + visual/a11y) and confirm both
   trigger correctly (a manual `workflow_dispatch` substitutes for waiting
   on the actual cron).
4. Add a `CODEOWNERS` file mapping at least two test directories to
   different owners, then deliberately break a test in each directory and
   confirm the right reviewers get auto-requested on the resulting PRs.
