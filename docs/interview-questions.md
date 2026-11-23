---
title: "Playwright Interview Questions and Answers (179 Questions)"
description: "179 Playwright interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Playwright Mastery Path."
---

# Playwright Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Playwright Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is Playwright &amp; Why E2E Testing](level-1/01-what-is-playwright.md)

**Q1.** What is Playwright?

**Answer:** A library for automating and testing web browsers (Chromium, Firefox, WebKit) from code

One API, three engines.

**Q2.** Where does an end-to-end test sit in the test pyramid?

**Answer:** At the top: few, slow tests that exercise the whole system like a user

Balance with unit tests.

**Q3.** Which languages does Playwright officially support?

**Answer:** JavaScript/TypeScript, Python, Java and .NET

Same concepts across them.

**Q4.** Why do Playwright tests get less flaky than older tools?

**Answer:** Built-in auto-waiting and retrying web-first assertions

Core design.

### [02 · Installation &amp; Project Setup](level-1/02-installation-setup.md)

**Q5.** What does `playwright install` do?

**Answer:** Downloads the browser binaries Playwright drives

Run it after pip install.

**Q6.** Which package gives you the `page` fixture in pytest?

**Answer:** pytest-playwright

Plugin.

**Q7.** How run your tests headed for debugging?

**Answer:** pytest --headed

Also --browser firefox.

**Q8.** Why use a virtual environment?

**Answer:** Isolate project dependencies and versions

Reproducible.

### [03 · Your First Script](level-1/03-first-script.md)

**Q9.** What does this print?

```python
# page content:
# <h1>Shop</h1><ul><li class="p">Pen</li><li class="p">Book</li><li class="p">Bag</li></ul><button>Buy</button>
page.goto("about:blank")
page.set_content("<p id='x'>hi</p>")
print(page.title() == "", page.locator("#x").text_content())
```

**Answer:** True hi

about:blank has an empty title. Verified by running it with Playwright for Python.

**Q10.** What does `browser.new_page()` give you?

**Answer:** A new page in a fresh default context

Isolated state.

**Q11.** What must a script always do at the end?

**Answer:** Close the browser to free resources

Use with-blocks.

**Q12.** What does sync_playwright() manage?

**Answer:** Starting and stopping the Playwright driver in the sync API

Context manager.

### [04 · Locators](level-1/04-locators.md)

**Q13.** What does this print?

```python
# page content:
# <h1>Shop</h1><ul><li class="p">Pen</li><li class="p">Book</li><li class="p">Bag</li></ul><button>Buy</button>
print(page.locator("li.p").count(), page.get_by_role("button", name="Buy").text_content(), page.locator("li.p").nth(1).text_content(), page.locator("li.p").last.text_content())
```

**Answer:** 3 Buy Book Bag

Locators are lazy descriptions; nth is zero-based. Verified by running it with Playwright for Python.

**Q14.** What does this print?

```python
# page content:
# <label for="e">Email</label><input id="e"><label for="c">Remember me</label><input id="c" type="checkbox"><select id="s"><option value="a">Alpha</option><option value="b">Beta</option></select><button disabled>Send</button>
page.get_by_label("Email").fill("a@b.c")
print(page.locator("#e").input_value(), page.get_by_text("Remember me").count())
```

**Answer:** a@b.c 1

get_by_label finds the input by its label. Verified by running it with Playwright for Python.

**Q15.** Which locator style is most recommended?

**Answer:** User-facing: get_by_role, get_by_label, get_by_text

Resilient.

**Q16.** What happens when a locator matches several elements and you call click()?

**Answer:** Strict mode raises an error

Narrow it or use .first.

### [05 · Actions](level-1/05-actions.md)

**Q17.** What does this print?

```python
# page content:
# <label for="e">Email</label><input id="e"><label for="c">Remember me</label><input id="c" type="checkbox"><select id="s"><option value="a">Alpha</option><option value="b">Beta</option></select><button disabled>Send</button>
page.get_by_label("Email").fill("x")
page.get_by_label("Remember me").check()
page.select_option("#s", "b")
print(page.get_by_label("Remember me").is_checked(), page.locator("#s").input_value(), page.locator("#e").input_value())
```

**Answer:** True b x

fill, check and select_option. Verified by running it with Playwright for Python.

**Q18.** What does this print?

```python
# page content:
# <button id="b" onclick="this.textContent='Clicked'">Go</button>
page.click("#b")
print(page.text_content("#b"))
```

**Answer:** Clicked

click runs the handler. Verified by running it with Playwright for Python.

**Q19.** What does fill() do differently from type()/press_sequentially()?

**Answer:** fill sets the value at once; typing sends key events one by one

Use typing only to test key handlers.

**Q20.** How press Enter?

**Answer:** page.keyboard.press('Enter') or locator.press('Enter')

Key names.

### [06 · Assertions with expect](level-1/06-assertions.md)

**Q21.** What does this print?

