# 08 · Handling Forms & Dropdowns

## A complete form submission

```python
from playwright.sync_api import Page, expect

def test_contact_form_submission(page: Page):
    page.goto("https://www.saucedemo.com/")
    page.get_by_placeholder("Username").fill("standard_user")
    page.get_by_placeholder("Password").fill("secret_sauce")
    page.get_by_role("button", name="Login").click()

    expect(page).to_have_url("https://www.saucedemo.com/inventory.html")
```

```text
tests/test_form.py::test_contact_form_submission PASSED                  [100%]
```

The general shape of a form test: fill every field, submit, assert on the
result — either a success indicator, a navigation, or an error message if
you're testing validation.

## Text inputs and textareas

```python
page.get_by_label("Full name").fill("Priya Sharma")
page.get_by_label("Full name").fill("")          # clear it
page.get_by_label("Comments").fill("Line one\nLine two")
```

```text
# no output — fill() replaces the entire current value; passing
# "" clears the field; \n inserts a literal newline into a
# multi-line textarea
```

## Native `<select>` dropdowns

```python
page.get_by_label("Country").select_option("India")
page.get_by_label("Country").select_option(value="IN")
page.get_by_label("Country").select_option(index=2)

selected = page.get_by_label("Country").input_value()
print(selected)
```

```text
India
```

`select_option` accepts a visible label, an `<option value="...">`, or a
zero-based index — pick whichever is stable across the page's likely future
changes. A `value="IN"` attribute set intentionally by the backend is usually
more stable than a visible label copy that a designer might reword.

## Custom (non-native) dropdowns

Many production UIs replace `<select>` with a styled `<div>`-based dropdown
for visual control. These need to be driven as a click-to-open,
click-to-choose sequence rather than `select_option`, because there's no
underlying `<select>` element for Playwright's dropdown-specific API to
target:

```python
page.get_by_role("combobox", name="Country").click()
page.get_by_role("option", name="India").click()

expect(page.get_by_role("combobox", name="Country")).to_have_text("India")
```

```text
# no output — a well-built custom dropdown exposes role="combobox"
# on the trigger and role="option" on each choice, so get_by_role
# still works even though it isn't a native <select>
```

If a custom dropdown doesn't expose proper ARIA roles, that's itself an
accessibility bug worth flagging — but pragmatically, you may need to fall
back to a CSS locator scoped to the dropdown's known container until it's
fixed.

## Radio buttons and checkbox groups

```python
page.get_by_label("Standard shipping").check()
assert page.get_by_label("Standard shipping").is_checked()
assert not page.get_by_label("Express shipping").is_checked()

page.get_by_label("Gift wrap").check()
page.get_by_label("Send updates via SMS").check()
```

```text
# no output — checking one radio in a native <input type="radio">
# group automatically unchecks the others in that group, exactly
# as it would for a real user
```

## Client-side validation

```python
def test_email_field_rejects_invalid_format(page: Page):
    page.goto("https://example.com/signup")
    page.get_by_label("Email").fill("not-an-email")
    page.get_by_role("button", name="Sign up").click()

    expect(page.get_by_text("Enter a valid email address")).to_be_visible()

def test_required_field_blocks_submission(page: Page):
    page.goto("https://example.com/signup")
    page.get_by_role("button", name="Sign up").click()

    expect(page.get_by_text("Full name is required")).to_be_visible()
    expect(page).to_have_url("https://example.com/signup")   # didn't navigate away
```

```text
# each asserts on the visible error and, in the second case,
# also confirms the app didn't proceed past validation —
# testing the negative case is as important as the happy path
```

## Reading back form state for assertions

```python
expect(page.get_by_label("Email")).to_have_value("user@example.com")
expect(page.get_by_label("Terms and conditions")).to_be_checked()
expect(page.get_by_role("button", name="Submit")).to_be_disabled()
```

```text
# no output on success — to_have_value/to_be_checked/to_be_disabled
# retry like every other expect() assertion, so they tolerate a
# brief delay before client-side JS finishes wiring up the form
```

Asserting on form *state* this way (rather than just "no error shown") is
what catches a real class of bug: a submit button that stays disabled after
valid input because a validation flag never flipped, which a purely
happy-path click-and-hope test would miss entirely.

## Handling native browser dialogs

Forms sometimes trigger `confirm()`, `alert()`, or `prompt()` — these are
OS-level dialogs Playwright must be told how to handle *before* they appear,
because they block all further page JavaScript until dismissed:

```python
page.on("dialog", lambda dialog: dialog.accept())
page.get_by_role("button", name="Delete account").click()

def handle_prompt(dialog):
    print("Prompt text:", dialog.message)
    dialog.accept("my typed response")

page.on("dialog", handle_prompt)
page.get_by_role("button", name="Rename").click()
```

```text
Prompt text: Enter the new name:
```

Register the `dialog` handler before performing the action that triggers
it — if a dialog appears with no handler registered, Playwright
auto-dismisses it after a short delay and logs a warning, which usually
isn't the behavior your test wants.

## Exercise

1. On `https://www.saucedemo.com/`, submit the login form with an empty
   password and assert the exact validation error shown
   (`[data-test='error']`).
2. Log in successfully, add three items to the cart, go to checkout, and
   fill in the "Checkout: Your Information" form (first name, last name,
   zip code) using `fill()` on each `get_by_placeholder(...)` locator.
3. Submit that form with the zip code field left empty and assert the
   error message shown, then confirm via `to_have_url()` that you're still
   on the checkout information step.
4. Complete the form correctly this time, click "Continue," and assert the
   order summary total using `to_have_text()` or `to_contain_text()` on the
   `.summary_total_label` element.
5. Find (or build locally, if you prefer) a page with a custom
   `role="combobox"` dropdown, and write a test driving it with
   `get_by_role("combobox")` + `get_by_role("option")` rather than
   `select_option()` — confirming you understand when each applies.
