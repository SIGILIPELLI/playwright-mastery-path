# 07 · Test Ownership & Team Practices

Technical practices (retries, tracing, sharding) only pay off if the team
around the suite treats it as owned, maintained infrastructure rather than a
pile of files nobody wants to touch. This module covers the process side.

## CODEOWNERS for test directories

```text
# .github/CODEOWNERS
/tests/checkout/     @checkout-team
/tests/search/       @search-team
/tests/shared/       @qa-platform-team
/e2e/conftest.py     @qa-platform-team
```

```text
# this does two things: it auto-requests the right reviewers
# on a PR touching those tests, and it makes ownership legible
# to anyone browsing the repo — "who do I ask about this
# flaky checkout test" becomes a lookup, not a Slack guess
```

## A review checklist specific to E2E tests

```markdown
## E2E test PR checklist
- [ ] Does this test assert on user-visible behavior, not implementation
      detail (internal state, specific class names)?
- [ ] Are preconditions seeded via API (Level 3), not clicked through the UI,
      unless the UI flow itself is what's under test?
- [ ] Does it use `expect()` polling assertions instead of
      `wait_for_timeout()`?
- [ ] Is the locator resilient to reasonable markup changes (role/label
      over brittle CSS/XPath, per Level 3's advanced locators)?
- [ ] If genuinely flaky-prone (e.g. third-party embed), is it marked and
      ticketed, not left silently retried forever?
```

```text
# a short, specific checklist catches the same handful of
# anti-patterns every time, far more reliably than relying on
# a reviewer to remember them all from memory on every PR
```

## Triage rotation for flaky and failing tests

```text
Weekly rotation (one engineer per week):
  1. Review the flaky_log.jsonl / dashboard (Level 3 module 5,
     Level 4 module 3) for tests rerun more than N times this week
  2. For each: reproduce locally, root-cause using Trace Viewer
     (Level 4 module 5), then either fix, quarantine with a ticket,
     or escalate to the owning team if it's outside E2E's control
  3. Post a short summary in the team channel: fixed, quarantined,
     escalated — with links
```

```text
# a named, rotating owner (not "whoever notices") is what
# actually gets flaky tests looked at — an unowned dashboard,
# however good, gets ignored
```

## Deprecating and deleting tests deliberately

```python
import pytest

@pytest.mark.skip(reason="Feature removed in PR #4821; test kept 1 sprint for reference, delete by 2026-09-30")
def test_legacy_export_format(page):
    ...
```

```text
# an unexplained @pytest.mark.skip is indistinguishable from
# "someone silenced this and forgot" a year later — a reason
# plus a concrete removal date turns a skip into a tracked,
# time-boxed decision instead of permanent dead weight
```

## Setting an explicit "new feature = new test" norm

```markdown
## Definition of Done (excerpt)
A feature PR is not done until:
  - at least one E2E test covers its primary happy path
  - the PR description links the new test file/function
  - if the feature has no meaningful E2E surface (a pure
    backend refactor), the PR explicitly says so instead of
    silently having no test
```

```text
# making "no test" a stated, visible choice rather than a
# silent gap is what prevents coverage from eroding as a
# codebase grows — reviewers can push back on an unstated gap,
# they can't push back on one they don't know exists
```

## Onboarding: a runnable example over a wiki page

```python
# tests/examples/test_annotated_example.py
"""
Read this file top to bottom before writing your first E2E test.
It is intentionally over-commented — production tests should not
look like this.
"""
from playwright.sync_api import Page, expect

def test_can_add_item_to_cart(page: Page, api_context):
    # 1. Seed preconditions via API, not UI clicks (Level 3 module 9)
    product = api_context.post("/api/products", data={"name": "Widget", "price": 9.99}).json()

    # 2. Navigate directly to the page under test
    page.goto(f"/products/{product['id']}")

    # 3. Act using role/label locators, never raw CSS unless unavoidable
    page.get_by_role("button", name="Add to cart").click()

    # 4. Assert on user-visible outcome via expect() (auto-retrying)
    expect(page.get_by_text("1 item in cart")).to_be_visible()
```

```text
# a real, runnable, heavily commented example test that lives
# in the repo (and gets updated when conventions change) stays
# accurate for longer than a wiki page describing the same
# conventions in prose, which drifts the moment the real
# pattern moves on
```

## Exercise

1. Add a `CODEOWNERS` entry mapping at least two test directories to
   different (real or hypothetical) team handles.
2. Write a short PR review checklist specific to your own suite's known
   anti-patterns (pull at least one item from a real past incident, not
   just this module's list).
3. Design a one-paragraph flaky-test triage rotation for your team size,
   including where the weekly summary gets posted.
4. Convert one `@pytest.mark.skip` in your codebase (or a hypothetical one)
   from an unexplained skip into one with a reason and a concrete
   removal/review date.