```python
# page content:
# <label for="e">Email</label><input id="e"><label for="c">Remember me</label><input id="c" type="checkbox"><select id="s"><option value="a">Alpha</option><option value="b">Beta</option></select><button disabled>Send</button>
try:
    expect(page.get_by_role("button", name="Send")).to_be_enabled()
except AssertionError:
    print("not enabled")
expect(page.get_by_role("button", name="Send")).to_be_disabled()
expect(page.locator("h1")).to_have_count(0)
print("ok")
```

**Answer:** not enabled
ok

Web-first assertions retry then raise AssertionError. Verified by running it with Playwright for Python.

**Q22.** What does this print?

```python
# page content:
# <h1>Shop</h1><ul><li class="p">Pen</li><li class="p">Book</li><li class="p">Bag</li></ul><button>Buy</button>
expect(page.locator("li.p")).to_have_count(3)
expect(page.locator("h1")).to_have_text("Shop")
expect(page.locator("li.p").first).to_contain_text("Pe")
print("passed")
```

**Answer:** passed

Auto-retrying assertions. Verified by running it with Playwright for Python.

**Q23.** Why prefer expect(locator).to_be_visible() to assert locator.is_visible()?

**Answer:** expect retries until the condition holds or times out; is_visible is a one-shot check

Avoid flakiness.

**Q24.** What does expect(page).to_have_url() check?

**Answer:** The current URL (string or regex), with retries

Navigation result.

### [07 · Waiting &amp; Auto-Wait Philosophy](level-1/07-waiting-auto-wait.md)

**Q25.** What does this print?

```python
# page content:
# <div id="r"></div><script>setTimeout(()=>{document.getElementById("r").textContent="loaded"},300)</script>
page.set_default_timeout(3000)
expect(page.locator("#r")).to_have_text("loaded")
print(page.locator("#r").text_content())
```

**Answer:** loaded

expect waits for the asynchronous update. Verified by running it with Playwright for Python.

**Q26.** What is auto-wait?

**Answer:** Playwright waits for elements to be visible, stable, enabled and receiving events before acting

Actionability checks.

**Q27.** Why avoid page.wait_for_timeout(5000)?

**Answer:** Fixed sleeps are slow when fast and flaky when slow; wait for a condition instead

Use expect or wait_for.

**Q28.** What does the default action timeout typically do?

**Answer:** Fails the action with TimeoutError when the element is not actionable in time

Configurable.

### [08 · Handling Forms &amp; Dropdowns](level-1/08-forms-dropdowns.md)

**Q29.** What does this print?

```python
# page content:
# <label for="e">Email</label><input id="e"><label for="c">Remember me</label><input id="c" type="checkbox"><select id="s"><option value="a">Alpha</option><option value="b">Beta</option></select><button disabled>Send</button>
page.select_option("#s", label="Beta")
print(page.locator("#s").input_value())
page.get_by_label("Remember me").set_checked(True)
page.get_by_label("Remember me").set_checked(False)
print(page.get_by_label("Remember me").is_checked())
```

**Answer:** b
False

select by label; set_checked is idempotent. Verified by running it with Playwright for Python.

**Q30.** How do you pick an option by its value?

**Answer:** select_option('b') or select_option(value='b')

Also by label/index.

**Q31.** Which locator targets an input via its visible label?

**Answer:** get_by_label('Email')

Accessible.

**Q32.** Why is disabled-button clicking a problem?

**Answer:** Actionability waits until it is enabled, then times out if it never is

Assert state first.

### [09 · Running Headed vs. Headless](level-1/09-headed-vs-headless.md)

**Q33.** What is headless mode?

**Answer:** Running the browser without a visible window

Default in CI.

**Q34.** Why run headed locally?

**Answer:** To watch and debug what the test does

--headed, slow_mo.

**Q35.** What does slow_mo do?

**Answer:** Slows each operation by N ms so you can follow along

Debugging aid.

**Q36.** What does headless differ in for tests?

**Answer:** Mostly nothing; same engine, but rendering/viewport defaults can differ

Set viewport explicitly.

### [10 · Project — Automate a Login Flow](level-1/10-project-login-flow.md)

**Q37.** What does the login flow project combine?

**Answer:** Locators, fills, clicks, assertions and a pytest test

Level 1 project.

**Q38.** What proves a login worked?

**Answer:** Assert URL or a logged-in element (e.g. dashboard heading) is visible

User-visible outcome.

**Q39.** How test a failed login?

**Answer:** Submit wrong credentials and assert the error message

Negative paths.

**Q40.** Why use a public demo site for practice?

**Answer:** It is safe, stable and built for automation practice

Don't hit production.

## Level 2 · Intermediate

### [01 · Page Object Model](level-2/01-page-object-model.md)

**Q41.** What does this print?

```python
# page content:
# <label for="e">Email</label><input id="e"><label for="c">Remember me</label><input id="c" type="checkbox"><select id="s"><option value="a">Alpha</option><option value="b">Beta</option></select><button disabled>Send</button>
class FormPage:
    def __init__(self, page):
        self.page = page
        self.email = page.get_by_label("Email")
        self.remember = page.get_by_label("Remember me")
    def fill(self, e):
        self.email.fill(e)
        self.remember.check()
        return self
fp = FormPage(page).fill("a@b.c")
print(fp.email.input_value(), fp.remember.is_checked())
```

**Answer:** a@b.c True

