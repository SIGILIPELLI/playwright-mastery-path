# 01 · Page Object Model

## Why not just write scripts?

A script like Level 1's login flow is fine for one test. Once you have
dozens of tests touching the same login page, search bar, or product page,
duplicating locators everywhere becomes a liability: change one CSS class
on the real site and you're editing twenty test files. The Page Object
Model (POM) fixes this by giving each page (or meaningful component) of the
app under test a single Python class that owns its locators and the actions
a user can take on it. Tests then read like user stories, not DOM archaeology.

```python
# pages/login_page.py
from playwright.sync_api import Page

class LoginPage:
    def __init__(self, page: Page):
        self.page = page
        self.email = page.get_by_label("Email")
        self.password = page.get_by_label("Password")
        self.submit = page.get_by_role("button", name="Sign in")
        self.error = page.get_by_text("Invalid credentials")

    def goto(self):
        self.page.goto("https://example.com/login")

    def login(self, email: str, password: str):
        self.email.fill(email)
        self.password.fill(password)
        self.submit.click()
```

```text
# no output — this module defines a class; nothing runs until
# a test instantiates LoginPage(page) and calls its methods
```

Notice locators are built once in `__init__` and stored as attributes —
that's safe because a `Locator` is a lazy description, not a live element
reference (Level 1, Module 4), so building it before the page has even
navigated is harmless.

## Using the page object in a test

```python
# tests/test_login.py
from playwright.sync_api import Page, expect
from pages.login_page import LoginPage

def test_valid_login_redirects_to_dashboard(page: Page):
    login = LoginPage(page)
    login.goto()
    login.login("user@example.com", "correct-password")
    expect(page).to_have_url("https://example.com/dashboard")

def test_invalid_login_shows_error(page: Page):
    login = LoginPage(page)
    login.goto()
    login.login("user@example.com", "wrong-password")
    expect(login.error).to_be_visible()
```

```text
2 passed in 1.84s
```

The test file no longer knows *how* the login form is built — no CSS
selectors, no knowledge of which field comes first. If the login page's
markup changes, only `login_page.py` needs an update, and every test that
uses it is automatically fixed.

## A base page for shared behavior

Most real apps share chrome — a nav bar, a toast/notification area, a
loading spinner. Put that in a base class other page objects inherit from.

```python
# pages/base_page.py
from playwright.sync_api import Page, expect

class BasePage:
    def __init__(self, page: Page):
        self.page = page
        self.toast = page.locator(".toast-message")

    def expect_toast(self, text: str):
        expect(self.toast).to_have_text(text)

    def open_nav_menu(self):
        self.page.get_by_role("button", name="Menu").click()
```

```python
# pages/dashboard_page.py
from playwright.sync_api import Page
from pages.base_page import BasePage

class DashboardPage(BasePage):
    def __init__(self, page: Page):
        super().__init__(page)
        self.welcome_heading = page.get_by_role("heading", name="Welcome")

    def logout(self):
        self.open_nav_menu()
        self.page.get_by_role("menuitem", name="Log out").click()
```

```text
# no output — DashboardPage now has both its own locators
# and everything BasePage provides (toast, nav menu)
```

## Component objects for repeated widgets

Not everything maps 1:1 to a URL. A product card, a data-table row, or a
modal dialog that appears on several pages deserves its own small class,
constructed from a `Locator` rather than a `Page`.

```python
# pages/components/product_card.py
from playwright.sync_api import Locator

class ProductCard:
    def __init__(self, root: Locator):
        self.root = root
        self.title = root.locator("h3")
        self.add_to_cart = root.get_by_role("button", name="Add to cart")

    def name(self) -> str:
        return self.title.text_content()
```

```python
# usage inside a test or page object
cards = page.locator(".product-card")
for i in range(cards.count()):
    card = ProductCard(cards.nth(i))
    if "Mouse" in card.name():
        card.add_to_cart.click()
        break
```

```text
# no output — ProductCard wraps one scoped Locator per card,
# so add_to_cart.click() only ever targets that specific card
```

## What POM is *not*

A page object should expose **actions and state**, not assertions about
business logic. Keep `expect(...)` calls in the test file (or in small
helper methods that clearly assert one thing), and keep multi-step business
workflows (e.g., "complete checkout") out of a single page's class — those
belong in a higher-level "flow" or in the test itself composing multiple
page objects. Page objects that grow into 500-line god classes doing
assertions, waiting, *and* orchestration are a sign the boundaries slipped.

## Exercise

Using `https://demoqa.com/login` (a public demo site with a real, if flaky,
login form):

1. Create `pages/base_page.py` with a `BasePage` class holding a `page`
   reference and one shared locator you notice is common across demoqa
   pages (e.g. the top banner ad container, so future page objects can
   assert it doesn't cover the form).
2. Create `pages/login_page.py` with a `LoginPage(BasePage)` that exposes
   `username`, `password`, `login_button` locators (found via
   `get_by_placeholder` — demoqa's inputs use placeholders) and a `login()`
   method.
3. Write two tests in `tests/test_demoqa_login.py`: one submitting a
   deliberately invalid username and asserting `#name` (the error banner
   demoqa shows) becomes visible; one leaving fields blank and asserting the
   fields get an `is-invalid` class via `expect(locator).to_have_class(...)`.
4. Refactor: notice both tests repeat `LoginPage(page); login.goto()` —
   pull that into a pytest fixture (previewed here, covered fully next
   module) named `login_page` and use it in both tests instead.
