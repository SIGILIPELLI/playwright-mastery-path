# 04 · Network Interception & Mocking

## Why intercept network traffic in a test

Real end-to-end tests should hit a real backend most of the time — that's
the point of E2E. But some scenarios are impractical to trigger honestly:
a 500 error from a payment provider, a slow/flaky third-party API, a
response with data that doesn't exist yet in your test environment.
Playwright can intercept, inspect, modify, or fully replace any network
request the page makes, at the browser level, with no proxy setup needed.

## Reading requests and responses: `page.on`

```python
from playwright.sync_api import Page

def test_search_calls_expected_api(page: Page):
    requests = []
    page.on("request", lambda req: requests.append(req.url))

    page.goto("https://example.com/search")
    page.get_by_placeholder("Search").fill("laptop")
    page.get_by_placeholder("Search").press("Enter")
    page.wait_for_response(lambda r: "/api/search" in r.url)

    assert any("/api/search?q=laptop" in url for url in requests)
```

```text
1 passed in 1.12s
```

`page.on("request", ...)` and `page.on("response", ...)` are passive
observers — good for asserting *that* a call happened, with what
parameters, without changing anything about how the app behaves.

## `wait_for_response` for precise synchronization

```python
with page.expect_response(lambda r: "/api/search" in r.url and r.status == 200) as resp_info:
    page.get_by_placeholder("Search").press("Enter")
response = resp_info.value
data = response.json()
assert len(data["results"]) > 0
```

```text
# no output on success — response.json() parses the body,
# and the assertion passes silently
```

This is more reliable than an arbitrary `page.wait_for_timeout(1000)` after
pressing Enter: the test waits for the *specific* network event that
proves the search actually completed, not an arbitrary clock duration that
either wastes time or isn't long enough on a slow CI runner.

## Mocking a response with `page.route`

```python
import json
from playwright.sync_api import Page

def test_search_with_no_results(page: Page):
    def handle_search(route):
        route.fulfill(
            status=200,
            content_type="application/json",
            body=json.dumps({"results": []}),
        )

    page.route("**/api/search**", handle_search)
    page.goto("https://example.com/search")
    page.get_by_placeholder("Search").fill("nonexistent-item-xyz")
    page.get_by_placeholder("Search").press("Enter")

    from playwright.sync_api import expect
    expect(page.get_by_text("No results found")).to_be_visible()
```

```text
1 passed in 0.94s
```

`page.route(pattern, handler)` intercepts every request matching the glob
pattern *before* it reaches the network. Calling `route.fulfill(...)`
short-circuits it entirely — the real backend is never called — letting
you deterministically test the empty-state UI without needing a backend
that actually returns zero results for some magic query string.

## Simulating errors and slow networks

```python
def test_search_shows_error_banner_on_500(page: Page):
    page.route(
        "**/api/search**",
        lambda route: route.fulfill(status=500, body="Internal Server Error"),
    )
    page.goto("https://example.com/search")
    page.get_by_placeholder("Search").fill("laptop")
    page.get_by_placeholder("Search").press("Enter")

    from playwright.sync_api import expect
    expect(page.get_by_text("Something went wrong. Try again.")).to_be_visible()
```

```text
1 passed in 0.88s
```

```python
import time

def test_shows_loading_spinner_while_search_pending(page: Page):
    def slow_response(route):
        time.sleep(2)  # simulate backend latency
        route.continue_()

    page.route("**/api/search**", slow_response)
    page.goto("https://example.com/search")
    page.get_by_placeholder("Search").press("Enter")

    from playwright.sync_api import expect
    expect(page.get_by_role("status", name="Loading")).to_be_visible()
```

```text
1 passed in 2.31s
```

This is the only realistic way to test a 500-error banner or a loading
spinner deterministically — you cannot reliably make a real backend fail
or lag on demand, but you can always make the browser *believe* it did.

## Modifying a real response instead of replacing it

```python
def test_feature_flag_forced_on(page: Page):
    def add_feature_flag(route):
        response = route.fetch()
        body = response.json()
        body["features"]["new_checkout"] = True
        route.fulfill(response=response, json=body)

    page.route("**/api/config", add_feature_flag)
    page.goto("https://example.com")
```

```text
# no output — route.fetch() performs the real network request,
# then fulfill(response=..., json=...) replays it with one field
# patched, leaving everything else (headers, other fields) intact
```

`route.fetch()` followed by `route.fulfill(response=...)` is the pattern
for "mostly real, one field different" — safer than hand-writing a full
mock body that will silently drift out of sync with the real API shape.

## Stopping unwanted third-party calls

```python
page.route(
    "**/*.{google-analytics,doubleclick,facebook}*/**",
    lambda route: route.abort(),
)
```

```text
# no output — route.abort() fails the request outright; useful
# to keep tests fast and to avoid polluting real analytics with
# test traffic every CI run generates
```

## Exercise

Using `https://httpbin.org` as a stand-in backend and any simple page you
control (or `https://the-internet.herokuapp.com/`):

1. Write a test that navigates to a page making an XHR/fetch call, use
   `page.on("request", ...)` to log every request URL, and assert the page
   makes at least one request containing `/get` or `/api`.
2. Use `page.route` to intercept a request to `**/get**` and
   `route.fulfill()` a fixed JSON body of your choosing; reload the page
   and assert the mocked data renders instead of the real response.
3. Use `route.abort()` to block a specific request pattern and assert the
   page shows a fallback/error state instead of hanging.
4. Combine `page.expect_response(...)` with an action to assert a specific
   request fires with the correct query string when a filter/search input
   is used — without mocking anything, just observing.
5. Write `route.fetch()` + `route.fulfill(response=..., json=patched_body)`
   against a real `httpbin.org/get` call, patching one key in the returned
   JSON, and assert the patched value appears in the page (e.g. by writing
   a tiny HTML page that fetches and renders it, or by asserting via
   `response.json()` directly in the test).