Page objects own locators and actions; locators are lazy. Verified by running it with Playwright for Python.

**Q42.** What belongs in a page object?

**Answer:** Locators and user actions for a page, not assertions about test goals

Keep tests readable.

**Q43.** Why build locators in __init__?

**Answer:** Locators are lazy descriptions so it's safe before navigation

Resolve on use.

**Q44.** What does a base page share?

**Answer:** Common chrome like navigation, toasts and helpers

Inheritance.

### [02 · Fixtures &amp; Test Hooks](level-2/02-fixtures-hooks.md)

**Q45.** What does a pytest fixture do?

**Answer:** Provides setup (and teardown after yield) to tests via parameters

Dependency injection.

**Q46.** What is fixture scope function vs session?

**Answer:** Function runs per test; session runs once per run

Expensive setup.

**Q47.** What does `yield` do in a fixture?

**Answer:** Splits setup from teardown, which runs even if the test fails

try/finally.

**Q48.** Where do shared fixtures live?

**Answer:** conftest.py

Auto-discovered.

### [03 · Multiple Tabs, Frames &amp; Popups](level-2/03-tabs-frames-popups.md)

**Q49.** What does this print?

```python
# page content:
# <a id="l" href="#x" target="_blank">go</a>
with page.context.expect_page() as info:
    page.click("#l")
new = info.value
print(len(page.context.pages), new.url)
```

**Answer:** 2 about:blank#x

expect_page captures the new tab (target=_blank opens a popup). Verified by running it with Playwright for Python.

**Q50.** What does this print?

```python
# page content:
# <iframe srcdoc="<button>Inner</button>"></iframe><button>Outer</button>
print(page.get_by_role("button").count(), page.frame_locator("iframe").get_by_role("button").text_content())
```

**Answer:** 1 Inner

Locators don't pierce iframes; use frame_locator. Verified by running it with Playwright for Python.

**Q51.** How handle a JavaScript alert?

**Answer:** Register page.on('dialog', handler) to accept or dismiss it

Unhandled dialogs auto-dismiss.

**Q52.** What does context isolation give?

**Answer:** Separate cookies/storage per context, like incognito profiles

Multi-user tests.

### [04 · Network Interception &amp; Mocking](level-2/04-network-interception.md)

**Q53.** What does this print?

```python
# page content:
# <div id="o"></div>
page.route("http://test.local/api", lambda r: r.fulfill(json={"name": "mocked"}))
page.evaluate("fetch('http://test.local/api').then(r => r.json()).then(j => { document.getElementById('o').textContent = j.name })")
expect(page.locator("#o")).to_have_text("mocked")
print(page.text_content("#o"))
```

**Answer:** mocked

route + fulfill mocks the network response. Verified by running it with Playwright for Python.

**Q54.** What does page.route do?

**Answer:** Intercepts matching requests to fulfill, modify, abort or continue them

Network control.

**Q55.** Why mock a flaky third-party API in tests?

**Answer:** Deterministic, fast tests independent of external services

But keep some real contract tests.

**Q56.** What does route.abort() simulate?

**Answer:** A failed request (e.g. offline) to test error handling

Resilience.

### [05 · File Upload &amp; Download](level-2/05-file-upload-download.md)

**Q57.** How upload a file?

**Answer:** locator.set_input_files('path') on the file input

Also lists and buffers.

**Q58.** How capture a download?

**Answer:** with page.expect_download() as d: click; then d.value.save_as(path)

Event-based.

**Q59.** Why use tmp_path for test files?

**Answer:** Isolated temporary directory cleaned by pytest

No leftovers.

**Q60.** What should you assert after a download?

**Answer:** Suggested filename and file contents

Verify behavior.

### [06 · Parallel Execution &amp; Sharding](level-2/06-parallel-sharding.md)

**Q61.** How run tests in parallel with pytest?

**Answer:** pytest -n auto with pytest-xdist

Workers.

**Q62.** Why must tests be independent for parallelism?

**Answer:** Order and shared state would otherwise cause flaky failures

Isolation.

**Q63.** What is sharding?

**Answer:** Splitting the suite across multiple machines/jobs

CI matrix.

**Q64.** What gives each test isolated state?

**Answer:** A fresh browser context per test

pytest-playwright default.

### [07 · Test Configuration &amp; Projects](level-2/07-test-configuration.md)

**Q65.** Where configure browser, viewport and base URL for pytest-playwright?

**Answer:** pytest.ini options or conftest fixtures (browser_context_args)

Central config.

**Q66.** What does --base-url do?

**Answer:** Lets page.goto('/path') resolve relative paths

Environment switch.

**Q67.** What does browser_context_args customize?

**Answer:** Viewport, locale, timezone, storage state and more for each context

Fixture override.

**Q68.** Why keep config out of tests?

**Answer:** Same tests run across environments without edits

Separation.

### [08 · Environment &amp; Test Data Management](level-2/08-env-test-data.md)

**Q69.** Where keep secrets for test environments?

**Answer:** Environment variables/CI secrets, not committed files

Security.

**Q70.** What does python-dotenv do?

**Answer:** Loads variables from a .env file for local runs

