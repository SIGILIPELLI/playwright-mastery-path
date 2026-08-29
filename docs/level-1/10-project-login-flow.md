# 10 · Project — Automate a Login Flow

This project ties together every Module 1–9 concept into one small but real
test file: project setup, first script structure, locators, actions,
`expect` assertions, waiting, forms, and headed/headless execution. It uses
`https://www.saucedemo.com/`, a public demo store built specifically for
automation practice, with a known set of test accounts.

## Goal

Write a `pytest` test file that:

1. Logs in with valid credentials and asserts success.
2. Logs in with invalid credentials and asserts the specific error shown.
3. Logs in as a known "locked out" user and asserts that specific error.
4. Adds an item to the cart after logging in and asserts the cart badge.
5. Logs out and confirms you're returned to the login page.

## Project setup

```text
playwright-practice/
├── pytest.ini
└── tests/
    └── test_login_flow.py
```

`pytest.ini`:

```ini
[pytest]
testpaths = tests
addopts = --browser chromium
```

## The test file

```python
# tests/test_login_flow.py
from playwright.sync_api import Page, expect

BASE_URL = "https://www.saucedemo.com/"


def login(page: Page, username: str, password: str) -> None:
    page.goto(BASE_URL)
    page.get_by_placeholder("Username").fill(username)
    page.get_by_placeholder("Password").fill(password)
    page.get_by_role("button", name="Login").click()


def test_valid_login_reaches_inventory_page(page: Page):
    login(page, "standard_user", "secret_sauce")

    expect(page).to_have_url(BASE_URL + "inventory.html")
    expect(page.get_by_text("Products")).to_be_visible()


def test_invalid_password_shows_error(page: Page):
    login(page, "standard_user", "wrong_password")

    error = page.locator("[data-test='error']")
    expect(error).to_be_visible()
    expect(error).to_contain_text("Username and password do not match")
    expect(page).to_have_url(BASE_URL)  # never navigated away


def test_locked_out_user_shows_locked_error(page: Page):
    login(page, "locked_out_user", "secret_sauce")

    error = page.locator("[data-test='error']")
    expect(error).to_contain_text("Sorry, this user has been locked out")


def test_login_then_add_item_updates_cart_badge(page: Page):
    login(page, "standard_user", "secret_sauce")

    first_product = page.locator(".inventory_item").first
    product_name = first_product.locator(".inventory_item_name").text_content()
    first_product.get_by_role("button", name="Add to cart").click()

    badge = page.locator(".shopping_cart_badge")
    expect(badge).to_have_text("1")

    page.locator(".shopping_cart_link").click()
    expect(page.locator(".inventory_item_name")).to_have_text(product_name)


def test_logout_returns_to_login_page(page: Page):
    login(page, "standard_user", "secret_sauce")
    expect(page).to_have_url(BASE_URL + "inventory.html")

    page.locator("#react-burger-menu-btn").click()
    page.get_by_role("link", name="Logout").click()

    expect(page).to_have_url(BASE_URL)
    expect(page.get_by_role("button", name="Login")).to_be_visible()
```

## Running it

```bash
pytest tests/test_login_flow.py -v
```

```text
tests/test_login_flow.py::test_valid_login_reaches_inventory_page PASSED  [ 20%]
tests/test_login_flow.py::test_invalid_password_shows_error PASSED       [ 40%]
tests/test_login_flow.py::test_locked_out_user_shows_locked_error PASSED [ 60%]
tests/test_login_flow.py::test_login_then_add_item_updates_cart_badge PASSED [ 80%]
tests/test_login_flow.py::test_logout_returns_to_login_page PASSED       [100%]

============================== 5 passed in 4.87s ===============================
```

```bash
pytest tests/test_login_flow.py --headed --slowmo 200
```

```text
tests/test_login_flow.py::test_valid_login_reaches_inventory_page PASSED  [ 20%]
tests/test_login_flow.py::test_invalid_password_shows_error PASSED       [ 40%]
tests/test_login_flow.py::test_locked_out_user_shows_locked_error PASSED [ 60%]
tests/test_login_flow.py::test_login_then_add_item_updates_cart_badge PASSED [ 80%]
tests/test_login_flow.py::test_logout_returns_to_login_page PASSED       [100%]

============================== 5 passed in 12.41s ===============================
```

## What this project deliberately practices

- **A helper function (`login`)** instead of repeating the same three lines
  in every test — a small preview of the Page Object Model formalized in
  Level 2 Module 1.
- **Testing the negative path** (wrong password, locked-out account) with
  the same rigor as the happy path — a login flow that's only ever tested
  with correct credentials hasn't actually verified the error handling a
  real user will eventually hit.
- **Asserting on state that changed, not just "no crash happened"** — the
  cart badge count and the product name carried across pages are concrete,
  checkable facts, not vague "did anything look wrong" checks.
- **`expect(...)` everywhere**, never a plain `assert` on a live-fetched DOM
  value, so the suite tolerates normal small rendering delays without being
  flaky.

## Extending it further (optional)

- Parametrize the invalid-login test over multiple bad usernames/passwords
  using `@pytest.mark.parametrize`, asserting the same error path each time.
- Add a test for the `problem_user` account (another built-in saucedemo
  test account) and see if you can spot the intentionally broken behavior
  it exhibits — a good exercise in noticing an app defect through E2E
  testing rather than being told about it in advance.
- Run the whole file with `--browser firefox` and `--browser webkit` and
  confirm all five tests still pass identically across engines.

## Exercise

1. Build the project exactly as shown and get all 5 tests passing headless.
2. Re-run headed with `--slowmo 200` and watch each test execute — confirm
   your mental model of each step matches what you see on screen.
3. Add a parametrized version of the invalid-login test covering at least
   three different bad credential combinations from a single test function.
4. Investigate the `problem_user` account: log in with it, add an item to
   the cart, and see if the product image or some other element looks
   wrong compared to `standard_user`. Write one sentence describing the bug
   you found — this is the exact kind of defect E2E testing exists to catch.
5. Run the full file against all three engines (`--browser chromium`,
   `--browser firefox`, `--browser webkit`) and confirm identical results,
   completing Level 1.
