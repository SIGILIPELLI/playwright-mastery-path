# 07 · Accessibility Testing

Playwright can't tell you your app is *usable* by someone on a screen
reader, but it can automatically catch the mechanical accessibility bugs —
missing labels, bad color contrast, invalid ARIA — that make it impossible.
The `axe-core` engine, via `axe-playwright-python`, is the standard tool for
this.

## Setup

```bash
pip install axe-playwright-python
```

```python
from axe_playwright_python.sync_playwright import Axe

def test_homepage_has_no_a11y_violations(page):
    page.goto("/")
    axe = Axe()
    results = axe.run(page)
    assert results.violations_count == 0, results.generate_report()
```

```text
# axe.run() injects the axe-core JS engine into the page,
# executes its full ruleset against the current DOM, and
# returns a structured result — results.generate_report()
# formats every violation with the failing selector, the rule
# that failed, and a link to the WCAG success criterion
```

## Scoping to a component and excluding known issues

```python
def test_checkout_form_a11y(page):
    page.goto("/checkout")
    axe = Axe()
    results = axe.run(
        page,
        context={"include": [["#checkout-form"]], "exclude": [[".third-party-widget"]]},
    )
    assert results.violations_count == 0, results.generate_report()
```

```text
# context lets you scope the scan to just the region you own —
# useful when a page embeds a third-party widget (a chat bubble,
# an ad) whose accessibility bugs aren't yours to fix and
# would otherwise fail every scan on that page permanently
```

## Reading a violation report

```text
Rule: color-contrast (serious)
  Elements affected: 1
    - <button class="btn-secondary">Cancel</button>
      Fix: Element has insufficient color contrast of 2.98
      (foreground #999999, background #ffffff, font size 14px,
      font weight normal). Expected contrast ratio of 4.5:1.

Rule: label (critical)
  Elements affected: 1
    - <input type="email" id="email-input">
      Fix: Form element does not have an implicit (wrapping)
      or explicit label.
```

```text
# each violation reports a severity (minor/moderate/serious/
# critical), the exact selector, and enough detail to fix it —
# the "label" rule here means a screen reader user has no idea
# what that input field is for at all, not just a cosmetic issue
```

## Asserting specific ARIA attributes directly

`axe` catches broad classes of problems; use direct assertions for
component-specific ARIA contracts you've committed to:

```python
def test_dropdown_aria_contract(page):
    page.goto("/settings")
    trigger = page.get_by_role("button", name="Select theme")
    expect(trigger).to_have_attribute("aria-expanded", "false")

    trigger.click()
    expect(trigger).to_have_attribute("aria-expanded", "true")

    listbox = page.get_by_role("listbox")
    expect(listbox).to_be_visible()
    expect(page.get_by_role("option", name="Dark")).to_have_attribute("aria-selected", "false")
```

## Keyboard navigation tests

Accessibility isn't just markup — a component that's only operable with a
mouse fails users who navigate by keyboard alone:

```python
def test_modal_is_keyboard_operable(page):
    page.goto("/settings")
    page.get_by_role("button", name="Delete account").click()

    dialog = page.get_by_role("dialog")
    expect(dialog).to_be_visible()

    # focus should move into the dialog automatically
    expect(page.get_by_role("button", name="Cancel")).to_be_focused()

    # Tab should cycle within the dialog, not escape to the page behind it
    page.keyboard.press("Tab")
    expect(page.get_by_role("button", name="Confirm delete")).to_be_focused()

    # Escape should close it
    page.keyboard.press("Escape")
    expect(dialog).not_to_be_visible()
```

```text
# to_be_focused() checks document.activeElement directly —
# this test would fail if the modal opened but left focus on
# whatever button triggered it, which is exactly the kind of
# bug that traps keyboard users behind a modal they can see
# but can't interact with
```

## Failing the build on new violations, not existing ones

Rolling an a11y gate into an existing large app usually means starting from
non-zero violations. Snapshot the current count and fail only on regressions
rather than blocking all progress on a single PR fixing everything at once:

```python
import json, os

BASELINE_PATH = "a11y_baseline.json"

def test_no_new_a11y_violations(page):
    page.goto("/")
    axe = Axe()
    results = axe.run(page)

    baseline = json.load(open(BASELINE_PATH)) if os.path.exists(BASELINE_PATH) else {"count": 0}
    assert results.violations_count <= baseline["count"], (
        f"{results.violations_count} violations, baseline allows {baseline['count']}\n"
        + results.generate_report()
    )
```

## How It Actually Works

`axe.run(page)` doesn't run inside Playwright's own process at all — it
takes the bundled `axe-core` JavaScript library and injects it into the
page via `Page.addScriptToEvaluateOnNewDocument`/`Runtime.evaluate` (the
same CDP mechanism `page.evaluate()` uses), so the entire accessibility
audit executes as real JavaScript inside the page's own execution context,
walking the live DOM and the browser's actual computed accessibility tree —
the identical data source `get_by_role()` reads from (Level 1 Module 4).
This is why axe can catch things a human visual check misses entirely
(a `color-contrast` failure is computed from real computed CSS values via
`getComputedStyle`, not a screenshot heuristic) and why results are only as
current as the DOM state at the moment `axe.run()` executes — a violation
introduced by a state change after the scan runs won't be caught until you
scan again.

`to_be_focused()` reads `document.activeElement` via the same
`Runtime.evaluate`/accessibility-tree channel every other assertion uses,
comparing it against the resolved node for your locator — which is
precisely why it's a meaningful check for keyboard-trap bugs: it verifies
the actual OS/browser-level focus target, the same state a real screen
reader or keyboard user's Tab key would be constrained by, not merely
whether an element exists in the DOM. `page.keyboard.press("Tab")` in turn
dispatches a real `Input.dispatchKeyEvent`, letting the browser's own
native focus-management logic (not Playwright) decide where focus lands
next — so this test only passes if the app's actual focus-trap
implementation is correct, not because Playwright is simulating tab order
itself.

## Exercise

1. Run `Axe().run(page)` against a real page in your app and read through
   the generated report for at least one genuine violation.
2. Fix that violation (add a label, adjust a contrast ratio) and confirm
   `violations_count` drops.
3. Write a keyboard-navigation test for one interactive component (a modal,
   a dropdown, a menu) asserting both focus placement on open and an Escape
   or Tab behavior.
4. Set up a baseline-count gate like the one above against a page with
   existing violations, and confirm it fails when you introduce one *new*
   violation beyond the baseline, but passes when the count stays the same.