Gitignore it.

**Q71.** What's a good test data approach?

**Answer:** Create the data a test needs via API/fixtures and clean it up

Independence.

**Q72.** Why not hard-code URLs?

**Answer:** They differ between dev, staging and production

Config.

### [09 · Debugging Failing Tests](level-2/09-debugging-failing-tests.md)

**Q73.** What do you open to debug interactively?

**Answer:** PWDEBUG=1 (Playwright Inspector) or page.pause()

Step through.

**Q74.** What does the trace capture?

**Answer:** Actions, DOM snapshots, network and console for each step

Post-mortem.

**Q75.** What do screenshots on failure help with?

**Answer:** Seeing the page state when the test failed

--screenshot only-on-failure.

**Q76.** What's the first step with a failing test?

**Answer:** Reproduce it and read the actual error and trace

Evidence.

### [10 · Project — POM Test Suite](level-2/10-project-pom-suite.md)

**Q77.** What does the POM suite project combine?

**Answer:** Page objects, fixtures, config, test data and parallel runs

Level 2 project.

**Q78.** How keep the suite maintainable?

**Answer:** Small page objects, clear names and no logic in tests

Readability.

**Q79.** Where do assertions live?

**Answer:** In the tests, using page-object state

Intent visible.

**Q80.** How verify the suite is stable?

**Answer:** Run repeatedly and in parallel

Flakiness check.

## Level 3 · Advanced

### [01 · Visual Regression Testing](level-3/01-visual-regression.md)

**Q81.** What does visual regression compare?

**Answer:** Screenshots against approved baselines

expect(page).to_have_screenshot().

**Q82.** Why are screenshots flaky across machines?

**Answer:** Fonts, rendering and OS differences; use the same container/CI environment

Consistency.

**Q83.** When update baselines?

**Answer:** After a deliberate, reviewed UI change

Review diffs.

**Q84.** What reduces noise?

**Answer:** Masking dynamic regions and disabling animations

mask=[...].

### [02 · Authentication State Reuse](level-3/02-auth-state-reuse.md)

**Q85.** What does storage_state do?

**Answer:** Saves cookies/localStorage after login to reuse in other contexts

Skip repeated UI logins.

**Q86.** Why reuse auth state?

**Answer:** Faster, more stable tests that don't repeat the login UI

Test login once.

**Q87.** Where store the state file?

**Answer:** Gitignored location, treated as a secret

Contains session tokens.

**Q88.** When state expires?

**Answer:** Regenerate it in a setup step before the run

Fresh.

### [03 · API Testing with Request Context](level-3/03-api-testing.md)

**Q89.** Does page.route() intercept calls made with page.request / APIRequestContext?

**Answer:** No: routing applies to browser traffic; API requests go straight to the server

Use a real test API or mock the server instead.

**Q90.** What is request.new_context for?

**Answer:** Calling APIs directly (no browser UI) with shared headers/auth

API tests.

**Q91.** Why combine API setup with UI checks?

**Answer:** Fast arrange via API, then verify the user-visible result

Efficient.

**Q92.** What do you assert for API responses?

**Answer:** Status code, headers and body shape

Contract.

### [04 · CI Integration (GitHub Actions)](level-3/04-ci-integration.md)

**Q93.** What should CI do on each push?

**Answer:** Install dependencies and browsers, run tests and upload artifacts

Quality gate.

**Q94.** Which artifacts help debugging?

**Answer:** Traces, screenshots and HTML reports

Post-mortem.

**Q95.** Why use the official Playwright Docker image?

**Answer:** Browsers and OS dependencies are preinstalled and consistent

Reproducibility.

**Q96.** How speed up CI?

**Answer:** Cache dependencies and shard tests across jobs

Parallel.

### [05 · Test Retries &amp; Flaky Test Triage](level-3/05-retries-flaky-tests.md)

**Q97.** What are flaky tests?

**Answer:** Tests that pass and fail without code changes

Erode trust.

**Q98.** What do retries do?

**Answer:** Rerun failures to detect flakiness, which should still be investigated

Mitigation, not a cure.

**Q99.** Most common flake cause?

**Answer:** Timing assumptions and shared state

Use auto-wait.

**Q100.** What should you do with a flaky test?

**Answer:** Fix the root cause or quarantine it with a ticket

Track.

### [06 · Advanced Locator Strategies](level-3/06-advanced-locators.md)

**Q101.** What does this print?

```python
# page content:
# <ul><li><button>Edit</button> Pen</li><li><button>Edit</button> Book</li></ul>
row = page.get_by_role("listitem").filter(has_text="Book")
print(row.get_by_role("button", name="Edit").count(), page.get_by_role("listitem").count(), page.get_by_role("listitem").nth(0).text_content().strip())
```

**Answer:** 1 2 Edit Pen

filter() scopes locators. Verified by running it with Playwright for Python.

**Q102.** What does this print?

```python
# page content:
# <div data-testid="card"><h2>A</h2></div><div data-testid="card"><h2>B</h2></div>
cards = page.get_by_test_id("card")
print(cards.count(), cards.filter(has=page.get_by_role("heading", name="B")).count(), cards.nth(1).locator("h2").text_content())
```

