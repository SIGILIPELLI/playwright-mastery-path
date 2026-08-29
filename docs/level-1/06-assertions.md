# 06 · Assertions with expect

## Why `expect` instead of plain `assert`

```python
from playwright.sync_api import Page, expect

def test_login_shows_dashboard(page: Page):
    page.goto("https://www.saucedemo.com/")
    page.get_by_placeholder("Username").fill("standard_user")
    page.get_by_placeholder("Password").fill("secret_sauce")
    page.get_by_role("button", name="Login").click()

    expect(page.get_by_text("Products")).to_be_visible()
```

```text
tests/test_login.py::test_login_shows_dashboard PASSED                   [100%]
```

You could write `assert page.get_by_text("Products").is_visible()` instead —
but `is_visible()` checks the DOM at the exact instant it's called and
returns `False` immediately if the element isn't there *yet*, even if it
would appear one frame later. `expect(...).to_be_visible()` **retries** the
check against the live page until it passes or a timeout elapses (5 seconds
by default). This single difference eliminates the majority of timing-related
flakiness in E2E suites — you stop needing to guess how long to wait before
asserting.

```python
# fragile — checks once, right now, no retry:
assert page.get_by_text("Products").is_visible()

# robust — polls until it's true or times out:
expect(page.get_by_text("Products")).to_be_visible()
```

```text
# the assert version fails immediately if the DOM update hasn't
# landed yet; expect() gives the app up to 5s (configurable) to
# finish rendering before failing
```

## Common assertions

```python
from playwright.sync_api import expect

expect(page).to_have_title("Swag Labs")
expect(page).to_have_url("https://www.saucedemo.com/inventory.html")

expect(page.get_by_text("Products")).to_be_visible()
expect(page.get_by_text("Out of stock")).to_be_hidden()

expect(page.get_by_role("button", name="Login")).to_be_enabled()
expect(page.get_by_role("button", name="Checkout")).to_be_disabled()

expect(page.get_by_label("Remember me")).to_be_checked()
expect(page.locator(".error-message")).to_have_text("Invalid credentials")
expect(page.locator(".error-message")).to_contain_text("Invalid")

expect(page.locator(".product")).to_have_count(6)
expect(page.get_by_role("link", name="Cart")).to_have_attribute("href", "/cart")
```

```text
# each raises an AssertionError with a readable diff after
# retrying for up to the timeout if the condition never becomes true
```

| Assertion | Checks |
|---|---|
| `to_be_visible()` / `to_be_hidden()` | Rendered and occupying layout space, vs. not |
| `to_be_enabled()` / `to_be_disabled()` | Interactive state |
| `to_be_checked()` | Checkbox/radio state |
| `to_have_text(...)` | Exact (trimmed) text content |
| `to_contain_text(...)` | Substring match |
| `to_have_value(...)` | Current value of an input |
| `to_have_count(n)` | Number of elements a locator matches |
| `to_have_attribute(name, value)` | A specific HTML attribute |
| `to_have_url(...)` / `to_have_title(...)` | Page-level state |

## Reading a failure

```python
def test_wrong_credentials_shows_error(page):
    page.goto("https://www.saucedemo.com/")
    page.get_by_placeholder("Username").fill("standard_user")
    page.get_by_placeholder("Password").fill("wrong_password")
    page.get_by_role("button", name="Login").click()

    expect(page.locator("[data-test='error']")).to_have_text(
        "Epic sadface: Wrong password"
    )
```

If the copy actually reads slightly differently, the failure looks like this:

```text
AssertionError: Locator expected to have text "Epic sadface: Wrong password"
Actual value: Epic sadface: Username and password do not match any user in this service

Call log:
  - expect.to_have_text with timeout 5000ms
  - waiting for locator("[data-test='error']")
  -   locator resolved to <h3 data-test="error">…</h3>
  -   unexpected value "Epic sadface: Username and password do not match any user in this service"
  - retrying …
```

The call log shows the resolved element and every retry — you can see
exactly what the locator found and how its text changed (or didn't) across
retries, which is usually enough to tell whether the bug is in your
assertion or in the app.

## Soft assertions

Playwright Python doesn't ship a built-in "soft assert that collects failures
and reports them all at the end" the way some frameworks do; each `expect(...)`
call raises immediately on failure. For test-suite-level soft-assertion
behavior, pair pytest with a plugin such as `pytest-check`, or structure the
test so each independently meaningful check is its own test function — which
also gives you a clearer failure name per check (Module 8 of
Python Testing Mastery Path covers the "one logical assertion per test"
principle this maps to).

## Negating an assertion

```python
expect(page.get_by_text("Loading...")).not_to_be_visible()
expect(page.get_by_role("button", name="Submit")).not_to_be_disabled()
```

```text
# no output on success — not_to_* still retries, waiting for
# the condition to become false rather than checking once
```

`not_to_be_visible()` is not the same as "never became visible" — it means
"is not visible *right now, after waiting up to the timeout for it to
disappear*." If the element was visible a moment ago and disappears within
the timeout window, this still passes. That's usually exactly what you
want (waiting for a spinner to go away), but be aware it wouldn't catch a
spinner that flashes visible-then-hidden faster than you could ever observe.

## Custom timeout per assertion

```python
expect(page.get_by_text("Report generated")).to_be_visible(timeout=15000)
```

```text
# waits up to 15 seconds instead of the default 5 for this
# specific assertion — useful for a known-slow operation
# (report generation, a large file upload) without loosening
# the default for every other assertion in the suite
```

## Exercise

Using `https://www.saucedemo.com/` (`standard_user` / `secret_sauce`):

1. Write a test that logs in and asserts `expect(page).to_have_url(...)`
   for the inventory page, and `expect(page.locator(".inventory_item")).to_have_count(6)`.
2. Write a second test using the wrong password and assert the exact error
   text shown, using `to_have_text()` (find the real copy by running the
   flow manually first — don't guess).
3. Add an item to the cart and assert the cart badge
   (`.shopping_cart_badge`) shows `"1"` with `to_have_text()`, then add a
   second item and assert it updates to `"2"`.
4. Assert that the "Checkout" button on the cart page is enabled with
   `to_be_enabled()`, and that removing all items makes the cart badge
   disappear entirely — assert this with `not_to_be_visible()` rather than
   checking for empty text.
5. Deliberately assert an incorrect expected value (e.g. the wrong cart
   count) and read the full failure output, including the call log — note
   how it differs from a plain Python `AssertionError`.
