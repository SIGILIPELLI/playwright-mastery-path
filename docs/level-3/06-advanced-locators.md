# 06 · Advanced Locator Strategies

Level 1 covers `get_by_role`, `get_by_text`, and friends. Real apps often
need to combine locators, filter by descendant content, or fall back to
structural selectors when the markup gives you nothing semantic to hook
into — this module covers that toolbox.

## Chaining and scoping

```python
row = page.get_by_role("row", name="Widget Pro")
row.get_by_role("button", name="Delete").click()
```

```text
# get_by_role("button", name="Delete") alone would match every
# delete button on the page (a strict-mode violation); scoping
# it to the specific row first narrows the search to just that
# row's subtree
```

## `.filter()` — narrow by text or a nested locator

```python
# by visible text anywhere inside the element
page.get_by_role("listitem").filter(has_text="Out of stock").get_by_role("button", name="Notify me").click()

# by a nested locator matching, rather than plain text
rows = page.get_by_role("row").filter(has=page.get_by_role("button", name="Refund issued"))
expect(rows).to_have_count(2)

# by a nested locator NOT matching
active_rows = page.get_by_role("row").filter(has_not_text="Archived")
```

```text
# filter() doesn't change what the locator points at directly —
# it returns a new locator that only matches elements from the
# original set whose subtree satisfies the given condition,
# which composes cleanly with .first, .nth(), and to_have_count()
```

## `locator.and_()` and `locator.or_()`

```python
# matches elements satisfying both locators
save_or_disabled = page.get_by_role("button").and_(page.locator("[data-dirty='true']"))

# matches either locator — useful for "one of two possible labels"
close_button = page.get_by_role("button", name="Close").or_(page.get_by_role("button", name="Dismiss"))
close_button.click()
```

## CSS and XPath — the pragmatic fallback

Prefer role/text/label locators, but some legacy markup genuinely has no
accessible name. CSS is the next-best option; XPath should be a last resort
reserved for relationships CSS can't express (matching by text content
combined with ancestor traversal in one expression):

```python
# CSS: attribute + structural selector
page.locator("table.orders tbody tr:nth-child(3) td.status")

# CSS: combining with pseudo-classes Playwright extends
page.locator("button:visible:not([disabled])")

# XPath: find an ancestor from a text match — hard to express in CSS
page.locator("xpath=//td[text()='Overdue']/ancestor::tr")
```

```text
# `:visible` and `:enabled`/`:disabled` are Playwright's own CSS
# extensions layered on top of standard CSS — they work inside
# page.locator() but are not valid outside Playwright's engine
```

## `has_text` vs `get_by_text` — substring vs. exact

```python
# get_by_text does an exact match by default (whitespace-normalized)
page.get_by_text("Save")            # will NOT match "Save changes"
page.get_by_text("Save", exact=False)  # substring match, matches "Save changes"

# has_text on a locator/filter is always a substring match
page.get_by_role("button").filter(has_text="Save")  # matches "Save changes"
```

Getting this distinction wrong is the single most common cause of a locator
that "should obviously match" silently returning zero elements.

## `nth()` and why it's usually the wrong tool

```python
# fragile — relies on DOM order staying exactly the same
page.get_by_role("row").nth(2).get_by_role("button", name="Edit").click()

# better — identify the row by its actual content
page.get_by_role("row", name="SKU-4821").get_by_role("button", name="Edit").click()
```

`nth()` is appropriate when the elements are genuinely interchangeable (e.g.
"the third item in a carousel, whichever it is") — not as a workaround for a
locator that isn't specific enough.

## Custom test-id locators for markup with no semantics

```python
# playwright.config equivalent: set once, globally
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    p.selectors.set_test_id_attribute("data-qa")
```

```python
# now get_by_test_id reads from data-qa instead of the default data-testid
page.get_by_test_id("checkout-total").click()
```

```text
# changing the attribute name is a one-line, global config
# change — teams that already use a data-qa or data-cy
# convention (e.g. migrating from Cypress) don't need to touch
# their markup to adopt Playwright
```

## Full worked example: a locator toolbox for a data table

```python
def delete_row_by_sku(page, sku: str):
    row = page.get_by_role("row").filter(has_text=sku)
    expect(row).to_have_count(1)  # fail loudly if the SKU is ambiguous or missing
    row.get_by_role("button", name="Delete").click()
    page.get_by_role("button", name="Confirm delete").click()

def get_visible_enabled_actions(page):
    return page.locator("button.row-action:visible:not([disabled])")

def find_overdue_rows(page):
    return page.locator("xpath=//td[contains(text(),'Overdue')]/ancestor::tr")
```

## Exercise

1. Build a locator using `.filter(has_text=...)` that finds a specific row
   in a table by content, then acts on a button scoped inside that row.
2. Write one locator using `.filter(has=...)` (a nested-locator condition,
   not text) and explain in a comment the difference from `has_text`.
3. Find (or construct) a piece of markup with no accessible role or label,
   write a CSS locator for it, then write an equivalent XPath locator and
   note which relationship (if any) only XPath could express cleanly.
4. Change the test-id attribute globally with `set_test_id_attribute` and
   confirm `get_by_test_id` now reads from the new attribute name instead
   of `data-testid`.