**Answer:** 2 1 B

has= filters by inner locator. Verified by running it with Playwright for Python.

**Q103.** When use data-testid?

**Answer:** When no accessible role or text is stable

Fallback.

**Q104.** What does locator.and_/or_ do?

**Answer:** Combine locators to match either/both conditions

Advanced.

### [07 · Accessibility Testing](level-3/07-accessibility-testing.md)

**Q105.** What does axe-core check?

**Answer:** Common accessibility violations (contrast, labels, roles)

Automated subset.

**Q106.** What can automation not prove?

**Answer:** Full accessibility; manual testing with assistive tech is still needed

Limits.

**Q107.** Why prefer get_by_role?

**Answer:** It mirrors how assistive technology sees the page and encourages accessible markup

Dual benefit.

**Q108.** When fail the build on a11y?

**Answer:** On new violations, gradually ratcheting standards

Baseline.

### [08 · Performance &amp; Tracing](level-3/08-performance-tracing.md)

**Q109.** What can a trace record?

**Answer:** Screenshots, snapshots, network and console per action

context.tracing.start().

**Q110.** What does page.evaluate with performance.timing/getEntries give?

**Answer:** Browser-reported timing metrics

Basic perf.

**Q111.** What's a limit of E2E perf checks?

**Answer:** Noisy; use dedicated tools for load and budgets

Lighthouse/load tests.

**Q112.** How use tracing in CI?

**Answer:** Record on first retry and upload as an artifact

trace on-first-retry.

### [09 · Test Data Seeding via API](level-3/09-test-data-seeding.md)

**Q113.** Why seed test data via API/DB?

**Answer:** Fast, deterministic setup instead of clicking through UIs

Arrange.

**Q114.** How avoid collisions in parallel tests?

**Answer:** Unique data per test (uuid suffixes) and cleanup

Isolation.

**Q115.** What should cleanup do?

**Answer:** Remove created data even if the test fails (fixture teardown)

yield.

**Q116.** Why not rely on pre-existing data?

**Answer:** It drifts and breaks tests unpredictably

Own your data.

### [10 · Project — Full CI E2E Pipeline](level-3/10-project-ci-pipeline.md)

**Q117.** What does the CI pipeline project combine?

**Answer:** Config, auth reuse, API setup, parallelism, artifacts and reports

Level 3 project.

**Q118.** What gates a merge?

**Answer:** Green tests plus reviewed artifacts for failures

Quality.

**Q119.** What do you publish after a run?

**Answer:** HTML report and traces for failures

Debuggability.

**Q120.** How keep pipelines fast?

**Answer:** Shard, cache and run only affected tests where possible

Feedback time.

## Level 4 · Master

### [01 · Cross-Browser &amp; Device Strategy](level-4/01-cross-browser-strategy.md)

**Q121.** What does cross-browser strategy decide?

**Answer:** Which browsers/devices to cover and how often, based on usage and risk

Balance cost.

**Q122.** How run one test on several engines?

**Answer:** pytest --browser chromium --browser firefox --browser webkit

Matrix.

**Q123.** What does WebKit give you?

**Answer:** A proxy for Safari's engine on any OS

Not exact.

**Q124.** How emulate a phone?

**Answer:** playwright.devices['iPhone 13'] context options

Viewport, UA, touch.

### [02 · Component Testing](level-4/02-component-testing.md)

**Q125.** What does component testing mount?

**Answer:** A single UI component in a real browser instead of the full app

Fast feedback.

**Q126.** When choose component over E2E?

**Answer:** For rendering and interaction logic of isolated UI parts

Pyramid.

**Q127.** What stays E2E?

**Answer:** Critical user journeys across the full stack

Few and valuable.

**Q128.** What's a limitation?

**Answer:** Doesn't cover integration with real backends

Complement.

### [03 · Custom Reporters](level-4/03-custom-reporters.md)

**Q129.** What is a custom reporter for?

**Answer:** Sending results to your own format or system (Slack, dashboards)

Hooks into results.

**Q130.** Which pytest hook adds report data?

**Answer:** pytest_runtest_makereport

Per test outcome.

**Q131.** What belongs in a failure report?

**Answer:** Test name, error, trace link and environment

Actionable.

**Q132.** Why track trends?

**Answer:** Spot flaky tests and slow tests over time

Metrics.

### [04 · Scaling a Test Suite](level-4/04-scaling-test-suite.md)

**Q133.** What helps scale a big suite?

**Answer:** Parallel workers, sharding, stable tests and fast setup via API

Combination.

**Q134.** What's a test impact strategy?

**Answer:** Run only tests relevant to changed code, with full runs nightly

Speed.

**Q135.** What kills scale?

**Answer:** Flaky tests and shared state

Fix first.

**Q136.** How organize tests?

**Answer:** By feature with tags/markers for suites (smoke, regression)

pytest markers.

### [05 · Debugging with Trace Viewer](level-4/05-trace-viewer.md)

**Q137.** How open a trace?

**Answer:** playwright show-trace trace.zip or trace.playwright.dev

Viewer.

**Q138.** What does the timeline show?

**Answer:** Each action, snapshot before/after, network and console

