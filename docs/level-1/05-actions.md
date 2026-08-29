# 05 · Actions

Every action method below performs Playwright's **actionability checks**
first (Module 7 covers these in depth): the target element must be attached
to the DOM, visible, stable (not mid-animation), able to receive events (not
covered by another element), and — for most actions — enabled. If any check
keeps failing past the timeout, Playwright raises a clear error naming which
check failed, instead of silently clicking the wrong thing or a
disabled control.

## Clicking

```python
page.get_by_role("button", name="Sign in").click()
page.get_by_role("link", name="Terms").click(button="right")
page.get_by_role("button", name="Submit").click(modifiers=["Shift"])
page.get_by_role("button", name="Preview").dblclick()
```

```text
# no output on success — click() raises a TimeoutError naming the
# failed actionability check if the button never becomes clickable
```

`click()` defaults to a left click at the element's visible center.
`button="right"` and `modifiers=[...]` (`"Alt"`, `"Control"`, `"Meta"`,
`"Shift"`) cover context menus and modified clicks (open-in-new-tab, for
example).

## Typing text

```python
page.get_by_label("Email").fill("user@example.com")
page.get_by_label("Search").type("wireless mouse", delay=100)
page.get_by_label("Comments").press_sequentially("Looks good!", delay=50)
```

```text
# no output — fill() sets the value in one operation; type() and
# press_sequentially() dispatch individual keystroke events
```

`fill()` clears the field and sets its value directly — it's the right
default for almost every form field, because it's fast and doesn't depend on
timing. Reach for `press_sequentially()` (the modern name for what used to be
called `type()`, which is now deprecated) only when the page has JavaScript
that reacts to individual keystrokes — an autocomplete dropdown, a live
character counter, or masked input formatting — since those only fire on
real per-key events.

## Keyboard and individual key presses

```python
page.get_by_label("Search").press("Enter")
page.keyboard.press("Control+A")
page.keyboard.press("Escape")
```

```text
# no output — press() on a locator focuses that element first,
# then dispatches the named key; page.keyboard acts wherever
# focus currently is
```

## Checkboxes, radio buttons, and selects

```python
page.get_by_label("Remember me").check()
page.get_by_label("Send me promotional emails").uncheck()
assert page.get_by_label("Remember me").is_checked()

page.get_by_label("Country").select_option("India")
page.get_by_label("Country").select_option(value="IN")
page.get_by_label("Toppings").select_option(["Cheese", "Olives"])
```

```text
# no output — select_option accepts a visible label string,
# value="..." for the underlying <option value>, or a list
# for a multi-select
```

`check()`/`uncheck()` are idempotent — calling `check()` on an already-checked
box is a no-op rather than toggling it, which avoids a whole class of bug
where `.click()` on a checkbox flips state unpredictably depending on
whatever state a previous test left it in.

## Hovering and drag-and-drop

```python
page.get_by_text("Account").hover()
page.get_by_role("menuitem", name="Settings").click()

source = page.locator("#item-3")
target = page.locator("#dropzone")
source.drag_to(target)
```

```text
# no output — hover() is essential for menus that only render
# their items on :hover; drag_to() performs mouse-down, move,
# mouse-up in sequence between the two elements' centers
```

## Uploading files

```python
page.get_by_label("Upload resume").set_input_files("resume.pdf")
page.get_by_label("Attach files").set_input_files(
    ["photo1.jpg", "photo2.jpg"]
)
page.get_by_label("Attach files").set_input_files([])  # clear selection
```

```text
# no output — works even on a visually hidden <input type="file">,
# because it sets the file list directly rather than simulating
# an OS file picker dialog
```

Module 5 of Level 2 covers upload/download in more depth, including
handling a custom drag-and-drop upload widget and intercepting the
`download` event.

## Scrolling into view

```python
page.get_by_text("Contact us").scroll_into_view_if_needed()
```

```text
# no output — most action methods already auto-scroll the target
# into view; this exists for the rare case you need to force it
# ahead of a screenshot or a manual check
```

You rarely need this explicitly — `click()`, `fill()`, and friends already
scroll their target into view as part of their actionability checks. It's
useful before taking a screenshot of a specific element, or before a manual
`is_visible()` check where nothing is being "acted on" to trigger auto-scroll.

## Executing JavaScript directly (the deliberate escape hatch)

```python
title = page.evaluate("document.title")
page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
count = page.locator(".item").evaluate_all("els => els.length")
```

```text
# no output for the scrollTo call; title and count hold the
# returned JS values in Python
```

`evaluate()` runs arbitrary JavaScript inside the page and returns the
(JSON-serializable) result to Python. Useful for reading application state
that isn't exposed through the DOM, or for setup a real user action can't
easily trigger (seeding `localStorage` before a test, for example) — but it
bypasses actionability checks and real event dispatch entirely, so prefer a
real action method whenever one exists. A test that only ever manipulates
the page through `evaluate()` isn't really testing what a user experiences.

## Exercise

Using `https://books.toscrape.com/`:

1. Fill and submit the search box (top-right on some builds; if this
   particular demo site has no functional search, instead practice on
   `https://www.saucedemo.com/` — a public demo store built for test
   automation practice — logging in with username `standard_user` and
   password `secret_sauce` using `fill()` on both fields, then `.click()`
   on the login button).
2. On saucedemo, hover over the hamburger menu icon, then click "Reset App
   State" from the menu that appears — this exercises `hover()` plus a menu
   item click.
3. Check a product's "Add to cart" checkbox-styled button is not applicable
   here, so instead: click "Add to cart" on two different products, then
   assert the cart badge shows `"2"` via `text_content()`.
4. Use `select_option()` on the sort dropdown (`Name (A to Z)` /
   `Price (low to high)`, etc.) and confirm the first product's name changes
   after selecting a different sort order.
5. Use `page.keyboard.press("Escape")` after opening the hamburger menu to
   close it without clicking anywhere, and assert the menu is no longer
   visible.
