# 09 · Debugging Failing Tests

## Read the error message first

Playwright's timeout errors are unusually informative — resist the urge to
immediately add `page.wait_for_timeout(5000)` before reading what it says.

```text
playwright._impl._errors.TimeoutError: Locator.click: Timeout 30000ms exceeded.
Call log:
  - waiting for get_by_role("button", name="Submit")
    - locator resolved to 2 elements. Proceeding with the first one.
  - attempting click action
    - waiting for element to be visible, enabled and stable
    - element is not visible
  - retrying click action
    - waiting 20ms
  ...
```

This tells you three things without a single breakpoint: the locator
matched *two* elements (a strict-mode ambiguity worth fixing on its own),
it picked the first, and that first one is present but not visible —
pointing you at a CSS/layout issue or a modal covering it, not "the button
doesn't exist."

## `--headed` and `--slow-mo` for visual debugging

```bash
pytest tests/test_checkout.py::test_apply_discount --headed --slowmo 500
```

```text
# opens a real visible browser window, 500ms pause between
# every Playwright action, so you can watch exactly where the
# flow diverges from what you expect
```

## Playwright Inspector: step through interactively

```bash
PWDEBUG=1 pytest tests/test_checkout.py::test_apply_discount
```

```text
# launches the Playwright Inspector alongside the browser —
# a GUI with step/resume/pick-locator controls, pausing before
# the first action so you can single-step the whole test
```

The Inspector's "pick locator" button lets you click any element in the
live page and get back Playwright's own suggested locator for it — useful
both for debugging *and* for writing new locators without guessing.

## `page.pause()` for a targeted breakpoint

```python
def test_apply_discount(page):
    page.goto("/cart")
    page.get_by_placeholder("Discount code").fill("SAVE10")
    page.pause()   # opens the Inspector right here, mid-test
    page.get_by_role("button", name="Apply").click()
```

```text
# execution halts at page.pause(); the Inspector opens with the
# browser in exactly the state your test left it, so you can
# manually poke at the DOM before deciding what the next
# assertion or action should be
```

Unlike `--headed` alone, `page.pause()` lets you drop a breakpoint at the
*exact* line you're suspicious of, rather than watching the whole test
play out slowly from the start.

## Traces: the most powerful post-mortem tool

```ini
# pytest.ini
[pytest]
addopts = --tracing=retain-on-failure
```

```bash
pytest tests/test_checkout.py
playwright show-trace test-results/test_apply_discount/trace.zip
```

```text
# opens the Trace Viewer: a timeline scrubber over every action,
# with a DOM snapshot at each step, network requests, console
# logs, and (if enabled) screenshots/video, all correlated
```

A trace answers "what did the page actually look like right before this
failed" far better than a screenshot alone, because you can click any
step in the timeline and see the exact DOM state, hover to inspect
elements, and check the network tab for that instant — turning an
un-reproducible CI-only flake into something you can fully inspect after
the fact, once, without re-running anything (Level 4 covers Trace Viewer
in full depth, including trace groups and custom annotations).

## Console and page errors

```python
def test_checkout_flow(page):
    page.on("console", lambda msg: print(f"[console] {msg.type}: {msg.text}"))
    page.on("pageerror", lambda exc: print(f"[pageerror] {exc}"))
    page.goto("/checkout")
```

```text
[console] error: Failed to load resource: 404 (/api/promo-codes)
[pageerror] TypeError: Cannot read properties of undefined (reading 'code')
```

A test that fails on a UI assertion is sometimes really failing because of
a JS exception the page itself threw — wiring up `console`/`pageerror`
listeners (even just for local debugging, or permanently as an
`autouse` fixture that fails the test on any `pageerror`) surfaces the
real root cause instead of a confusing downstream symptom.

## Strict mode violations

```text
playwright._impl._errors.Error: Locator.click: Error: strict mode
violation: get_by_role("button", name="Delete") resolved to 3 elements:
    1) <button>Delete</button> aka locator("li:nth-child(1) >> button")
    2) <button>Delete</button> aka locator("li:nth-child(2) >> button")
    3) <button>Delete</button> aka locator("li:nth-child(3) >> button")
```

Strict mode (the default for action methods) refuses to guess which
element you meant — this is a feature, not friction, because it forces you
to scope the locator (`row.get_by_role("button", name="Delete")`) rather
than silently clicking whichever row happened to render first, which
would pass today and click the wrong row the moment the list re-orders.

## A debugging checklist

1. Read the full error and call log before touching code.
2. Reproduce with `--headed --slowmo`.
3. If it's timing/ordering-dependent, reproduce with tracing on and open
   the trace rather than guessing.
4. If a locator is ambiguous, scope it rather than reaching for `.first`
   as a reflex (`.first` can silently hide a real bug where two elements
   shouldn't both match).
5. Check `console`/`pageerror` output before assuming the test's
   assertion, not the app, is wrong.
6. Only add an explicit wait (`expect(...).to_be_visible()` before
   acting, or `page.wait_for_response(...)`) once you know exactly what
   condition you're waiting for — never a blind `wait_for_timeout`.

## Exercise

1. Take a test that currently passes and deliberately rename a locator's
   expected text so it fails; read the resulting error's call log line by
   line and describe, in a comment, exactly what it tells you.
2. Run that same failing test with `--tracing=on` (force on, not just on
   failure), open the trace with `playwright show-trace`, and step through
   the timeline to the failing action.
3. Introduce a genuine strict-mode violation (a locator matching two
   elements) into a test, observe the exact error Playwright gives, then
   fix it by scoping the locator to a parent container.
4. Add a permanent `autouse` fixture to `conftest.py` that listens for
   `pageerror` events and fails the test (via `pytest.fail(...)`) if any
   uncaught JS exception occurs during the test — run it against a page
   you know throws one, and confirm the fixture catches it.