Time travel.

**Q139.** When use trace over screenshot?

**Answer:** To see what happened between steps, especially in CI failures

Rich.

**Q140.** What can you inspect per step?

**Answer:** DOM snapshot, locators used, errors and network calls

Deep dive.

### [06 · Playwright Architecture Internals](level-4/06-architecture-internals.md)

**Q141.** How does Playwright talk to browsers?

**Answer:** A driver process over a protocol (CDP or equivalents), one API for all engines

Direct control.

**Q142.** Why is it faster than WebDriver-based tools?

**Answer:** Persistent connection and event-driven control instead of request/response commands

Architecture.

**Q143.** What is a browser context?

**Answer:** An isolated profile (cookies, storage) inside one browser

Cheap to create.

**Q144.** What does auto-wait rely on?

**Answer:** Actionability checks the driver performs before each action

Built in.

### [07 · Test Ownership &amp; Team Practices](level-4/07-test-ownership-practices.md)

**Q145.** Who should own tests?

**Answer:** The team that owns the feature, with shared standards

Shift left.

**Q146.** What makes a good test?

**Answer:** Independent, deterministic, readable and focused on user behavior

Quality.

**Q147.** What do you do with obsolete tests?

**Answer:** Delete them; tests are code with maintenance cost

Prune.

**Q148.** How review tests?

**Answer:** Like production code, checking clarity and stability

Standards.

### [08 · Migrating from Selenium](level-4/08-migrating-from-selenium.md)

**Q149.** What maps Selenium to Playwright concepts?

**Answer:** WebDriver ↔ browser/context/page; explicit waits → auto-wait

Migration.

**Q150.** What do you usually remove when migrating?

**Answer:** Explicit waits and sleeps

Auto-wait.

**Q151.** How migrate a big suite?

**Answer:** Incrementally: new tests in Playwright, port high-value ones, retire old gradually

Risk.

**Q152.** What improves after migration?

**Answer:** Speed and reliability from parallel contexts and web-first assertions

Benefits.

### [09 · Extending Playwright (Custom Fixtures/Plugins)](level-4/09-extending-playwright.md)

**Q153.** How extend Playwright?

**Answer:** Custom fixtures, helper functions and plugins for your domain

Composition.

**Q154.** What do custom expect helpers do?

**Answer:** Encapsulate repeated assertions with clear failure messages

Readable.

**Q155.** Why wrap Playwright in your own layer?

**Answer:** Hide repetitive details, but avoid over-abstracting

Balance.

**Q156.** What is a pytest plugin?

**Answer:** A package that adds fixtures, hooks and options via entry points

Distribution.

### [10 · Capstone — Production-Grade E2E Framework](level-4/10-capstone-project.md)

**Q157.** What does the capstone combine?

**Answer:** Architecture, POM, fixtures, API setup, CI, reports and cross-browser runs

Production suite.

**Q158.** How prove it works?

**Answer:** Stable repeated CI runs with traces on failure

Evidence.

**Q159.** What documentation helps?

**Answer:** README, conventions and how to debug failures

Onboarding.

**Q160.** What sustains it?

**Answer:** Ownership, flake triage and regular dependency updates

Maintenance.

## Mixed questions across all levels

**Q161.** What does this print?

```python
# page content:
# <ul><li>Pen</li><li>Book</li></ul>
loc = page.locator("li")
page.evaluate("document.querySelector('ul').insertAdjacentHTML('beforeend', '<li>Bag</li>')")
print(loc.count(), loc.last.text_content())
```

**Answer:** 3 Bag

A Locator is lazy: it queries the DOM each time you use it. Verified by running it with Playwright for Python.

**Q162.** Why are locators lazy descriptions and not element handles?

**Answer:** They re-resolve on every use so they keep working when the DOM re-renders

Resilience to change.

**Q163.** What does this print?

```python
# page content:
# <button>Save</button><button>Save as</button>
try:
    page.get_by_role("button", name="Save").click()
except Exception as e:
    print(type(e).__name__)
print(page.get_by_role("button", name="Save", exact=True).count())
```

**Answer:** Error
1

Strict mode violation: name matches substrings unless exact=True. Verified by running it with Playwright for Python.

**Q164.** Which assertion style waits for async updates?

**Answer:** expect(locator).to_have_text(...) (web-first, auto-retrying)

Retries until timeout.

**Q165.** What does this print?

```python
# page content:
# <div id="r">0</div><script>let n=0;const t=setInterval(()=>{document.getElementById("r").textContent=++n;if(n==3)clearInterval(t)},50)</script>
expect(page.locator("#r")).to_have_text("3")
print(page.text_content("#r"))
```

**Answer:** 3

Assertions retry until the text matches. Verified by running it with Playwright for Python.

**Q166.** Why is time.sleep(5) in a test a smell?

**Answer:** It's slow when the app is fast and still flaky when it is slow; wait for a condition

Auto-wait and expect.

**Q167.** A test passes alone but fails in parallel. Likely cause?

**Answer:** Shared state (accounts, data, files) between tests

Make tests independent.

**Q168.** What is the best use of storage_state?

