# 03 · API Testing with Request Context

Not every check needs a browser. Playwright ships a standalone HTTP client —
`APIRequestContext` — that shares its cookie jar with a `BrowserContext`,
which makes it uniquely useful for seeding data, asserting on backend
responses directly, and mixing API calls into UI tests without spinning up
`requests` or `httpx` as a separate dependency.

## A pure API test, no browser at all

```python
import pytest
from playwright.sync_api import APIRequestContext, sync_playwright

@pytest.fixture(scope="session")
def api_context(playwright):
    context = playwright.request.new_context(
        base_url="https://api.example.com",
        extra_http_headers={"Authorization": "Bearer test-token"},
    )
    yield context
    context.dispose()

def test_get_user(api_context: APIRequestContext):
    response = api_context.get("/users/42")
    assert response.ok
    assert response.status == 200
    body = response.json()
    assert body["id"] == 42
    assert body["email"].endswith("@example.com")
```

```text
# playwright.request.new_context() creates an APIRequestContext
# independent of any browser — no Chromium/Firefox process is
# launched, so these tests run much faster than UI tests and
# are a good fit for backend contract checks
```

## POST, PUT, DELETE with JSON bodies

```python
def test_create_and_delete_todo(api_context):
    create = api_context.post("/todos", data={"title": "Buy milk", "done": False})
    assert create.status == 201
    todo_id = create.json()["id"]

    update = api_context.put(f"/todos/{todo_id}", data={"done": True})
    assert update.ok
    assert update.json()["done"] is True

    delete = api_context.delete(f"/todos/{todo_id}")
    assert delete.status == 204

    confirm = api_context.get(f"/todos/{todo_id}")
    assert confirm.status == 404
```

```text
# `data=` is JSON-encoded automatically when it's a dict; use
# `data=raw_bytes` or `multipart=` for other body types. The
# request/response cycle here never touches a browser, so this
# is a fast, reliable way to verify backend behavior in isolation
# from any frontend rendering bugs
```

## Sharing auth between API calls and a real browser context

Because `APIRequestContext` and `BrowserContext` can share cookies, you can
log in via the API and hand the resulting session to a real page — skipping
a UI login entirely, even faster than the storage-state approach from the
previous module:

```python
def test_dashboard_after_api_login(playwright, browser):
    api_context = playwright.request.new_context(base_url="https://example.com")
    login = api_context.post("/api/login", data={"email": "qa@example.com", "password": "secret"})
    assert login.ok

    storage_state = api_context.storage_state()
    browser_context = browser.new_context(storage_state=storage_state)
    page = browser_context.new_page()
    page.goto("/dashboard")
    expect(page.get_by_text("Welcome")).to_be_visible()

    browser_context.close()
    api_context.dispose()
```

```text
# api_context.storage_state() returns the same cookie/origin
# shape a BrowserContext produces, so it can be fed directly
# into new_context(storage_state=...) — logging in over HTTP
# is typically far faster than filling and submitting a form
```

## Mixing API setup into a UI test

The most common real-world pattern: use the API to get the app into a
specific state instantly, then use the browser only to test the UI behavior
that actually needs rendering.

```python
def test_edit_existing_order(page, api_context):
    order = api_context.post("/orders", data={"item": "Widget", "qty": 3}).json()

    page.goto(f"/orders/{order['id']}/edit")
    page.get_by_label("Quantity").fill("5")
    page.get_by_role("button", name="Save").click()

    expect(page.get_by_text("Order updated")).to_be_visible()

    confirm = api_context.get(f"/orders/{order['id']}")
    assert confirm.json()["qty"] == 5
```

```text
# the order is created via a single fast API call instead of
# clicking through a multi-step "create order" UI flow just to
# get to the screen under test, and the final assertion
# double-checks the backend directly rather than trusting the
# UI's own success message alone
```

## Asserting on response headers and timing

```python
def test_api_response_has_cache_headers(api_context):
    response = api_context.get("/products/1")
    assert response.headers.get("cache-control") == "max-age=300"
    assert "etag" in response.headers
```

## Full worked example: contract test for a paginated endpoint

```python
# tests/test_api_pagination.py
import pytest

def test_products_pagination_contract(api_context):
    page1 = api_context.get("/products?page=1&limit=10").json()
    assert len(page1["items"]) == 10
    assert page1["page"] == 1
    assert page1["has_next"] is True

    page2 = api_context.get("/products?page=2&limit=10").json()
    ids_page1 = {item["id"] for item in page1["items"]}
    ids_page2 = {item["id"] for item in page2["items"]}
    assert ids_page1.isdisjoint(ids_page2)  # no duplicate items across pages
```

## Exercise

1. Write a fixture that creates an `APIRequestContext` pointed at a real or
   mock API and use it to `GET` a resource, asserting on both `status` and
   the parsed JSON body.
2. Write a test that `POST`s a resource, then `PUT`s an update, then
   `DELETE`s it, asserting the expected status code at each step.
3. Log in via an API call, extract `storage_state()`, and use it to open an
   authenticated page in a real browser context — confirm no login form was
   ever rendered.
4. Rewrite one existing UI test so it uses the API to set up preconditions
   (e.g. creating a record) instead of clicking through the UI to create it,
   and note in a comment how much of the test's runtime that setup used to
   take.
