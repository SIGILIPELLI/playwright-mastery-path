# 03 · Your First Script

## The core objects

Playwright's object model has four layers, each nested inside the last:

```text
Playwright  →  Browser  →  BrowserContext  →  Page
```

- **Playwright** — the entry point; gives you access to `chromium`,
  `firefox`, `webkit`.
- **Browser** — one running browser process (e.g. one Chromium instance).
- **BrowserContext** — an isolated "browser profile" inside that process,
  with its own cookies, local storage, and cache. Contexts are cheap to
  create and fully isolated from each other — this is how Playwright runs
  independent, parallel-safe tests without launching a new browser process
  per test.
- **Page** — one tab/window inside a context. Most of the API you use daily
  (`goto`, `click`, `fill`, locators) is a method on `Page`.

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    context = browser.new_context()
    page = context.new_page()

    page.goto("https://example.com")
    print("Title:", page.title())
    print("URL:", page.url)

    page.screenshot(path="example.png")

    context.close()
    browser.close()
```

```text
Title: Example Domain
URL: https://example.com/
```

Running this also writes `example.png` to your working directory — a full
screenshot of the rendered page, useful for a quick visual sanity check or
for attaching evidence to a bug report.

## Launch options worth knowing immediately

```python
browser = p.chromium.launch(
    headless=False,   # show the actual browser window
    slow_mo=250,      # add a 250ms delay between actions, for watching along
)
```

```text
# no console output — a visible Chromium window opens, navigates,
# and each action pauses briefly before the next one
```

`headless=False` and `slow_mo` are your two best debugging tools while
writing a new script: watch it happen at human speed instead of reading a
stack trace after the fact. Module 9 covers headed vs. headless in depth,
including when each belongs in your actual workflow.

## `goto` and navigation waiting

```python
page.goto("https://example.com")
```

`goto` waits for the `load` event by default before returning — the page's
HTML, CSS, images, and other resources have finished loading. You can ask
for a different point instead:

```python
page.goto("https://example.com", wait_until="domcontentloaded")
```

```text
# returns as soon as HTML is parsed, without waiting for
# images/stylesheets/subframes to finish loading
```

| `wait_until` value | Waits for |
|---|---|
| `"load"` (default) | The full `load` event — most resources finished |
| `"domcontentloaded"` | HTML parsed and DOM built, before images/styles finish |
| `"networkidle"` | No network connections for 500ms — useful for heavy SPAs, but discouraged as a default (see below) |
| `"commit"` | Navigation committed, HTML not yet parsed — rarely what you want |

`"networkidle"` is tempting for single-page apps that keep polling in the
background, but polling that never truly goes idle (analytics beacons,
websockets) can make it wait the full navigation timeout for nothing. Prefer
waiting for a specific element to appear (Module 7) over `"networkidle"` —
it's both faster and more precise about what you actually need loaded.

## Reading data back from the page

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    page.goto("https://example.com")

    heading_text = page.locator("h1").text_content()
    paragraph_text = page.locator("p").first.text_content()
    html_snippet = page.locator("div").first.inner_html()

    print("Heading:", heading_text)
    print("Paragraph starts with:", paragraph_text[:40])

    browser.close()
```

```text
Heading: Example Domain
Paragraph starts with: This domain is for use in illustrative
```

`text_content()` returns the rendered text; `inner_html()` returns the raw
markup inside the element. Module 4 covers `locator()` itself in depth — for
now, treat it as "find the element(s) matching this description."

## Closing things down (and why context managers matter)

```python
with sync_playwright() as p:
    browser = p.chromium.launch()
    ...
    browser.close()
```

`sync_playwright()` used as a context manager (`with ... as p:`) guarantees
the underlying driver process is shut down cleanly even if your script
raises an exception partway through. Forgetting to close a `Browser`
explicitly leaks the OS process; in a long-running test suite this shows up
as accumulating zombie Chromium processes on a CI runner. Always pair every
`launch()` with a `close()`, or better, let a fixture do it for you — which
is exactly what `pytest-playwright`'s `page` fixture does starting next
module.

## Exercise

1. Write a script `first_script.py` that launches Chromium headed with
   `slow_mo=300`, navigates to `https://books.toscrape.com/` (a public site
   built for scraping/automation practice), and prints the page title.
2. Extend it to read the text of the first product's `<h3>` title on the
   page (`page.locator("h3").first.text_content()`) and print it.
3. Take a screenshot of the full page and save it as `books_home.png`.
4. Change `wait_until` on `goto` to `"domcontentloaded"` and confirm the
   script still works — this site doesn't have heavy async loading, so the
   difference won't be visually obvious, but get comfortable changing the
   parameter.
5. Deliberately forget to call `browser.close()` and comment out the `with`
   block's indentation so `sync_playwright()` isn't used as a context
   manager. Run it, then check your process list (`ps aux | grep -i chrom`
   or your OS's task manager) — confirm you can see a leftover browser
   process, then fix the script back and confirm it's cleaned up.