**Answer:** Log in once in setup and reuse cookies/localStorage for other tests

Faster, fewer login flakes.

**Q169.** How do you capture a popup opened by a click?

**Answer:** with page.context.expect_page() (or page.expect_popup()) around the click

Event-based.

**Q170.** How should network mocks be used?

**Answer:** To make third-party/error scenarios deterministic, keeping some real end-to-end coverage

Balance.

**Q171.** What does a trace help with in CI?

**Answer:** Replaying a failed run step by step with DOM, network and console

Upload as an artifact.

**Q172.** Screenshot comparisons differ between laptop and CI. What helps?

**Answer:** Generate and compare baselines in the same containerized environment

Fonts and rendering.

**Q173.** A flaky test is retried and passes. What should you do?

**Answer:** Investigate the root cause; retries only reveal and tolerate flakiness

Track flaky tests.

**Q174.** Which locator is usually most resilient?

**Answer:** get_by_role('button', name='Pay')

User-facing.

**Q175.** What does the Page Object Model buy you?

**Answer:** One place to change when the UI changes and tests that read like user stories

Maintainability.

**Q176.** What does a fresh browser context per test provide?

**Answer:** Isolated cookies and storage at low cost

Test independence.

**Q177.** How can one suite run on Chromium, Firefox and WebKit?

**Answer:** pytest --browser chromium --browser firefox --browser webkit

One API.

**Q178.** What should fail a pull request pipeline?

**Answer:** Failing tests, with traces/screenshots uploaded for diagnosis

Quality gate.

**Q179.** Automated accessibility checks (axe) are best described as?

**Answer:** A useful baseline that catches common issues but doesn't replace manual testing

Limits.

## Practical coding questions

### A page object and isolated login tests

Implement LoginPage (locators for the Email and Password fields, the Sign in button and the alert; a login() method that returns self), then run the three login cases in fresh browser contexts. A deliberately wrong expectation shows how a failure is reported.

Starter code:

```
from playwright.sync_api import sync_playwright, expect

LOGIN_HTML = '<form id="f"><label>Email <input id="email"></label><label>Password <input id="pw" type="password"></label>\n<button type="submit">Sign in</button><p id="msg" role="alert"></p></form>\n<script>document.getElementById("f").addEventListener("submit",e=>{e.preventDefault();\nconst ok=document.getElementById("email").value==="ada@example.com"&&document.getElementById("pw").value==="s3cret";\ndocument.getElementById("msg").textContent=ok?"Welcome, Ada":"Invalid credentials";});</script>'

class LoginPage:
    def __init__(self, page):
        self.page = page
        # create locators for: Email field, Password field, the "Sign in" button and the alert message
        # (use get_by_label, get_by_role) -- locators are lazy, so it is fine to build them here

    def login(self, email, password):
        # fill both fields, click the button and return self so calls can be chained
        return self

def check(name, fn):
    try:
        fn()
        print("PASS", name)
    except AssertionError:
        print("FAIL", name)

with sync_playwright() as p:
    browser = p.chromium.launch()
    for email, pw, expected in [("ada@example.com", "s3cret", "Welcome, Ada"),
                                ("ada@example.com", "wrong", "Invalid credentials"),
                                ("", "", "Invalid credentials")]:
        context = browser.new_context()            # fresh, isolated state per test
        page = context.new_page()
        page.set_content(LOGIN_HTML)
        page.set_default_timeout(1000)
        expect.set_options(timeout=1000)
        lp = LoginPage(page).login(email, pw)
        check(f"login({email or 'empty'!r}, {pw or 'empty'!r})", lambda: expect(lp.message).to_have_text(expected))
        context.close()
    # a deliberately wrong expectation shows how a failing check is reported
    context = browser.new_context(); page = context.new_page(); page.set_content(LOGIN_HTML)
    expect.set_options(timeout=500)
    check("expects wrong text", lambda: expect(LoginPage(page).login("ada@example.com", "s3cret").message).to_have_text("Goodbye"))
    browser.close()
```

Expected output:

```text
PASS login('ada@example.com', 's3cret')
PASS login('ada@example.com', 'wrong')
PASS login('empty', 'empty')
FAIL expects wrong text
```

