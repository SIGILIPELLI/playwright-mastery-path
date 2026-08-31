# 06 · Playwright Architecture Internals

Knowing *why* Playwright behaves the way it does — one process, a single
WebSocket-driven protocol, auto-waiting baked into the client rather than
the browser — makes its error messages and edge cases far less mysterious.

## The out-of-process browser model

```text
┌─────────────────┐         WebSocket / pipe        ┌──────────────────┐
│  Python process  │ ───────────────────────────────▶│  Browser process  │
│  (your test)      │◀─────────────────────────────── │  (Chromium/FF/WK) │
│  playwright client │        CDP / BiDi-like protocol │  actual rendering │
└─────────────────┘                                  └──────────────────┘
```

```text
# unlike Selenium's WebDriver (HTTP request/response per
# command), Playwright keeps one persistent connection to a
# separate browser process and communicates over a
# message-based protocol — closer to how Chrome DevTools
# itself talks to the browser than to a REST API
```

Chromium and WebKit are driven through a Playwright-patched build with
extra automation hooks baked in; Firefox uses a similar patched build via a
juggler protocol layer. This is *why* `playwright install` downloads its own
browser binaries rather than reusing your system Chrome — the automation
hooks aren't present in a stock browser release.

## Why auto-waiting lives in the client, not the browser

```python
page.get_by_role("button", name="Submit").click()
```

```text
# before performing the click, the Playwright client itself:
#  1. resolves the locator to a live DOM node reference
#  2. re-checks the actionability conditions (attached,
#     visible, stable, receives events, enabled) in a polling
#     loop — this loop runs in the CLIENT, re-querying the
#     browser each iteration, not as a single browser-side wait
#  3. only dispatches the actual click once all conditions
#     hold on the same iteration
```

This is why a `Locator` is a *description* that gets re-resolved every time
you act on it — `page.locator("...")` alone does not query the DOM at all;
only calling `.click()`, `.fill()`, or an `expect()` on it does. This is also
why chaining a `Locator` (`row.get_by_role(...)`) is cheap and doesn't fail
even if `row` doesn't exist yet at the time you write that line — the actual
query happens lazily, at the moment of action.

## Contexts vs. pages vs. browsers

```text
Browser        — one actual browser process (chromium.launch())
  BrowserContext — an isolated "incognito-like" session: its own
                   cookies, localStorage, cache — cheap to create,
                   the recommended unit of test isolation
    Page         — one tab within that context
```

```python
browser = playwright.chromium.launch()      # expensive: ~200-500ms
context = browser.new_context()             # cheap: a few ms
page = context.new_page()                   # cheap: a few ms
```

```text
# this is why the standard advice is "one browser per test
# session (or worker), one fresh context per test" rather than
# launching a whole new browser per test — a browser process
# is the expensive resource; a context is deliberately cheap
# because it's the unit meant to be thrown away constantly
```

## Strict mode: a deliberate design choice, not a limitation

```text
# all "action" methods (click, fill, check, etc.) enforce
# strict mode: they throw if the locator resolves to more than
# one element, rather than silently acting on the first match —
# this traces back to the same underlying design principle as
# auto-waiting: prefer a loud, immediate, specific failure
# over a test that "usually" does the right thing
```

`.first`, `.last`, and `.nth()` exist specifically to opt out of strict mode
when multiplicity is genuinely expected — using them as a reflex fix for
every strict-mode error defeats the feature.

## The protocol underneath: CDP, and why `new_cdp_session` exists

```python
client = page.context.new_cdp_session(page)
client.send("Network.emulateNetworkConditions", {...})
```

```text
# Playwright's own driver already speaks CDP to Chromium
# internally for standard actions — new_cdp_session exposes a
# raw channel to that same protocol for capabilities Playwright's
# high-level API doesn't wrap (like network/CPU throttling,
# covered in Level 3) — you're not going around Playwright here,
# you're using the same transport it already relies on
```

## Why headless and headed can behave subtly differently

```text
# headless mode historically ran a different code path inside
# the browser (no real compositor/GPU pipeline by default) —
# modern Chromium's "new headless" mode (Playwright's default
# since v1.9x-era Chromium) closes most of this gap, but GPU-
# dependent rendering (WebGL, some video codecs, certain font
# hinting) can still differ — this is why a visual regression
# test flaking only in CI's headless run, never locally headed,
# is a real and known category of difference, not always a bug
# in the test
```

## Exercise

1. Explain in your own words (a short comment block is fine) why
   `page.locator("#foo")` doesn't throw immediately even if `#foo` doesn't
   exist on the page yet, but `page.locator("#foo").click()` will wait then
   throw.
2. Open a CDP session with `new_cdp_session` and call any raw CDP command
   not exposed by Playwright's high-level API (e.g. `Page.printToPDF` or
   `Network.getCookies`), confirming you get a real response back.
3. Create a `BrowserContext`, time how long `new_context()` takes versus
   `chromium.launch()`, and confirm the order-of-magnitude difference that
   justifies reusing one browser across many contexts.
4. Trigger a genuine strict-mode violation, then instead of reaching for
   `.first`, rewrite the locator to be unambiguous — and write a comment
   explaining a case where `.first` genuinely would be the correct choice.
