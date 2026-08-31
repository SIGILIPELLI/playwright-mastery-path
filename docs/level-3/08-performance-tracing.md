# 08 · Performance & Tracing

Playwright can capture real browser performance data — not just "did it
load" but "how long did it take, and where did the time go" — using the
Chrome DevTools Protocol (CDP) and the browser's own Performance/Navigation
Timing APIs, no external tooling required.

## Navigation timing via `page.evaluate`

```python
def test_homepage_load_performance(page):
    page.goto("/")
    timing = page.evaluate("""() => {
        const nav = performance.getEntriesByType('navigation')[0];
        return {
            domContentLoaded: nav.domContentLoadedEventEnd - nav.startTime,
            loadComplete: nav.loadEventEnd - nav.startTime,
            ttfb: nav.responseStart - nav.startTime,
        };
    }""")
    assert timing["ttfb"] < 500, f"TTFB too slow: {timing['ttfb']}ms"
    assert timing["domContentLoaded"] < 2000, f"DCL too slow: {timing['domContentLoaded']}ms"
```

```text
# performance.getEntriesByType('navigation') is a standard
# browser API returning precise timestamps for every phase of
# page load — running it through page.evaluate() gets real
# browser-measured numbers back into Python as plain data
```

## Core Web Vitals: LCP, CLS, and friends

```python
def test_largest_contentful_paint(page):
    page.goto("/")
    lcp = page.evaluate("""() => new Promise(resolve => {
        new PerformanceObserver((list) => {
            const entries = list.getEntries();
            resolve(entries[entries.length - 1].startTime);
        }).observe({ type: 'largest-contentful-paint', buffered: true });
    })""")
    assert lcp < 2500, f"LCP too slow: {lcp}ms"
```

```text
# LCP only fires once, asynchronously, so this wraps a
# PerformanceObserver in a Promise and awaits it inside
# page.evaluate() — buffered: true lets it retrieve an entry
# that already happened before the observer was attached
```

## CDP session: network and CPU throttling

```python
def test_page_usable_on_slow_network(page):
    client = page.context.new_cdp_session(page)
    client.send("Network.emulateNetworkConditions", {
        "offline": False,
        "downloadThroughput": 400 * 1024 / 8,   # 400kbps
        "uploadThroughput": 400 * 1024 / 8,
        "latency": 400,
    })
    page.goto("/")
    expect(page.get_by_role("heading", name="Dashboard")).to_be_visible(timeout=15000)
```

```text
# new_cdp_session() opens a raw Chrome DevTools Protocol
# channel — Network.emulateNetworkConditions throttles the
# actual network stack the browser uses, which is far more
# realistic than route-based delay injection because it
# affects every resource (JS, CSS, images) proportionally,
# not just the ones you've explicitly mocked
```

```python
def test_page_usable_on_throttled_cpu(page):
    client = page.context.new_cdp_session(page)
    client.send("Emulation.setCPUThrottlingRate", {"rate": 4})  # 4x slower CPU
    page.goto("/dashboard")
    expect(page.get_by_text("Loading complete")).to_be_visible(timeout=10000)
```

## Tracing for performance, not just failures

Level 2 covered tracing for debugging failures; the same trace also records
a full CPU/network timeline useful for spotting performance regressions:

```python
def test_checkout_flow_traced(page, context):
    context.tracing.start(screenshots=True, snapshots=True, sources=True)
    page.goto("/checkout")
    page.get_by_role("button", name="Place order").click()
    expect(page.get_by_text("Order confirmed")).to_be_visible()
    context.tracing.stop(path="checkout-trace.zip")
```

```bash
playwright show-trace checkout-trace.zip
```

```text
# the Trace Viewer's timeline includes a CPU/network graph
# alongside the action log — a long gap between two actions
# with no network activity underneath usually means expensive
# synchronous JS, which the "Call" and "Network" tabs at that
# timestamp can pinpoint
```

## Full worked example: a performance budget test

```python
# tests/test_performance_budget.py
import pytest

BUDGETS = {
    "/": {"ttfb": 500, "dcl": 2000},
    "/dashboard": {"ttfb": 700, "dcl": 3000},
}

@pytest.mark.parametrize("path", BUDGETS.keys())
def test_page_meets_performance_budget(page, path):
    page.goto(path)
    timing = page.evaluate("""() => {
        const nav = performance.getEntriesByType('navigation')[0];
        return {
            ttfb: nav.responseStart - nav.startTime,
            dcl: nav.domContentLoadedEventEnd - nav.startTime,
        };
    }""")
    budget = BUDGETS[path]
    assert timing["ttfb"] < budget["ttfb"], f"{path} TTFB {timing['ttfb']}ms exceeds {budget['ttfb']}ms"
    assert timing["dcl"] < budget["dcl"], f"{path} DCL {timing['dcl']}ms exceeds {budget['dcl']}ms"
```

```text
# treating performance numbers as an explicit, parametrized
# budget per route turns "the site feels slower lately" into a
# CI failure with a specific route and specific metric attached,
# the same way a functional assertion turns a bug report into
# a specific failing line
```

## Exercise

1. Write a test that reads Navigation Timing data via `page.evaluate()` and
   asserts a TTFB budget on a real page.
2. Capture LCP with a `PerformanceObserver` wrapped in a `Promise` and assert
   it's under 2500ms.
3. Open a CDP session and throttle the network with
   `Network.emulateNetworkConditions`; confirm a page that loads instantly
   at full speed now visibly takes longer, and that your test's timeout
   still accounts for that.
4. Define a per-route performance budget dict like the one above for at
   least two routes in your app, and parametrize a single test function
   across both.
