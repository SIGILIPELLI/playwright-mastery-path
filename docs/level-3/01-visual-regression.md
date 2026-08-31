# 01 · Visual Regression Testing

Functional assertions (`expect(locator).to_have_text(...)`) can't catch a
button that renders with the wrong padding, a broken CSS grid, or a font
that silently failed to load. Visual regression testing catches *that* class
of bug by comparing a screenshot of the current UI against a saved baseline,
pixel by pixel.

## `expect(page).to_have_screenshot()`

```python
from playwright.sync_api import Page, expect

def test_homepage_visual(page: Page):
    page.goto("/")
    expect(page).to_have_screenshot("homepage.png")
```

```text
# first run: no baseline exists yet, so Playwright writes
# homepage.png into a __screenshots__ (or configured) directory
# and marks the test as passed with a note that a baseline
# was created — commit that file to version control
#
# subsequent runs: Playwright takes a new screenshot, diffs it
# against the committed baseline pixel-by-pixel, and fails with
# a path to an actual/expected/diff image trio if they differ
# beyond the configured threshold
```

Run with `--update-snapshots` deliberately whenever a UI change is
intentional:

```bash
pytest tests/test_visual.py --update-snapshots
```

## Element-level screenshots

Full-page comparisons are noisy — a single pixel of unrelated content
shifting the whole page (a banner, an ad slot, a date) fails the entire
test. Scope screenshots to the component that actually matters:

```python
def test_pricing_card_visual(page: Page):
    page.goto("/pricing")
    card = page.get_by_test_id("pricing-card-pro")
    expect(card).to_have_screenshot("pricing-card-pro.png")
```

## Masking dynamic content

Timestamps, avatars, ads, and live counters change every run and will
always produce a diff. Mask them out rather than trying to freeze them:

```python
def test_dashboard_visual(page: Page):
    page.goto("/dashboard")
    expect(page).to_have_screenshot(
        "dashboard.png",
        mask=[page.get_by_test_id("last-updated"), page.get_by_test_id("avatar")],
        mask_color="#FF00FF",
    )
```

```text
# masked regions are painted a solid color before the
# screenshot is taken and before the diff is computed, so
# their actual pixel content never participates in the comparison
```

## Tolerance: threshold and max_diff_pixels

Anti-aliasing differs slightly between machines even with the same browser
build, so a zero-tolerance comparison is too brittle. Configure slack
explicitly rather than letting flakiness force people to skip visual tests
altogether:

```python
expect(page).to_have_screenshot(
    "homepage.png",
    max_diff_pixel_ratio=0.01,   # allow up to 1% of pixels to differ
    threshold=0.2,               # per-pixel color difference tolerance (0-1)
)
```

Set these once in `playwright.config` equivalents (or a shared fixture) so
every visual assertion in the suite uses the same tolerance, rather than
tuning each call site individually.

## Why baselines must be generated in CI, not locally

Font rendering, GPU rasterization, and even sub-pixel antialiasing differ
between macOS/Windows and the Linux containers CI runs on. A baseline
captured on a laptop will not match pixel-for-pixel in CI even when the UI
is identical.

```yaml
# .github/workflows/visual.yml (excerpt)
jobs:
  visual:
    runs-on: ubuntu-latest
    container: mcr.microsoft.com/playwright/python:v1.47.0-jammy
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt
      - run: pytest tests/test_visual.py --update-snapshots
        if: github.event_name == 'workflow_dispatch'
      - run: pytest tests/test_visual.py
        if: github.event_name != 'workflow_dispatch'
```

```text
# baselines are generated once via a manual workflow_dispatch
# run inside the same container image CI normally tests
# against, then committed — every regular CI run compares
# against that container-consistent baseline, never a
# developer's local screenshot
```

## Full worked example: a design-system smoke test

```python
# tests/test_visual_components.py
import pytest
from playwright.sync_api import Page, expect

COMPONENTS = ["button-primary", "button-disabled", "input-error", "modal-confirm"]

@pytest.mark.parametrize("component_id", COMPONENTS)
def test_component_visual(page: Page, component_id: str):
    page.goto(f"/storybook/iframe.html?id={component_id}")
    root = page.get_by_test_id("component-root")
    expect(root).to_be_visible()
    expect(root).to_have_screenshot(f"{component_id}.png", animations="disabled")
```

```text
# animations="disabled" freezes CSS transitions/animations at
# their end state before the screenshot, so a fade-in or
# spinner mid-animation never causes a false-positive diff
```

## Exercise

1. Add `expect(page).to_have_screenshot()` to a page in your own project and
   confirm it creates a baseline PNG on first run.
2. Deliberately change a CSS property (padding, color) on that page, re-run
   the test, and inspect the actual/expected/diff images Playwright writes
   for the failure.
3. Add a `mask=[...]` entry for one dynamic element and confirm the same
   change no longer causes an unrelated failure in that region.
4. Write a CI step (or a comment describing one) that only regenerates
   snapshots on manual `workflow_dispatch`, and explain in a comment why
   baselines generated on a contributor's laptop are unsafe to commit
   directly for a Linux CI runner.
