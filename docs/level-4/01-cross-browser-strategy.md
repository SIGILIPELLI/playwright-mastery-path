---
description: "Cross-Browser & Device Strategy — Running every test against every browser on every PR is rarely the right default — it multiplies CI time for coverage…"
---

# 01 · Cross-Browser & Device Strategy

Running every test against every browser on every PR is rarely the right
default — it multiplies CI time for coverage that's mostly redundant. This
module covers how to actually decide what runs where, and how to configure
multi-browser and multi-device runs when you do need them.

## Configuring multiple browsers with pytest-playwright

```ini
# pytest.ini
[pytest]
addopts = --browser chromium --browser firefox --browser webkit
```

```bash
pytest tests/test_checkout.py --browser chromium --browser webkit
```

```text
# passing --browser multiple times runs the entire selected
# test set once per named browser — a single logical test
# becomes N actual runs in the report, one per engine
```

## Device emulation

```python
import pytest

@pytest.fixture(scope="session")
def browser_context_args(browser_context_args, playwright):
    iphone = playwright.devices["iPhone 13"]
    return {**browser_context_args, **iphone}
```

```python
def test_mobile_nav_collapses_to_hamburger(page):
    page.goto("/")
    expect(page.get_by_role("button", name="Menu")).to_be_visible()
    expect(page.get_by_role("link", name="Products")).not_to_be_visible()
```

```text
# playwright.devices["iPhone 13"] bundles viewport size, user
# agent, device scale factor, and touch support into one dict —
# far more representative than just shrinking the viewport,
# since real mobile behavior often keys off touch capability
# and user agent sniffing, not just screen width
```

## A risk-based test tiering strategy

Not every test needs every browser. A workable default:

```text
Tier 1 — Smoke (every PR, every browser):
  login, checkout happy path, critical navigation
  → chromium + firefox + webkit, every push

Tier 2 — Feature coverage (every PR, one browser):
  full CRUD flows, form validation, most business logic
  → chromium only; browser-specific bugs here are rare
    enough to catch in tier 3 instead

Tier 3 — Full matrix (nightly / pre-release only):
  the entire suite × chromium, firefox, webkit, iPhone, Android
  → catches engine-specific rendering/timing differences
    without taxing every single PR
```

```python
# pytest markers implementing the tiers
import pytest

@pytest.mark.smoke
def test_login_succeeds(page): ...

@pytest.mark.feature
def test_order_can_be_partially_refunded(page): ...
```

```ini
# pytest.ini
[pytest]
markers =
    smoke: critical-path tests, run on every browser every PR
    feature: full coverage tests, chromium-only on PRs
```

```yaml
# CI: PRs run smoke on 3 browsers + feature on chromium only
# a nightly cron job runs the full "smoke or feature" set on
# the full device/browser matrix
```

## Deciding what to actually spend cross-browser coverage on

```text
Worth cross-browser testing:
  - CSS layout/flexbox/grid edge cases (real rendering differences)
  - Date/time input widgets (native <input type="date"> UI differs a lot)
  - File upload/download flows
  - Any code with a browser-specific workaround already in it

Rarely worth it:
  - Plain business logic already covered by chromium
  - API-only tests (no browser rendering involved at all)
  - Anything already covered by a unit/component test
```

## Full worked example: tiered browser matrix in CI

```yaml
# .github/workflows/e2e.yml
jobs:
  pr-smoke:
    if: github.event_name == 'pull_request'
    strategy:
      matrix:
        browser: [chromium, firefox, webkit]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt && playwright install --with-deps
      - run: pytest -m smoke --browser ${{ matrix.browser }}

  pr-feature:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt && playwright install --with-deps chromium
      - run: pytest -m feature --browser chromium

  nightly-full-matrix:
    if: github.event_name == 'schedule'
    strategy:
      matrix:
        browser: [chromium, firefox, webkit]
        device: ["Desktop", "iPhone 13", "Pixel 5"]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt && playwright install --with-deps
      - run: pytest --browser ${{ matrix.browser }} --device "${{ matrix.device }}"
```

## How It Actually Works

Passing `--browser` multiple times doesn't run one browser process
juggling three engines — `pytest-playwright` re-parametrizes the entire
collected test set once per named engine, and each engine gets its own
completely independent driver conversation: Chromium over its native CDP,
Firefox over Playwright's patched-Firefox remote protocol, WebKit over its
own remote-inspector-derived protocol (Level 1 Module 1). This is precisely
why "browser-specific" bugs exist at all and are worth budgeting real tier-3
coverage for — the three engines are genuinely different rendering and
layout implementations behind a unified Playwright API, so a CSS grid quirk
or a native `<input type="date">` rendering difference is a real engine
behavior difference the automation layer faithfully exposes, not a bug in
Playwright's abstraction leaking through.

Device emulation (`playwright.devices["iPhone 13"]`) only ever runs on the
Chromium engine underneath, even though it's applied via ordinary context
options — Chromium's `Emulation` CDP domain is what actually implements
touch-event translation, device-pixel-ratio rendering, and user-agent
overriding; Firefox and WebKit builds don't expose an equivalent set of
emulation primitives through Playwright, which is a good practical reason
the "full matrix" tier commonly emulates real devices only on Chromium and
relies on WebKit itself (rather than emulated "iPhone Safari") for genuine
Safari-engine coverage.

## 🔀 Related lessons on other tracks

- [AI Manager — 05 · Cross-Team AI Platform Strategy](https://sigilipelli.github.io/ai-manager-mastery-path/level-3/05-cross-team-ai-platform-strategy/)
- [Java Testing — 09 · Cross-Browser Testing](https://sigilipelli.github.io/java-testing-mastery-path/level-2/09-cross-browser-testing/)
- [Python Testing — 09 · Cross-Browser Testing](https://sigilipelli.github.io/python-testing-mastery-path/level-2/09-cross-browser-testing/)

## Exercise

1. Configure `--browser` to run one test file against chromium, firefox,
   and webkit and confirm the test report shows three separate runs.
2. Add an `iPhone 13` device-emulated context via `browser_context_args` and
   write a test asserting mobile-specific UI (a hamburger menu, a bottom
   nav bar) that only appears at that viewport/user-agent combination.
3. Add `smoke` and `feature` pytest markers to an existing suite, and split
   CI so PRs run `smoke` on 3 browsers but `feature` on chromium only.
4. Add a nightly scheduled workflow that runs the full marker set across a
   browser × device matrix, and explain in a comment why this tiering
   reduces average PR CI time without silently losing real coverage.
