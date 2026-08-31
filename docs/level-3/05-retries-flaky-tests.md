# 05 · Test Retries & Flaky Test Triage

A flaky test — one that passes and fails on the same code, non-deterministically
— is worse than no test at all: it trains the team to re-run CI instead of
reading failures, until eventually a *real* failure gets waved through too.
Retries are a mitigation, not a fix; this module covers both.

## Configuring retries

```ini
# pytest.ini
[pytest]
addopts = --reruns 2 --reruns-delay 1
```

```bash
pip install pytest-rerunfailures
```

```text
# a test that fails is immediately re-run up to 2 more times
# (with a 1s delay between attempts) before being reported as
# a real failure — this absorbs genuine one-off flakiness
# (a slow network blip) without hiding a consistently broken test
```

Scope retries to specific tests rather than the whole suite when only a few
are known to be environment-sensitive:

```python
import pytest

@pytest.mark.flaky(reruns=3, reruns_delay=2)
def test_third_party_widget_loads(page):
    page.goto("/embed")
    expect(page.frame_locator("#widget-frame").get_by_text("Loaded")).to_be_visible()
```

## Retries hide flakiness, they don't fix it — track the rate

```python
# conftest.py
import json, os
from datetime import datetime, timezone

def pytest_runtest_logreport(report):
    if report.when == "call" and report.outcome == "rerun":
        log_path = "flaky_log.jsonl"
        entry = {
            "test": report.nodeid,
            "timestamp": datetime.now(timezone.utc).isoformat(),
        }
        with open(log_path, "a") as f:
            f.write(json.dumps(entry) + "\n")
```

```text
# every time pytest-rerunfailures triggers a rerun, this hook
# appends a record — over weeks this file becomes a ranked list
# of the flakiest tests in the suite, which is the actual input
# needed to prioritize fixing them instead of just tolerating
# a growing rerun count
```

## Common root causes and their real fixes

**1. Race condition: acting before the app is ready**

```python
# flaky — click can land before the SPA finishes rendering the list
def test_delete_item_flaky(page):
    page.goto("/items")
    page.get_by_role("button", name="Delete").first.click()

# fixed — wait for a concrete signal the list has data
def test_delete_item_fixed(page):
    page.goto("/items")
    expect(page.get_by_role("listitem")).to_have_count(3)
    page.get_by_role("button", name="Delete").first.click()
```

**2. Test order dependence: shared state between tests**

```python
# flaky — depends on test_create_user having run first in this worker
def test_login_flaky(page):
    page.goto("/login")
    page.get_by_label("Email").fill("shared@example.com")

# fixed — each test creates its own fixture data
def test_login_fixed(page, api_context):
    user = api_context.post("/users", data={"email": f"user-{uuid4()}@example.com"}).json()
    page.goto("/login")
    page.get_by_label("Email").fill(user["email"])
```

**3. Animation/transition timing**

```python
# flaky — modal is technically present but still fading in
def test_modal_flaky(page):
    page.get_by_role("button", name="Open").click()
    page.get_by_role("dialog").get_by_role("button", name="Confirm").click()

# fixed — assert visibility (which waits for the animation to
# settle) before acting
def test_modal_fixed(page):
    page.get_by_role("button", name="Open").click()
    dialog = page.get_by_role("dialog")
    expect(dialog).to_be_visible()
    dialog.get_by_role("button", name="Confirm").click()
```

**4. Network-dependent third-party content**

```python
# fixed — stub the flaky third party instead of retrying against it
def test_page_with_ad_slot(page):
    page.route("**/ads.example.com/**", lambda route: route.fulfill(status=204, body=""))
    page.goto("/article")
    expect(page.get_by_role("heading")).to_be_visible()
```

## Quarantining a known-flaky test instead of deleting it

Deleting a flaky test loses coverage silently; quarantining keeps it running
and visible without blocking merges while it's being fixed:

```python
import pytest

@pytest.mark.flaky(reruns=5)
@pytest.mark.xfail(reason="JIRA-4821: intermittent 3rd-party auth widget timeout", strict=False)
def test_sso_widget_login(page):
    ...
```

```text
# strict=False means an unexpected pass doesn't fail the suite
# either — the test keeps running and reporting, but a known,
# ticketed flake can't block a PR while it's being investigated
```

## A retry policy that fits the team, not just the tool

```ini
[pytest]
# global light retry for genuine environment blips
addopts = --reruns 1

# tests known to need more get an explicit @pytest.mark.flaky(reruns=N)
# — never raise the global number past 1-2, or the suite starts
# tolerating real bugs instead of surfacing them
```

## Exercise

1. Write a test with a deliberate race condition (act immediately after
   `page.goto()` on a slow-loading element) and reproduce a flaky failure by
   running it several times.
2. Fix it using an `expect(...)` assertion as an explicit wait condition
   instead of a fixed sleep, and confirm it now passes consistently across
   10 runs (`pytest --count=10` with `pytest-repeat`, or a shell loop).
3. Add `pytest-rerunfailures` with `--reruns 2`, deliberately leave one test
   flaky, and confirm the JUnit/console output distinguishes "passed after
   rerun" from "passed on first try."
4. Add the `pytest_runtest_logreport` hook above (or equivalent) and confirm
   it writes an entry to a log file the one time your reintroduced flaky
   test needs a rerun.
