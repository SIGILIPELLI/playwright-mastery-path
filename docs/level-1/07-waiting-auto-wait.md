# 07 · Waiting & Auto-Wait Philosophy

## The problem auto-waiting solves

A page is rarely "done" the instant `goto()` returns. A button might be in
the DOM but not yet interactive because a JavaScript bundle hasn't finished
attaching its click handler; an element might exist but sit behind a loading
overlay; a list might render an empty state for one frame before the real
data arrives. In tools without auto-waiting, this class of timing gap is
usually patched over with a hardcoded sleep:

```python
# the old, brittle way — don't do this
import time
page.click("#submit")
time.sleep(2)          # "hope 2 seconds is enough"
page.click("#confirm")
```

```text
# passes locally, fails intermittently in CI when the machine
# is slower, or wastes 2 real seconds on every run when it
# would have been ready in 200ms
```

This either wastes time (waiting longer than necessary, multiplied across
thousands of test runs) or is still too short under load (CI runners are
often slower than a developer's laptop), producing exactly the flaky,
"pass on retry" tests that erode trust in a suite.

## What Playwright checks before acting

Before most actions (`click`, `fill`, `check`, `hover`, and others),
Playwright runs a sequence of **actionability checks** on the target element
and retries the whole action until every check passes or the timeout
(30 seconds by default) elapses:

| Check | Meaning |
|---|---|
| Attached | The element exists in the DOM |
| Visible | Has non-empty bounding box, no `visibility: hidden` / `display: none` |
| Stable | Not currently mid-animation/transition (its bounding box is unchanged across two consecutive frames) |
| Receives events | Nothing else (a modal, an overlay, a tooltip) is on top of it at the point of interaction |
| Enabled | Not `disabled` (for `click`, `fill`, `check`, etc.) |
| Editable | Not `readonly` (for `fill` specifically) |

```python
page.get_by_role("button", name="Save").click()
```

```text
# no output on success — Playwright silently retries the
# actionability checks (often across just a handful of
# milliseconds) until Save is attached, visible, stable,
# unobstructed, and enabled, then performs the click
```

If the timeout is reached, the error tells you exactly which check kept
failing:

```text
TimeoutError: Timeout 30000ms exceeded.
Call log:
  - waiting for get_by_role("button", name="Save")
  -   locator resolved to <button disabled>Save</button>
  - element is not enabled
  - retrying click action
```

That's a fundamentally different debugging experience from a plain
`ElementNotInteractableException` with no history — you know immediately
that the button existed and was visible, but stayed disabled the whole time,
pointing you straight at a form-validation bug rather than a selector bug.

## Auto-waiting is not the same as "wait for navigation"

Auto-waiting covers interacting with elements already conceptually on the
page. It does **not** automatically know to wait for a full page navigation
or an async data fetch that hasn't started yet — for those, you wait on the
*result* you actually care about:

```python
# after an action that triggers navigation, chain the next
# locator call — Playwright will wait for it to appear on
# whatever page loads next:
page.get_by_role("link", name="Next").click()
expect(page.get_by_role("heading", name="Page 2")).to_be_visible()

# waiting for a specific element to prove data has loaded,
# instead of an arbitrary sleep or "networkidle":
page.get_by_role("button", name="Load more").click()
expect(page.locator(".item")).to_have_count(20)
```

```text
# no output — both patterns wait for a real, observable
# outcome rather than a fixed duration
```

This is the practical alternative to `wait_until="networkidle"` mentioned in
Module 3: wait for the specific piece of evidence that the state you care
about has arrived (an element count, a text value, a URL), not for network
traffic to go quiet — the former is precise and fast; the latter is a proxy
that can be wrong in both directions.

## Explicit waits, when you actually need one

```python
page.wait_for_selector(".spinner", state="hidden")
page.wait_for_url("**/checkout/confirmation")
page.wait_for_load_state("networkidle")
page.wait_for_timeout(1000)   # last resort — a literal fixed pause
page.wait_for_function("() => window.appReady === true")
```

```text
# no output — each returns once its condition is met or
# raises TimeoutError if it never is
```

`wait_for_timeout()` is the direct equivalent of the old `time.sleep()`
pattern and should be treated the same way: a debugging aid or a genuine
last resort (e.g. waiting out a fixed client-side animation with no
observable completion signal), never a routine part of a real test.
`wait_for_function()` is the more honest tool for "the app sets a flag when
it's ready" — you're still waiting for a real condition, just one exposed
through custom JavaScript state instead of the DOM.

## Setting timeouts

```python
page.set_default_timeout(10_000)                 # applies to this page's actions
page.get_by_role("button", name="Save").click(timeout=5000)   # overrides for one call
```

```text
# no output — a per-call timeout always overrides the page-level
# default for that single action
```

Lowering the default timeout in a fast, well-behaved suite makes a genuinely
broken test fail fast instead of hanging for 30 seconds; raising it for one
specific slow operation (a report export, a large upload) avoids loosening
the default for everything else.

## Exercise

1. On `https://www.saucedemo.com/`, log in, then intentionally slow the
   whole thing down with `slow_mo=100` and watch a click happen — confirm in
   your own observation that nothing appears to "wait" because the app is
   already fast; auto-waiting is invisible when there's nothing to wait for.
2. On the same site, click "Add to cart" for an item and immediately assert
   `expect(page.locator(".shopping_cart_badge")).to_have_text("1")` without
   any explicit wait — confirm it passes reliably across several runs.
3. Deliberately write `page.locator(".shopping_cart_badge").text_content() ==
   "1"` as a plain Python `assert` instead of using `expect`, run it 10
   times in a loop, and see whether it ever fails intermittently on your
   machine (it may not on this particular fast app — the point is
   understanding *why* it's a latent risk, not necessarily reproducing a
   failure).
4. Trigger a `TimeoutError` on purpose: try to click a button using a
   locator string that doesn't match anything real, catch the exception, and
   read the call log it prints — identify which actionability check it
   never got past (in this case, none — it never even attached).
5. Use `page.wait_for_url("**/cart.html")` after clicking the cart icon, and
   explain in your own notes why this is more precise than
   `wait_for_load_state("networkidle")` here.
