# 03 · Multiple Tabs, Frames & Popups

## Popups: a new page, not a new browser

Clicking a link with `target="_blank"`, or code calling `window.open()`,
opens a new tab. In Playwright terms that's a new `Page` object inside the
same `BrowserContext`. You catch it with the context's `expect_page`
context manager, which waits for the popup event around the action that
triggers it.

```python
from playwright.sync_api import Page

def test_opens_pricing_in_new_tab(page: Page):
    page.goto("https://example.com")
    with page.context.expect_page() as popup_info:
        page.get_by_role("link", name="View pricing (opens in new tab)").click()
    popup = popup_info.value
    popup.wait_for_load_state()
    assert "pricing" in popup.url
    popup.close()
```

```text
1 passed in 1.31s
```

The `with ... as popup_info:` block is essential — `expect_page()` starts
listening for a new page *before* the click happens, so it can't miss the
event racing against the click. Calling `context.pages[-1]` after the fact
would work sometimes and flake other times, depending on how fast the new
tab opens relative to your next line of code.

## Working with the popup

Once you have the popup `Page`, it behaves exactly like any other page —
its own locators, its own navigation, its own lifecycle.

```python
with page.context.expect_page() as popup_info:
    page.get_by_role("button", name="Share").click()
share_popup = popup_info.value
share_popup.get_by_role("textbox", name="Recipient email").fill("a@b.com")
share_popup.get_by_role("button", name="Send").click()
share_popup.wait_for_event("close")
```

```text
# no output — the original `page` is untouched; all three lines
# after popup_info.value act on the second, independent Page object
```

## Iframes: a different document, same page

An `<iframe>` embeds a separate HTML document. Elements inside it are
invisible to `page.locator()` directly — you must first get a
`FrameLocator` for the iframe, then locate within that.

```python
from playwright.sync_api import Page, expect

def test_payment_iframe(page: Page):
    page.goto("https://example.com/checkout")
    payment_frame = page.frame_locator("iframe[title='Payment form']")
    payment_frame.get_by_label("Card number").fill("4242 4242 4242 4242")
    payment_frame.get_by_label("Expiry").fill("12/29")
    payment_frame.get_by_label("CVC").fill("123")
    page.get_by_role("button", name="Pay now").click()
    expect(page.get_by_text("Payment successful")).to_be_visible()
```

```text
1 passed in 2.04s
```

This pattern is extremely common for payment providers (Stripe, Braintree)
that deliberately isolate card fields in a cross-origin iframe for PCI
compliance — your test code can't reach into it any other way, and
`frame_locator` is the only supported approach (older `page.frame(...)`
APIs that return a `Frame` object still exist but `frame_locator` is
preferred since it re-resolves the frame automatically, same as any other
locator, instead of grabbing a snapshot reference).

## Nested iframes

```python
outer = page.frame_locator("#outer-frame")
inner = outer.frame_locator("#inner-frame")
inner.get_by_role("button", name="Confirm").click()
```

```text
# no output — chaining frame_locator() drills through nested
# iframes exactly like chaining .locator() drills through elements
```

## Multiple regular tabs opened deliberately

Sometimes a test needs two tabs open at once on purpose — e.g. verifying
that an action in one tab (marking a notification read) reflects in
another tab open to the same account.

```python
def test_notification_syncs_across_tabs(context, page: Page):
    page.goto("https://example.com/inbox")
    second_page = context.new_page()
    second_page.goto("https://example.com/inbox")

    page.get_by_role("button", name="Mark all read").click()
    second_page.reload()
    from playwright.sync_api import expect
    expect(second_page.get_by_text("0 unread")).to_be_visible()
    second_page.close()
```

```text
1 passed in 1.77s
```

Both pages share the same `BrowserContext`, so cookies and localStorage
are shared between them automatically — this is what makes the
cross-tab-sync scenario realistic, mirroring how a real user's two browser
tabs behave.

## Switching context: `context.pages`

```python
print(len(page.context.pages))     # 2, after opening second_page
for p in page.context.pages:
    print(p.url)
```

```text
2
https://example.com/inbox
https://example.com/inbox
```

`context.pages` gives you every open page/tab in that context if you ever
lose track of a reference — useful when a flow opens several popups in
sequence and you need the *last* one, or need to iterate and find the one
matching a URL pattern.

## Exercise

Using `https://the-internet.herokuapp.com/windows` (opens a new tab) and
`https://the-internet.herokuapp.com/iframe` (a WYSIWYG editor in an
iframe):

1. On the windows page, click "Click Here" inside an `expect_page()` block,
   capture the popup, and assert its text contains "New Window" via
   `expect(popup.locator("h3")).to_have_text("New Window")`.
2. On the iframe page, use `page.frame_locator("#mce_0_ifr")` to get the
   editor body (`.locator("body#tinymce")`), clear it, and type a sentence
   with `.fill(...)`; assert the content with `.inner_text()`.
3. Open the iframe page in two separate `page`s from the same `context`,
   type different text in each, and confirm each frame body only reflects
   its own page's edit (frames are per-page, not shared across tabs).
4. Deliberately call `page.locator("body#tinymce")` (no `frame_locator`)
   and observe the timeout error — note in your own words why Playwright
   can't find an element that genuinely exists in the DOM tree the browser
   renders, connecting it back to "different document."
