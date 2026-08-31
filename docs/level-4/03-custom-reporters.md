# 03 · Custom Reporters

The built-in pytest console output and JUnit XML are enough to know pass/
fail, but a team running hundreds of E2E tests usually wants more: results
posted to Slack, flaky-test trends over time, or a dashboard tying failures
to the PR that introduced them. `pytest`'s hook system lets you build that
without forking the test runner.

## The pytest hooks that matter for reporting

```python
# conftest.py
def pytest_runtest_logreport(report):
    """Called once per test phase: setup, call, teardown."""
    if report.when == "call":
        print(f"[{report.outcome.upper()}] {report.nodeid} ({report.duration:.2f}s)")

def pytest_sessionfinish(session, exitstatus):
    """Called once, after the entire run finishes."""
    print(f"Session finished with exit status {exitstatus}")
```

```text
# pytest_runtest_logreport fires three times per test (setup/
# call/teardown) — checking report.when == "call" isolates the
# actual test result from fixture setup/teardown noise, which
# matters because a fixture failure also produces a report but
# with when == "setup", not "call"
```

## A minimal custom reporter class

```python
# reporters/slack_reporter.py
import json
import urllib.request

class SlackReporter:
    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url
        self.failures = []

    def pytest_runtest_logreport(self, report):
        if report.when == "call" and report.outcome == "failed":
            self.failures.append(report.nodeid)

    def pytest_sessionfinish(self, session, exitstatus):
        if not self.failures:
            return
        message = {
            "text": f":x: {len(self.failures)} test(s) failed:\n"
            + "\n".join(f"• {name}" for name in self.failures)
        }
        req = urllib.request.Request(
            self.webhook_url,
            data=json.dumps(message).encode(),
            headers={"Content-Type": "application/json"},
        )
        urllib.request.urlopen(req)
```

```python
# conftest.py
import os
from reporters.slack_reporter import SlackReporter

def pytest_configure(config):
    webhook = os.environ.get("SLACK_WEBHOOK_URL")
    if webhook:
        config.pluginmanager.register(SlackReporter(webhook), "slack_reporter")
```

```text
# pytest_configure runs once at startup — registering the
# reporter as a plugin object (rather than bare module-level
# functions) lets it hold state (self.failures) across the
# whole run and only fire the Slack post once, in
# pytest_sessionfinish, with the full failure list batched
```

## Writing a structured JSON report for a dashboard

```python
# reporters/json_reporter.py
import json
import time

class JSONReporter:
    def __init__(self, output_path: str):
        self.output_path = output_path
        self.results = []

    def pytest_runtest_logreport(self, report):
        if report.when != "call":
            return
        self.results.append({
            "nodeid": report.nodeid,
            "outcome": report.outcome,
            "duration": report.duration,
            "timestamp": time.time(),
        })

    def pytest_sessionfinish(self, session, exitstatus):
        with open(self.output_path, "w") as f:
            json.dump({
                "exit_status": exitstatus,
                "total": len(self.results),
                "passed": sum(1 for r in self.results if r["outcome"] == "passed"),
                "failed": sum(1 for r in self.results if r["outcome"] == "failed"),
                "results": self.results,
            }, f, indent=2)
```

```text
# a stable JSON schema per run is what makes a historical
# dashboard possible at all — feed successive runs' JSON files
# into any time-series store (even a flat file appended to
# in CI) to plot pass rate and duration trends over weeks
```

## Attaching trace/screenshot links to a failure report

```python
class TraceLinkingReporter:
    def pytest_runtest_logreport(self, report):
        if report.when == "call" and report.outcome == "failed":
            trace_path = f"test-results/{report.nodeid.replace('::', '-')}/trace.zip"
            print(f"::error title=Test Failed::{report.nodeid} — trace: {trace_path}")
```

```text
# the `::error title=...::` syntax is GitHub Actions' own
# log-annotation format — printing it from a reporter surfaces
# the failure (and where its trace lives) directly in the
# PR's "Files changed" / Checks UI, not just buried in a raw log
```

## Full worked example: combined reporter registered via `pyproject.toml`

```toml
# pyproject.toml
[tool.pytest.ini_options]
addopts = "-p reporters.json_reporter -p reporters.slack_reporter"
```

```python
# reporters/json_reporter.py — exposed as an installable pytest plugin
def pytest_addoption(parser):
    parser.addini("json_report_path", default="report.json", help="Path for JSON report output")

def pytest_configure(config):
    path = config.getini("json_report_path")
    config.pluginmanager.register(JSONReporter(path), "json_reporter_instance")
```

```text
# pytest_addoption + getini lets the reporter's output path be
# configured from pytest.ini/pyproject.toml rather than hard-
# coded, which is what turns a one-off script into something
# reusable across projects and CI jobs
```

## Exercise

1. Write a `pytest_runtest_logreport` hook in `conftest.py` that prints a
   one-line summary (`[PASSED] test_name (0.42s)`) for every test, filtering
   to `report.when == "call"`.
2. Turn it into a plugin class registered via `pytest_configure`, with
   accumulated state (a list of failures) reported once in
   `pytest_sessionfinish`.
3. Write a JSON reporter that writes a structured summary file after every
   run, and run the suite twice to confirm the file's shape stays
   consistent both times.
4. Add a `pytest_addoption`/`getini` option to make the JSON output path
   configurable, and confirm setting it in `pytest.ini` changes where the
   file is written without touching the reporter's code.
