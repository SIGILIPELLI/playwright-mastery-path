# 09 · Test Data Seeding via API

Clicking through the UI to create the preconditions for a test ("create a
customer, add three orders, mark one refunded") is slow and makes the test
fragile to unrelated UI changes. Seed that state directly through the
backend API instead, and reserve the UI for the behavior actually under
test.

## A seeding fixture

```python
# conftest.py
import pytest
import uuid

@pytest.fixture
def seeded_customer(api_context):
    email = f"test-{uuid.uuid4().hex[:8]}@example.com"
    customer = api_context.post("/api/customers", data={
        "email": email,
        "name": "Test Customer",
    }).json()
    yield customer
    api_context.delete(f"/api/customers/{customer['id']}")  # cleanup
```

```python
def test_customer_profile_page(page, seeded_customer):
    page.goto(f"/customers/{seeded_customer['id']}")
    expect(page.get_by_text(seeded_customer["email"])).to_be_visible()
```

```text
# the fixture both creates the precondition data AND tears it
# down afterward (the code after `yield` always runs, even if
# the test fails) — so tests stay independent and don't
# accumulate junk records across runs
```

## Building a fixture factory for related records

```python
@pytest.fixture
def make_order(api_context, seeded_customer):
    created_ids = []
    def _make(item: str, qty: int = 1, status: str = "pending"):
        order = api_context.post("/api/orders", data={
            "customer_id": seeded_customer["id"],
            "item": item,
            "qty": qty,
            "status": status,
        }).json()
        created_ids.append(order["id"])
        return order
    yield _make
    for order_id in created_ids:
        api_context.delete(f"/api/orders/{order_id}")
```

```python
def test_order_history_shows_all_orders(page, seeded_customer, make_order):
    make_order("Widget", qty=2)
    make_order("Gadget", qty=1, status="shipped")

    page.goto(f"/customers/{seeded_customer['id']}/orders")
    expect(page.get_by_role("row")).to_have_count(3)  # 2 orders + header row
```

```text
# a factory fixture (a function the test calls, rather than a
# single fixed value) lets each test create exactly the records
# it needs, with whatever combination of fields matters for
# that specific scenario — while cleanup still happens
# automatically in one place
```

## Seeding via direct database access (when there's no API for it)

Some states (a user whose email is unverified after N days, a soft-deleted
record) have no API endpoint by design. A direct DB connection, scoped
tightly to test setup only, covers that gap:

```python
import psycopg
import pytest

@pytest.fixture
def db_connection():
    conn = psycopg.connect("postgresql://test_user:test_pass@localhost/app_test")
    yield conn
    conn.close()

@pytest.fixture
def unverified_user(db_connection, api_context):
    email = f"unverified-{uuid.uuid4().hex[:8]}@example.com"
    user = api_context.post("/api/users", data={"email": email, "password": "Test1234!"}).json()
    with db_connection.cursor() as cur:
        cur.execute("UPDATE users SET email_verified_at = NULL WHERE id = %s", (user["id"],))
        db_connection.commit()
    yield user
```

```text
# this must only ever point at a dedicated test database, never
# a shared staging or production instance — direct writes bypass
# every application-level validation and audit trail the API
# would normally enforce
```

## Factory pattern with realistic fake data

```python
from faker import Faker
fake = Faker()

@pytest.fixture
def make_customer(api_context):
    created = []
    def _make(**overrides):
        payload = {
            "email": fake.unique.email(),
            "name": fake.name(),
            "phone": fake.phone_number(),
            **overrides,
        }
        customer = api_context.post("/api/customers", data=payload).json()
        created.append(customer["id"])
        return customer
    yield _make
    for cid in created:
        api_context.delete(f"/api/customers/{cid}")
```

```text
# `Faker` generates realistic-looking data by default, and
# **overrides lets a specific test pin down only the field it
# actually cares about (e.g. make_customer(email="specific@x.com"))
# while everything else stays randomized and non-colliding
```

## Resetting to a known state instead of accumulating fixtures

For read-heavy tests against a large dataset (reports, search, filters),
seeding one record per test doesn't scale — reset to a fixed snapshot
instead:

```python
@pytest.fixture(scope="session", autouse=True)
def reset_test_database(api_context):
    api_context.post("/internal/test-only/reset-fixtures")
    yield
```

```text
# a backend endpoint that ONLY exists in the test environment
# (never deployed to production) restores a known fixture
# dataset once per test session — combined with per-test
# fixtures above for anything a specific test needs beyond
# the shared baseline
```

## Exercise

1. Write a fixture that creates a record via the API, yields it to the
   test, and deletes it afterward — confirm via a follow-up API call that
   the record is really gone once the test finishes.
2. Turn that fixture into a factory (a function the test calls, potentially
   multiple times) so a single test can create several related records with
   different field values.
3. Write a test using `Faker` (or hand-written realistic values) to seed a
   customer with a randomized but valid email, and assert the profile page
   shows it correctly.
4. Identify one piece of app state your API can't create directly (an
   expired token, a soft-deleted row) and write a fixture that reaches the
   database directly to set it up — with a comment explaining why this must
   never point at a shared or production database.
