# 01 · What Is Playwright & Why E2E Testing

## What end-to-end testing actually checks

Unit tests verify a function in isolation. Integration tests verify a few
components wired together. **End-to-end (E2E) tests** verify the product the
way a user experiences it — a real (or realistically headless) browser
loading real pages, clicking real buttons, and asserting on what actually
renders. A unit test can be green while the login page is completely broken
in production, because nothing in it ever rendered a browser. An E2E test
catches exactly that class of bug: broken wiring between frontend, backend,
and the browser itself.

E2E tests are slower and more expensive to write and maintain than unit
tests, so they sit at the top of the testing pyramid — you write far fewer of
them, reserved for the critical user journeys (login, checkout, search) where
a real regression would be expensive to miss.

## What Playwright is

Playwright is a browser automation library, originally built by Microsoft
(by engineers who had previously worked on Puppeteer at Google), released in
2020. It drives real browsers — **Chromium, Firefox, and WebKit** — through
each engine's own automation protocol, with official APIs in Python,
JavaScript/TypeScript, Java, and .NET. This site uses the **Python API**
throughout; where the JavaScript/TypeScript API differs in a way worth
knowing, each module calls it out explicitly.

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    page.goto("https://example.com")
    print(page.title())
    browser.close()
```

```text
Example Domain
```

That's a complete, runnable Playwright script: launch a browser, open a page,
navigate, read something back, close the browser. Every module from here
builds on this shape.

## Why Playwright over Selenium

Selenium has been the default browser automation tool for over a decade and
still has a huge install base — you'll meet it in Level 4 when we cover
migration. Playwright was built afterward, informed by where Selenium suites
tend to hurt teams in practice:

| Pain point with Selenium-era suites | How Playwright addresses it |
|---|---|
| Tests sprinkled with manual `sleep()`/explicit waits because elements aren't "ready" yet | **Auto-waiting** — Playwright waits for an element to be attached, visible, stable, and receiving events before acting on it (Module 7) |
| One browser driver + binary version per browser, easy to get out of sync | Ships browser binaries itself; `playwright install` fetches versions tested against that Playwright release |
| Debugging a CI failure means re-reading log lines and guessing | **Trace Viewer** records a full timeline (DOM snapshots, network, console) you scrub through after the fact (Level 4) |
| Cross-browser testing needs separate driver setup per engine | One API drives Chromium, Firefox, and WebKit (covers Safari's engine) |
| Network mocking bolted on via external proxies | Built-in request interception (`page.route`) (Level 2) |
| Flaky selectors tied to CSS classes that change with every refactor | Role- and text-based locators designed to mirror how a user finds things (Module 4) |

None of this makes Selenium obsolete — plenty of production suites run on it
successfully, and Selenium's WebDriver protocol is a W3C standard supported
by every major browser vendor. Playwright's advantage is that these
ergonomics are the *default*, not something a team has to bolt on with
extra libraries and conventions.

## The three engines, one API

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    for browser_type in (p.chromium, p.firefox, p.webkit):
        browser = browser_type.launch()
        page = browser.new_page()
        page.goto("https://example.com")
        print(f"{browser_type.name}: {page.title()}")
        browser.close()
```

```text
chromium: Example Domain
firefox: Example Domain
webkit: Example Domain
```

Same script, three rendering engines. `chromium` covers Chrome and Edge,
`firefox` covers Firefox, and `webkit` is the engine behind Safari — running
this on Linux or Windows still exercises real WebKit-specific rendering and
JavaScript behavior, which is the closest you can get to Safari coverage
without a Mac in CI.

## Where Playwright fits in a project

Two ways teams commonly adopt it, and both are valid starting points:

1. **`pytest-playwright`** — a pytest plugin that wraps Playwright's Python
   API with fixtures (`page`, `browser`, `context`) and CLI flags
   (`--headed`, `--browser=firefox`). Best if your team already runs pytest
   for everything else, including unit and API tests. This site uses this
   path from Module 2 onward.
2. **`@playwright/test`** — a standalone JavaScript/TypeScript test runner
   built specifically for Playwright, with its own assertion library, parallel
   execution, and trace viewer integration out of the box. Most public
   Playwright documentation examples use this runner. If you or your team
   works primarily in JS/TS, this is usually the more natural entry point.

Both drive the identical underlying automation engine — locators, actions,
and auto-waiting behave the same either way. This site teaches the concepts
through the Python `pytest-playwright` path, and calls out `@playwright/test`
differences inline where they matter.

!!! note "JS/TS parity"
    The Python snippet at the top of this module, in `@playwright/test` style:
    ```javascript
    import { test, expect } from '@playwright/test';

    test('loads example.com', async ({ page }) => {
      await page.goto('https://example.com');
      await expect(page).toHaveTitle('Example Domain');
    });
    ```
    Note the `await` on every Playwright call — the JS/TS API is fully
    asynchronous. Python offers both a `sync_api` (used throughout this site)
    and an `async_api` for use inside `asyncio` code; `sync_api` is the
    default choice unless your project is already async.

## Exercise

1. Install Python 3.9+ if you don't already have it, and confirm with
   `python3 --version`.
2. Without installing anything yet, write down (in a text file or scratchpad)
   three real user journeys you'd want E2E-tested for an app you use daily —
   e.g. "sign up with email", "add item to cart and check out", "reset
   password". For each, note which browser engines you'd care about testing
   it in and why (e.g. a payment flow probably needs WebKit coverage because
   a large share of mobile traffic is Safari).
3. Read the auto-wait row in the comparison table again and think of a time
   you (or a teammate) added a `time.sleep(2)` to a test to fix flakiness.
   Keep that example in mind — Module 7 explains exactly what Playwright does
   instead and why it's more reliable.

Module 2 installs Playwright for real and gets a project scaffolded.