??? success "Show a solution"

    ```
    from playwright.sync_api import sync_playwright, expect

    LOGIN_HTML = '<form id="f"><label>Email <input id="email"></label><label>Password <input id="pw" type="password"></label>\n<button type="submit">Sign in</button><p id="msg" role="alert"></p></form>\n<script>document.getElementById("f").addEventListener("submit",e=>{e.preventDefault();\nconst ok=document.getElementById("email").value==="ada@example.com"&&document.getElementById("pw").value==="s3cret";\ndocument.getElementById("msg").textContent=ok?"Welcome, Ada":"Invalid credentials";});</script>'

    class LoginPage:
        def __init__(self, page):
            self.page = page
            self.email = page.get_by_label("Email")
            self.password = page.get_by_label("Password")
            self.submit = page.get_by_role("button", name="Sign in")
            self.message = page.get_by_role("alert")

        def login(self, email, password):
            self.email.fill(email)
            self.password.fill(password)
            self.submit.click()
            return self

    def check(name, fn):
        try:
            fn()
            print("PASS", name)
        except AssertionError:
            print("FAIL", name)

    with sync_playwright() as p:
        browser = p.chromium.launch()
        for email, pw, expected in [("ada@example.com", "s3cret", "Welcome, Ada"),
                                    ("ada@example.com", "wrong", "Invalid credentials"),
                                    ("", "", "Invalid credentials")]:
            context = browser.new_context()            # fresh, isolated state per test
            page = context.new_page()
            page.set_content(LOGIN_HTML)
            page.set_default_timeout(1000)
            expect.set_options(timeout=1000)
            lp = LoginPage(page).login(email, pw)
            check(f"login({email or 'empty'!r}, {pw or 'empty'!r})", lambda: expect(lp.message).to_have_text(expected))
            context.close()
        # a deliberately wrong expectation shows how a failing check is reported
        context = browser.new_context(); page = context.new_page(); page.set_content(LOGIN_HTML)
        expect.set_options(timeout=500)
        check("expects wrong text", lambda: expect(LoginPage(page).login("ada@example.com", "s3cret").message).to_have_text("Goodbye"))
        browser.close()
    ```

    The page object owns the locators and actions; tests only describe intent. Each case gets a new context for isolation, and expect() retries until the message appears. Verified by running the solution with Playwright for Python and Chromium.

### Mock an API and read a table without sleeping

Implement run_case(browser, items): route the items endpoint to a canned JSON response, load the page, wait for #status to read "ready" with expect (no sleep), and read the table into dicts with read_table. Then report rows, the cheapest item, the total and the names.

Starter code:

```
from playwright.sync_api import sync_playwright, expect

PAGE = """<table id="t"><thead><tr><th>Item</th><th>Price</th></tr></thead><tbody id="b"></tbody></table><p id="status">loading</p>
<script>
setTimeout(() => fetch("http://shop.test/api/items").then(r => r.json()).then(items => {
  document.getElementById("b").innerHTML = items.map(i => `<tr><td>${i.name}</td><td>${i.price}</td></tr>`).join("");
  document.getElementById("status").textContent = "ready";
}), 200);
</script>"""

def read_table(page):
    # return a list of {"name": str, "price": float} dicts, one per row of #b tr
    return []

def run_case(browser, items):
    # new page; route http://shop.test/api/items to fulfill(json=items); set_content(PAGE);
    # wait with expect(...).to_have_text("ready") on #status (never sleep); read the table; close the page; return the data
    return []

with sync_playwright() as p:
    browser = p.chromium.launch()
    data = run_case(browser, [{"name": "Pen", "price": 2.5}, {"name": "Book", "price": 12}, {"name": "Bag", "price": 30.25}])
    print("rows:", len(data))
    print("cheapest:", min(data, key=lambda r: r["price"])["name"])
    print("total:", f"{sum(r['price'] for r in data):.2f}")
    print("names:", ", ".join(sorted(r["name"] for r in data)))
    empty = run_case(browser, [])
    print("empty rows:", len(empty))
    browser.close()
```

Expected output:

```text
rows: 3
cheapest: Pen
total: 44.75
names: Bag, Book, Pen
empty rows: 0
```

??? success "Show a solution"

    ```
    from playwright.sync_api import sync_playwright, expect

    PAGE = """<table id="t"><thead><tr><th>Item</th><th>Price</th></tr></thead><tbody id="b"></tbody></table><p id="status">loading</p>
    <script>
    setTimeout(() => fetch("http://shop.test/api/items").then(r => r.json()).then(items => {
      document.getElementById("b").innerHTML = items.map(i => `<tr><td>${i.name}</td><td>${i.price}</td></tr>`).join("");
      document.getElementById("status").textContent = "ready";
    }), 200);
    </script>"""

    def read_table(page):
        rows = page.locator("#b tr")
        return [{"name": rows.nth(i).locator("td").nth(0).text_content(),
                 "price": float(rows.nth(i).locator("td").nth(1).text_content())}
                for i in range(rows.count())]

    def run_case(browser, items):
        page = browser.new_page()
        page.set_default_timeout(3000)
        page.route("http://shop.test/api/items", lambda route: route.fulfill(json=items))
        page.set_content(PAGE)
        expect(page.locator("#status")).to_have_text("ready")     # waits for the async render, no sleep
        data = read_table(page)
        page.close()
        return data

    with sync_playwright() as p:
        browser = p.chromium.launch()
        data = run_case(browser, [{"name": "Pen", "price": 2.5}, {"name": "Book", "price": 12}, {"name": "Bag", "price": 30.25}])
        print("rows:", len(data))
        print("cheapest:", min(data, key=lambda r: r["price"])["name"])
        print("total:", f"{sum(r['price'] for r in data):.2f}")
        print("names:", ", ".join(sorted(r["name"] for r in data)))
        empty = run_case(browser, [])
        print("empty rows:", len(empty))
        browser.close()
    ```

    route.fulfill makes the data deterministic, expect waits for the asynchronous render, and locator.nth(i) reads the rows. Verified by running the solution with Playwright for Python and Chromium.

---

More free practice: the [Playwright Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
