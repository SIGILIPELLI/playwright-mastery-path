# 04 · Locators

## What a locator is

A `Locator` is a description of how to find an element — not a reference to
a specific DOM node captured at one moment in time. This distinction matters:
a locator is re-evaluated against the live page every time you act on it, so
if an element re-renders (React re-mounting a component, for instance) your
locator still finds the current version of it. This is a deliberate design
difference from Selenium's `WebElement`, which is a snapshot reference that
goes stale if the underlying DOM node is replaced.

```python
login_button = page.locator("#login-button")   # describes, doesn't fetch yet
login_button.click()                           # resolved and acted on now
```

```text
# no output — clicking succeeds as long as an element matching
# #login-button exists and is actionable at click time
```

## Locator priority: prefer what a user perceives

Playwright's own documentation states a clear preference order, and it's
worth internalizing *why*, not just memorizing it: a test should find
elements the way a real user or assistive technology would — by role, label,
or visible text — not by implementation details like CSS classes or DOM
structure, which change on every refactor with zero effect on what the user
actually experiences.

### 1. `get_by_role` — the recommended default

```python
page.get_by_role("button", name="Sign in").click()
page.get_by_role("link", name="Forgot password?").click()
page.get_by_role("heading", name="Welcome back", level=1)
page.get_by_role("checkbox", name="Remember me").check()
page.get_by_role("textbox", name="Email").fill("user@example.com")
```

```text
# no output — each resolves to the accessible role Chrome/Firefox/
# WebKit compute for that element, matched against its accessible name
```

This matches the ARIA role and accessible name the browser itself computes
for an element — the same information a screen reader uses. It works whether
the element is a `<button>`, a `<div role="button">`, or an `<input
type="submit">`, because it's testing the accessibility tree, not the tag.

### 2. `get_by_label` / `get_by_placeholder` — form fields

```python
page.get_by_label("Email address").fill("user@example.com")
page.get_by_label("Password").fill("hunter2")
page.get_by_placeholder("Search products...").fill("wireless mouse")
```

```text
# no output — get_by_label matches an <input> associated with a
# <label for="..."> or wrapping <label>, by the label's visible text
```

### 3. `get_by_text` — visible text content

```python
page.get_by_text("No results found")
page.get_by_text("Add to cart", exact=True)
```

```text
# no output — exact=True requires a full string match rather
# than substring containment, avoiding accidental matches on
# longer strings that merely contain "Add to cart"
```

Good for asserting on messages, headings, or content that isn't
interactive — less ideal for buttons/links where `get_by_role` gives you
richer matching (and doubles as a check that the element is exposed
correctly to assistive technology).

### 4. `get_by_test_id` — an explicit escape hatch

```python
page.get_by_test_id("checkout-submit-button").click()
```

```text
# no output — matches an element with data-testid="checkout-submit-button"
```

When a page genuinely has no stable accessible role, label, or text (a
purely decorative interactive icon, dynamically generated content with no
fixed copy), a `data-testid` attribute added specifically for tests is a
legitimate, explicit contract between the frontend and the test suite — far
more stable than a CSS class chosen for styling, which can change for
reasons that have nothing to do with test coverage.

### 5. CSS and XPath — last resort

```python
page.locator("#login-button")
page.locator(".product-card >> nth=2")
page.locator("//div[@class='card']//button")
```

```text
# no output — all three work, but none convey what the element
# means to a user, and all three break silently on a styling refactor
```

Playwright fully supports CSS and XPath selectors, and sometimes they're
genuinely the pragmatic choice on a legacy page you don't control. Reach for
them last, and prefer CSS over XPath when you do — XPath is more powerful
but harder to read and slower to evaluate in most engines.

## Chaining and filtering

```python
product_card = page.locator(".product-card").filter(has_text="Wireless Mouse")
product_card.get_by_role("button", name="Add to cart").click()

row = page.get_by_role("row").filter(has=page.get_by_text("Invoice #1042"))
row.get_by_role("button", name="Download").click()
```

```text
# no output — filter() narrows a locator that would otherwise
# match multiple elements down to the one matching the extra condition
```

`filter(has_text=...)` narrows by the element's own text; `filter(has=...)`
narrows by whether a *child* matches another locator. This composes far
better than a single long CSS selector for anything resembling "the row
containing X" or "the card titled Y."

## Locators are lists — `.first`, `.nth()`, `.count()`

```python
cards = page.locator(".product-card")
print("Total cards:", cards.count())
cards.first.click()
cards.nth(2).click()
for i in range(cards.count()):
    print(cards.nth(i).text_content())
```

```text
Total cards: 20
Wireless Mouse - $24.99
```

A `Locator` can match zero, one, or many elements; most action methods
(`.click()`, `.fill()`) require it to resolve to exactly one at the moment
of the action and raise a clear error if it matches more than one —
protecting you from silently clicking "whichever one happened to be first"
by accident. Use `.first`, `.last`, or `.nth(i)` when you deliberately mean
one of several matches.

## Scoping locators to a container

```python
navbar = page.locator("nav")
navbar.get_by_role("link", name="Pricing").click()
```

```text
# no output — resolves the link only within <nav>, so an
# identically-named "Pricing" link in the footer is never a candidate
```

Calling `.locator()` or `.get_by_role()` on an existing locator (instead of
on `page`) scopes the search to inside that element — the single most
effective technique for disambiguating a page with repeated text or
structure, like a navbar and footer that both link to "Pricing."

## Exercise

Using `https://books.toscrape.com/`:

1. Use `get_by_role("link", ...)` to click into a category from the sidebar
   (e.g. "Travel"), and confirm via `page.url` that navigation happened.
2. On the category page, use `page.locator(".product_pod")` to get all
   product cards, print `.count()`, then loop and print each product's title
   via `.locator("h3 a").get_attribute("title")` (the full title is in the
   `title` attribute because the visible text is truncated with CSS).
3. Use `.filter(has_text=...)` to find the specific product card whose title
   contains a word you choose (pick one you saw printed in step 2), and print
   its price via `.locator(".price_color")`.
4. Scope a locator to `.locator(".product_pod").first` and, from that scoped
   locator, get the "Add to basket" link's `href` attribute — confirming
   scoping avoids ambiguity when many cards share the same button text.
5. Deliberately write a `page.locator(".product_pod")` action call (like
   `.click()`) without `.first` or `.nth()` when more than one match exists,
   and read the resulting Playwright error message closely — it names the
   ambiguity and shows you how many elements it found.
