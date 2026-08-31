# 05 · Debugging with Trace Viewer

Earlier levels used tracing to grab a post-mortem on failure. This module
goes deeper into the Trace Viewer itself — the panels, how to read them
precisely, and workflows for the hardest class of bug: something that only
fails in CI and never reproduces locally.

## Capturing the richest possible trace

```python
context.tracing.start(
    screenshots=True,
    snapshots=True,
    sources=True,
    title="checkout-flow",
)
page.goto("/checkout")
# ... test steps ...
context.tracing.stop(path="trace.zip")
```

```text
# screenshots: a JPEG after every action, for a quick visual scrub
# snapshots: a full DOM + CSS snapshot per action, letting the
#   viewer render a pixel-faithful reconstruction you can
#   inspect/hover/right-click, not just a static image
# sources: embeds your test's source code, so the viewer can
#   highlight exactly which line produced each action
```

## Reading the Actions panel

```text
▼ page.goto("/checkout")           120ms
▼ locator.fill("Email")             45ms
▼ locator.click("Place order")     2.3s   ← unusually long
  ▸ waiting for element to be visible, enabled and stable
  ▸ scrolling into view if needed
  ▸ element is visible, enabled and stable
  ▸ performing click action
```

```text
# each action's duration is the first thing to scan for — a
# click action taking 2.3s when everything before it took
# under 200ms means the click itself waited on something
# (an element not yet stable, or a re-render loop); expanding
# it shows the exact actionability checks Playwright ran and
# in what order, pinpointing which one took the time
```

## Network tab: correlating a UI stall with a slow request

```text
Network
  GET  /api/cart               200   340ms
  POST /api/checkout/validate  200   1,850ms   ← the real bottleneck
  GET  /api/shipping-options   200   210ms
```

```text
# clicking a network entry jumps the DOM snapshot to that exact
# moment, so you can confirm the UI genuinely was waiting on
# that specific response — this is often what an "unusually
# long click" in the Actions panel actually was waiting on
```

## Console tab and uncaught exceptions

```text
Console
  [warning] React does not recognize the `isActive` prop on a DOM element
  [error]   Uncaught TypeError: Cannot read properties of undefined (reading 'total')
              at CartSummary.jsx:42
```

```text
# a JS exception logged here, timestamped against the same
# timeline as the failing action, is often the actual root
# cause of a UI assertion failure that looked purely visual —
# check this before assuming your selector or assertion is wrong
```

## The "before/after" snapshot toggle for a single action

```text
# each action row has a small camera icon toggle between the
# DOM snapshot immediately BEFORE and immediately AFTER that
# action executed — for a click that "didn't seem to do
# anything," comparing before/after directly answers whether
# the DOM changed at all, versus changed but not in the way
# the test expected
```

## Comparing a CI trace against a local trace for the same test

```bash
# download the CI-uploaded trace artifact (Level 3's --tracing setup)
playwright show-trace ci-trace.zip

# reproduce locally with an identical trace for comparison
pytest tests/test_checkout.py::test_apply_discount --tracing=on
playwright show-trace test-results/.../trace.zip
```

```text
# side-by-side, differences in viewport size, timing between
# actions, or a network response that differs (a feature flag
# on in CI but not locally, different seeded data) are usually
# visible directly in the two Network/DOM panels — this is the
# most reliable way to debug a "only fails in CI" report
# without adding print statements and re-running CI repeatedly
```

## Trace groups: organizing a long, multi-page-object test

```python
with context.tracing.group("Login"):
    page.goto("/login")
    page.get_by_label("Email").fill("user@example.com")
    page.get_by_role("button", name="Sign in").click()

with context.tracing.group("Add item to cart"):
    page.goto("/products/42")
    page.get_by_role("button", name="Add to cart").click()

with context.tracing.group("Checkout"):
    page.goto("/checkout")
    page.get_by_role("button", name="Place order").click()
```

```text
# tracing.group() nests a labeled, collapsible section in the
# Actions panel — a 40-step end-to-end test becomes 3-4
# collapsible groups instead of a flat unreadable list,
# turning "which of these 40 actions is the failing one" into
# "which group failed" as the first filter
```

## Exercise

1. Capture a trace with `screenshots=True, snapshots=True, sources=True` for
   a multi-step test and open it with `playwright show-trace`; identify the
   single longest action and expand it to see why.
2. Find a network request in the trace that correlates with a UI wait and
   click it to jump the DOM snapshot to that moment.
3. Wrap a multi-step test in 2-3 `tracing.group()` blocks and confirm the
   Actions panel shows them as collapsible sections.
4. Deliberately introduce a difference between a "local" and "CI-like" run
   (a different viewport, a stubbed vs. real API response) that changes
   test behavior, capture a trace of each, and describe in a comment
   exactly which panel revealed the difference.
