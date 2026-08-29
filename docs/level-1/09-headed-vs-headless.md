# 09 · Running Headed vs. Headless

## The difference

A **headed** browser renders an actual visible window you can watch — the
same Chromium/Firefox/WebKit UI a human would see. A **headless** browser
runs the identical rendering and JavaScript engine with no visible window at
all, communicating only through Playwright's automation protocol. Headless
is not a lesser or simulated version of the browser — it's the same engine,
just without drawing pixels to a screen, which makes it faster and lighter
to run on a server with no display attached (which is exactly what most CI
runners are).

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)   # default
    page = browser.new_page()
    page.goto("https://example.com")
    print(page.title())
    browser.close()
```

```text
Example Domain
```

```python
with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)  # opens a real window
    page = browser.new_page()
    page.goto("https://example.com")
    page.wait_for_timeout(3000)   # gives you time to actually see it
    browser.close()
```

```text
# no console output — a visible Chromium window opens, navigates,
# stays open for 3 seconds, then closes
```

`headless=True` is the default for every Playwright launch — you have to opt
into headed mode explicitly.

## When to use each

| Situation | Mode | Why |
|---|---|---|
| Writing a new test, figuring out locators | Headed | You want to *see* what's happening while iterating |
| A test fails locally and you don't know why | Headed + `slow_mo` | Watching it run at human speed often shows the bug immediately |
| Running the full suite locally before pushing | Headless | Faster, and matches what CI will actually run |
| CI pipeline | Headless | No display server available; also meaningfully faster |
| Debugging a CI-only failure | Trace Viewer (Level 4), not headed mode | You can't watch a CI machine's screen live — a recorded trace is the CI equivalent of "headed" debugging |

A good default workflow: write and debug new tests headed, but always run
the full suite headless before considering it done — a test that only
passes headed and fails headless (or vice versa) is telling you something
timing-related is being masked by the visible rendering, and that's worth
chasing down rather than shipping.

## `pytest-playwright`'s `--headed` flag

Since Module 2 you've had `pytest-playwright` installed, which wires
headed/headless into pytest's CLI directly instead of hardcoding it in your
script:

```bash
pytest                    # headless — the default
pytest --headed           # headed, for this run only
pytest --headed --slowmo 300
pytest --browser firefox --headed
```

```text
tests/test_login.py::test_valid_login PASSED                             [100%]
```

Nothing in your test code needs to know which mode is active — the `page`
fixture `pytest-playwright` provides (covered fully in Level 2) is
constructed according to these CLI flags. This is a strong argument for
using `pytest-playwright`'s fixtures over manually calling
`p.chromium.launch()` yourself: switching modes becomes a command-line
concern, not a code change.

## Headless quirks worth knowing about

Headless and headed are the same rendering engine, but a small number of
things can differ:

- **Window size** — headless defaults to a fixed viewport (1280×720 for
  Chromium via Playwright) unless you set one explicitly; a headed window's
  actual OS window size doesn't automatically match. Always set the viewport
  explicitly for anything layout-sensitive:

```python
context = browser.new_context(viewport={"width": 1440, "height": 900})
```

```text
# no output — every page created from this context now
# renders at exactly 1440x900 regardless of headed/headless
```

- **Media codecs / fonts** — a CI Docker image can be missing system fonts or
  codecs that your local machine has, changing text rendering enough to
  break a visual regression test (Level 3) even though the CSS is identical.
  This is a CI environment issue, not a headed/headless issue per se, but it
  surfaces most often when comparing local headed screenshots against CI
  headless ones.
- **New headless mode** — modern Chromium ships a newer headless
  implementation (`"new"`) that is closer to headed Chrome's rendering than
  the older headless mode was; Playwright uses the modern implementation by
  default in current versions, which is part of why headless/headed parity
  is much better today than it used to be.

## Watching a single failing test, quickly

```bash
pytest tests/test_checkout.py::test_checkout_with_expired_card --headed --slowmo 500 -s
```

```text
tests/test_checkout.py::test_checkout_with_expired_card FAILED           [100%]
```

Running one specific test (`::test_name`) headed with `--slowmo` is usually
faster than adding print statements — you watch exactly where the flow
diverges from what you expected, in real time.

## Exercise

1. Write a small script (not a pytest test yet) that launches headed with
   `slow_mo=250`, logs into `https://www.saucedemo.com/`, and adds two items
   to the cart — watch it run and confirm you can follow each step visually.
2. Run the same logic headless and confirm it completes successfully with no
   visible window — time both runs roughly (e.g. wrapping with `time
   python3 script.py`) and note the difference.
3. Convert the script into a `pytest-playwright` test using the `page`
   fixture (a preview — Level 2 covers fixtures properly) and run it three
   ways: `pytest`, `pytest --headed`, `pytest --headed --slowmo 300`.
4. Set an explicit `viewport` of `{"width": 375, "height": 812}` (a phone
   size) on a new context, navigate to your app, and take a screenshot —
   confirm the layout visibly reflows for mobile width.
5. Deliberately introduce a race condition — click a button and immediately
   read a value without an `expect()` wait — and check whether it's more
   likely to fail headless or headed on your machine. Write a sentence in
   your notes on why headed mode's slightly different timing characteristics
   can occasionally hide a bug that headless (and CI) will expose.
