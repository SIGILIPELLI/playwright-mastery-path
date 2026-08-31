# 08 · Migrating from Selenium

Teams rarely rewrite an entire Selenium suite overnight. This module covers
both the concept mapping (so existing Selenium knowledge transfers instead
of feeling thrown away) and a practical strategy for migrating incrementally
without a multi-month test-freeze.

## Concept mapping

```text
Selenium (WebDriver)              Playwright
─────────────────────────────────────────────────────────────
WebDriver                         Browser + BrowserContext
driver.find_element(By.ID, "x")   page.locator("#x")
driver.find_elements(...)         locator (already plural; use .all())
WebDriverWait + expected_conditions  built into every action + expect()
driver.get(url)                   page.goto(url)
element.send_keys(text)           locator.fill(text) / locator.type(text)
element.click()                   locator.click()
Actions class (hover, drag)       page.hover(), locator.drag_to()
driver.switch_to.frame(...)       page.frame_locator(...)
driver.switch_to.window(...)      context.pages / page.on("popup")
Page Object Model                 same pattern, same intent — locators
                                   just get lazier and auto-waiting
```

## The single biggest mental shift: explicit waits become unnecessary

```python
# Selenium — waiting is your responsibility, everywhere
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

wait = WebDriverWait(driver, 10)
button = wait.until(EC.element_to_be_clickable((By.ID, "submit")))
button.click()

# Playwright — the click call itself waits
page.locator("#submit").click()
```

```text
# porting a Selenium test 1:1 by wrapping every action in an
# explicit wait is not wrong, but it's redundant — Playwright's
# actionability checks (attached, visible, stable, enabled,
# receives events) already run before every action; the
# migration payoff is largely in DELETING the wait boilerplate,
# not adding a Playwright-flavored equivalent of it
```

## Handling `time.sleep()` calls carried over from Selenium

```python
# carried-over anti-pattern, now doubly unnecessary
page.locator("#save").click()
time.sleep(2)
assert page.locator("#success-message").is_visible()

# correct Playwright equivalent
page.locator("#save").click()
expect(page.locator("#success-message")).to_be_visible()
```

```text
# is_visible() alone (not wrapped in expect) returns the
# CURRENT state instantly with no retry — a common migration
# bug is replacing sleep+assert with is_visible() and calling
# it done, when it still has the same race condition the sleep
# was papering over; expect() is what actually adds the
# auto-retrying wait
```

## An incremental migration strategy

```text
1. New tests: write in Playwright from day one, even while the
   bulk of the suite is still Selenium — don't let the backlog
   grow.
2. Shared CI: run both suites as separate jobs in the same
   pipeline so nobody loses coverage during the transition.
3. Migrate by risk, not by ease: prioritize the flakiest,
   highest-maintenance Selenium tests first — that's where
   Playwright's auto-waiting pays off fastest and most visibly,
   building the case for continuing.
4. Delete the Selenium test the same PR that adds its Playwright
   replacement — don't let both versions of the same test
   coexist indefinitely; that's strictly worse than either
   suite alone (double maintenance, double flakiness surface).
5. Track migration % as a real metric so the effort doesn't
   silently stall at 60%.
```

## Side-by-side worked example: a login test

```python
# Selenium version
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

def test_login_selenium():
    driver = webdriver.Chrome()
    driver.get("https://example.com/login")
    driver.find_element(By.ID, "email").send_keys("user@example.com")
    driver.find_element(By.ID, "password").send_keys("secret")
    driver.find_element(By.ID, "submit").click()
    wait = WebDriverWait(driver, 10)
    heading = wait.until(EC.presence_of_element_located((By.TAG_NAME, "h1")))
    assert heading.text == "Dashboard"
    driver.quit()
```

```python
# Playwright equivalent
from playwright.sync_api import sync_playwright, expect

def test_login_playwright():
    with sync_playwright() as p:
        browser = p.chromium.launch()
        page = browser.new_page()
        page.goto("https://example.com/login")
        page.get_by_label("Email").fill("user@example.com")
        page.get_by_label("Password").fill("secret")
        page.get_by_role("button", name="Sign in").click()
        expect(page.get_by_role("heading")).to_have_text("Dashboard")
        browser.close()
```

```text
# note the locator style change too, not just the API: the
# migration is also a good opportunity to move from ID-based
# selectors (fragile, meaningless to future readers) to
# role/label locators (Level 1), rather than mechanically
# translating By.ID("email") into page.locator("#email")
```

## Exercise

1. Take one real Selenium test from an existing suite and port it to
   Playwright using the concept mapping table above, replacing every
   `WebDriverWait`/`expected_conditions` call with an equivalent
   auto-waiting action or `expect()`.
2. Find (or introduce) a `time.sleep()` in a ported test and replace it with
   a proper `expect()` assertion; explain in a comment what race condition
   the sleep was actually hiding.
3. Set up a CI pipeline running both a Selenium job and a Playwright job as
   separate steps, simulating a mid-migration repository.
4. Pick the flakiest test in a real or sample Selenium suite, migrate it
   first, and write a short note on why its flakiness is expected to
   improve under Playwright's actionability model specifically.
