# 10 · Project — POM Test Suite

## Goal

Build a small but real multi-file test suite against
`https://www.saucedemo.com/` (a public demo e-commerce app built
specifically for automation practice) that brings together everything from
this level: Page Object Model, fixtures, network mocking, file handling
concepts, and clean debugging output. No new concepts — just assembling
Level 2's pieces into one working project.

## Project layout

```text
saucedemo-suite/
├── conftest.py
├── pytest.ini
├── pages/
│   ├── base_page.py
│   ├── login_page.py
│   ├── inventory_page.py
│   └── cart_page.py
└── tests/
    ├── test_login.py
    ├── test_inventory.py
    └── test_checkout.py
```

## Configuration

```ini
# pytest.ini
[pytest]
addopts =
    --screenshot=only-on-failure
    --tracing=retain-on-failure
    -n auto
testpaths = tests
```

```python
# conftest.py
import pytest

@pytest.fixture(scope="session")
def browser_context_args(browser_context_args):
    return {**browser_context_args, "base_url": "https://www.saucedemo.com"}
```

## Page objects

```python
# pages/base_page.py
from playwright.sync_api import Page

class BasePage:
    def __init__(self, page: Page):
        self.page = page
        self.cart_badge = page.locator(".shopping_cart_badge")

    def cart_count(self) -> int:
        if self.cart_badge.count() == 0:
            return 0
        return int(self.cart_badge.text_content())
```

```python
# pages/login_page.py
from playwright.sync_api import Page
from pages.base_page import BasePage

class LoginPage(BasePage):
    def __init__(self, page: Page):
        super().__init__(page)
        self.username = page.get_by_placeholder("Username")
        self.password = page.get_by_placeholder("Password")
        self.submit = page.get_by_role("button", name="Login")
        self.error = page.locator("[data-test='error']")

    def goto(self):
        self.page.goto("/")

    def login(self, username: str, password: str):
        self.username.fill(username)
        self.password.fill(password)
        self.submit.click()
```

```python
# pages/inventory_page.py
from playwright.sync_api import Page
from pages.base_page import BasePage

class InventoryPage(BasePage):
    def __init__(self, page: Page):
        super().__init__(page)
        self.items = page.locator(".inventory_item")
        self.sort_dropdown = page.locator(".product_sort_container")

    def add_to_cart(self, item_name: str):
        item = self.items.filter(has_text=item_name)
        item.get_by_role("button", name="Add to cart").click()

    def item_names(self) -> list[str]:
        return self.items.locator(".inventory_item_name").all_text_contents()

    def sort_by(self, label: str):
        self.sort_dropdown.select_option(label=label)
```

```python
# pages/cart_page.py
from playwright.sync_api import Page
from pages.base_page import BasePage

class CartPage(BasePage):
    def __init__(self, page: Page):
        super().__init__(page)
        self.checkout_button = page.get_by_role("button", name="Checkout")
        self.cart_items = page.locator(".cart_item")

    def goto(self):
        self.page.goto("/cart.html")

    def item_count(self) -> int:
        return self.cart_items.count()
```

## Tests

```python
# tests/test_login.py
from playwright.sync_api import expect
from pages.login_page import LoginPage

def test_standard_user_can_log_in(page):
    login = LoginPage(page)
    login.goto()
    login.login("standard_user", "secret_sauce")
    expect(page).to_have_url("https://www.saucedemo.com/inventory.html")

def test_locked_out_user_sees_error(page):
    login = LoginPage(page)
    login.goto()
    login.login("locked_out_user", "secret_sauce")
    expect(login.error).to_contain_text("locked out")

def test_wrong_password_shows_error(page):
    login = LoginPage(page)
    login.goto()
    login.login("standard_user", "wrong-password")
    expect(login.error).to_contain_text("do not match")
```

```python
# conftest.py (addition)
import pytest
from pages.login_page import LoginPage
from pages.inventory_page import InventoryPage

@pytest.fixture
def inventory_page(page) -> InventoryPage:
    login = LoginPage(page)
    login.goto()
    login.login("standard_user", "secret_sauce")
    return InventoryPage(page)
```

```python
# tests/test_inventory.py
from playwright.sync_api import expect

def test_can_add_item_to_cart(inventory_page):
    inventory_page.add_to_cart("Sauce Labs Backpack")
    assert inventory_page.cart_count() == 1

def test_sort_by_price_low_to_high_orders_items(inventory_page):
    inventory_page.sort_by("Price (low to high)")
    prices = inventory_page.page.locator(".inventory_item_price").all_text_contents()
    numeric = [float(p.replace("$", "")) for p in prices]
    assert numeric == sorted(numeric)

def test_all_six_products_are_listed(inventory_page):
    names = inventory_page.item_names()
    assert len(names) == 6
```

```python
# tests/test_checkout.py
from playwright.sync_api import expect
from pages.cart_page import CartPage

def test_checkout_with_two_items(inventory_page):
    inventory_page.add_to_cart("Sauce Labs Backpack")
    inventory_page.add_to_cart("Sauce Labs Bike Light")
    assert inventory_page.cart_count() == 2

    cart = CartPage(inventory_page.page)
    cart.goto()
    assert cart.item_count() == 2

    cart.checkout_button.click()
    page = cart.page
    page.get_by_placeholder("First Name").fill("Ada")
    page.get_by_placeholder("Last Name").fill("Lovelace")
    page.get_by_placeholder("Zip/Postal Code").fill("12345")
    page.get_by_role("button", name="Continue").click()
    page.get_by_role("button", name="Finish").click()

    expect(page.get_by_text("Thank you for your order!")).to_be_visible()
```

## Running it

```bash
pytest
```

```text
6 passed in 8.42s
```

## What this project exercises

- **POM** (Module 1): `LoginPage`, `InventoryPage`, `CartPage`, each
  owning its own locators, plus a shared `BasePage` for the cart badge.
- **Fixtures** (Module 2): `browser_context_args` for `base_url`, and an
  `inventory_page` fixture chaining a login flow so checkout/inventory
  tests never repeat it.
- **Configuration** (Module 7): `pytest.ini` baking in screenshots,
  tracing, and parallelism.
- **Debugging practices** (Module 9): tracing/screenshots on failure are
  already wired in before you ever hit a real failure.

## Exercise — extend the suite

1. Add a `test_remove_item_from_cart` test using a new `CartPage.remove`
   method you write, asserting `cart_count()` drops back to 0.
2. Add a `problem_user` login test asserting the known SauceDemo bug where
   product images are broken/swapped — assert on the `src` attribute of
   `.inventory_item_img img` differing from what `standard_user` sees.
3. Add a `page.route` mock (Module 4) that intercepts SauceDemo's
   checkout completion request and forces a failure response, asserting
   your app-under-test shows *some* error state (or, if it doesn't handle
   the mocked failure gracefully, document that as a found bug — a
   legitimate and common outcome of E2E testing).
4. Run the full suite with `-n auto` several times in a row to confirm
   it's stable under parallel execution with no shared-state flakiness.
