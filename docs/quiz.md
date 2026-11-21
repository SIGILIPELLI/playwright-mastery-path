---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Playwright Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```python\n# page content:\n# <ul><li>Pen<\/li><li>Book<\/li><\/ul>\nloc = page.locator(\"li\")\npage.evaluate(\"document.querySelector('ul').insertAdjacentHTML('beforeend', '<li>Bag<\/li>')\")\nprint(loc.count(), loc.last.text_content())\n```\n",
    "options": [
      "2 Book",
      "3 Bag",
      "3 Book",
      "Error"
    ],
    "correctIndex": 1,
    "explanation": "A Locator is lazy: it queries the DOM each time you use it. Verified by running it with Playwright for Python."
  },
  {
    "question": "Why are locators lazy descriptions and not element handles?",
    "options": [
      "They re-resolve on every use so they keep working when the DOM re-renders",
      "Law",
      "Memory",
      "Speed"
    ],
    "correctIndex": 0,
    "explanation": "Resilience to change."
  },
  {
    "question": "What does this print?\n```python\n# page content:\n# <button>Save<\/button><button>Save as<\/button>\ntry:\n    page.get_by_role(\"button\", name=\"Save\").click()\nexcept Exception as e:\n    print(type(e).__name__)\nprint(page.get_by_role(\"button\", name=\"Save\", exact=True).count())\n```\n",
    "options": [
      "None\n1",
      "Error\n1",
      "Error\n2",
      "TimeoutError\n1"
    ],
    "correctIndex": 1,
    "explanation": "Strict mode violation: name matches substrings unless exact=True. Verified by running it with Playwright for Python."
  },
  {
    "question": "Which assertion style waits for async updates?",
    "options": [
      "expect(locator).to_have_text(...) (web-first, auto-retrying)",
      "time.sleep then assert",
      "print",
      "assert locator.text_content() == ..."
    ],
    "correctIndex": 0,
    "explanation": "Retries until timeout."
  },
  {
    "question": "What does this print?\n```python\n# page content:\n# <div id=\"r\">0<\/div><script>let n=0;const t=setInterval(()=>{document.getElementById(\"r\").textContent=++n;if(n==3)clearInterval(t)},50)<\/script>\nexpect(page.locator(\"#r\")).to_have_text(\"3\")\nprint(page.text_content(\"#r\"))\n```\n",
    "options": [
      "3",
      "1",
      "Error",
      "0"
    ],
    "correctIndex": 0,
    "explanation": "Assertions retry until the text matches. Verified by running it with Playwright for Python."
  },
  {
    "question": "Why is time.sleep(5) in a test a smell?",
    "options": [
      "Not allowed",
      "It's slow when the app is fast and still flaky when it is slow; wait for a condition",
      "Style",
      "Law"
    ],
    "correctIndex": 1,
    "explanation": "Auto-wait and expect."
  },
  {
    "question": "A test passes alone but fails in parallel. Likely cause?",
    "options": [
      "Python",
      "Network",
      "Browser",
      "Shared state (accounts, data, files) between tests"
    ],
    "correctIndex": 3,
    "explanation": "Make tests independent."
  },
  {
    "question": "What is the best use of storage_state?",
    "options": [
      "Speed pages",
      "Record video",
      "Mock API",
      "Log in once in setup and reuse cookies/localStorage for other tests"
    ],
    "correctIndex": 3,
    "explanation": "Faster, fewer login flakes."
  },
  {
    "question": "How do you capture a popup opened by a click?",
    "options": [
      "wait",
      "sleep",
      "with page.context.expect_page() (or page.expect_popup()) around the click",
      "pages[0]"
    ],
    "correctIndex": 2,
    "explanation": "Event-based."
  },
  {
    "question": "How should network mocks be used?",
    "options": [
      "Only prod",
      "Never",
      "To make third-party/error scenarios deterministic, keeping some real end-to-end coverage",
      "Everywhere"
    ],
    "correctIndex": 2,
    "explanation": "Balance."
  },
  {
    "question": "What does a trace help with in CI?",
    "options": [
      "Cost",
      "Speed",
      "Mocks",
      "Replaying a failed run step by step with DOM, network and console"
    ],
    "correctIndex": 3,
    "explanation": "Upload as an artifact."
  },
  {
    "question": "Screenshot comparisons differ between laptop and CI. What helps?",
    "options": [
      "Bigger threshold",
      "Skip",
      "Ignore",
      "Generate and compare baselines in the same containerized environment"
    ],
    "correctIndex": 3,
    "explanation": "Fonts and rendering."
  },
  {
    "question": "A flaky test is retried and passes. What should you do?",
    "options": [
      "Delete",
      "Investigate the root cause; retries only reveal and tolerate flakiness",
      "Add retries",
      "Nothing"
    ],
    "correctIndex": 1,
    "explanation": "Track flaky tests."
  },
  {
    "question": "Which locator is usually most resilient?",
    "options": [
      "get_by_role('button', name='Pay')",
      "nth-child chain",
      "//div[3]/div/button",
      ".btn-primary.x"
    ],
    "correctIndex": 0,
    "explanation": "User-facing."
  },
  {
    "question": "What does the Page Object Model buy you?",
    "options": [
      "Fewer browsers",
      "No fixtures",
      "One place to change when the UI changes and tests that read like user stories",
      "Speed"
    ],
    "correctIndex": 2,
    "explanation": "Maintainability."
  },
  {
    "question": "What does a fresh browser context per test provide?",
    "options": [
      "Faster page",
      "Shared login",
      "Isolated cookies and storage at low cost",
      "New browser"
    ],
    "correctIndex": 2,
    "explanation": "Test independence."
  },
  {
    "question": "How can one suite run on Chromium, Firefox and WebKit?",
    "options": [
      "Edit tests",
      "pytest --browser chromium --browser firefox --browser webkit",
      "Three codebases",
      "Selenium grid"
    ],
    "correctIndex": 1,
    "explanation": "One API."
  },
  {
    "question": "What should fail a pull request pipeline?",
    "options": [
      "Warnings",
      "Slow tests",
      "Nothing",
      "Failing tests, with traces/screenshots uploaded for diagnosis"
    ],
    "correctIndex": 3,
    "explanation": "Quality gate."
  },
  {
    "question": "Automated accessibility checks (axe) are best described as?",
    "options": [
      "Optional law",
      "Useless",
      "Complete",
      "A useful baseline that catches common issues but doesn't replace manual testing"
    ],
    "correctIndex": 3,
    "explanation": "Limits."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A page object and isolated login tests

Implement LoginPage (locators for the Email and Password fields, the Sign in button and the alert; a login() method that returns self), then run the three login cases in fresh browser contexts. A deliberately wrong expectation shows how a failure is reported.

**Starter code**

```python
from playwright.sync_api import sync_playwright, expect

LOGIN_HTML = '<form id="f"><label>Email <input id="email"></label><label>Password <input id="pw" type="password"></label>\n<button type="submit">Sign in</button><p id="msg" role="alert"></p></form>\n<script>document.getElementById("f").addEventListener("submit",e=>{e.preventDefault();\nconst ok=document.getElementById("email").value==="ada@example.com"&&document.getElementById("pw").value==="s3cret";\ndocument.getElementById("msg").textContent=ok?"Welcome, Ada":"Invalid credentials";});</script>'

class LoginPage:
    def __init__(self, page):
        self.page = page
        # create locators for: Email field, Password field, the "Sign in" button and the alert message
        # (use get_by_label, get_by_role) -- locators are lazy, so it is fine to build them here

    def login(self, email, password):
        # fill both fields, click the button and return self so calls can be chained
        return self

def check(name, fn):
    try:
        fn()
        print("PASS", name)
    except AssertionError:
        print("FAIL", name)

with sync_playwright() as p:
    browser = p.chromium.launch()
    for email, pw, expected in [("ada@example.com", "s3cret", "Welcome, Ada"),
                                ("ada@example.com", "wrong", "Invalid credentials"),
                                ("", "", "Invalid credentials")]:
        context = browser.new_context()            # fresh, isolated state per test
        page = context.new_page()
        page.set_content(LOGIN_HTML)
        page.set_default_timeout(1000)
        expect.set_options(timeout=1000)
        lp = LoginPage(page).login(email, pw)
        check(f"login({email or 'empty'!r}, {pw or 'empty'!r})", lambda: expect(lp.message).to_have_text(expected))
        context.close()
    # a deliberately wrong expectation shows how a failing check is reported
    context = browser.new_context(); page = context.new_page(); page.set_content(LOGIN_HTML)
    expect.set_options(timeout=500)
    check("expects wrong text", lambda: expect(LoginPage(page).login("ada@example.com", "s3cret").message).to_have_text("Goodbye"))
    browser.close()
```

**Expected output**

```text
PASS login('ada@example.com', 's3cret')
PASS login('ada@example.com', 'wrong')
PASS login('empty', 'empty')
FAIL expects wrong text
```

<details><summary>Show solution and explanation</summary>

<pre><code>from playwright.sync_api import sync_playwright, expect

LOGIN_HTML = &#x27;&lt;form id=&quot;f&quot;&gt;&lt;label&gt;Email &lt;input id=&quot;email&quot;&gt;&lt;/label&gt;&lt;label&gt;Password &lt;input id=&quot;pw&quot; type=&quot;password&quot;&gt;&lt;/label&gt;\n&lt;button type=&quot;submit&quot;&gt;Sign in&lt;/button&gt;&lt;p id=&quot;msg&quot; role=&quot;alert&quot;&gt;&lt;/p&gt;&lt;/form&gt;\n&lt;script&gt;document.getElementById(&quot;f&quot;).addEventListener(&quot;submit&quot;,e=&gt;{e.preventDefault();\nconst ok=document.getElementById(&quot;email&quot;).value===&quot;ada@example.com&quot;&amp;&amp;document.getElementById(&quot;pw&quot;).value===&quot;s3cret&quot;;\ndocument.getElementById(&quot;msg&quot;).textContent=ok?&quot;Welcome, Ada&quot;:&quot;Invalid credentials&quot;;});&lt;/script&gt;&#x27;

class LoginPage:
    def __init__(self, page):
        self.page = page
        self.email = page.get_by_label(&quot;Email&quot;)
        self.password = page.get_by_label(&quot;Password&quot;)
        self.submit = page.get_by_role(&quot;button&quot;, name=&quot;Sign in&quot;)
        self.message = page.get_by_role(&quot;alert&quot;)

    def login(self, email, password):
        self.email.fill(email)
        self.password.fill(password)
        self.submit.click()
        return self

def check(name, fn):
    try:
        fn()
        print(&quot;PASS&quot;, name)
    except AssertionError:
        print(&quot;FAIL&quot;, name)

with sync_playwright() as p:
    browser = p.chromium.launch()
    for email, pw, expected in [(&quot;ada@example.com&quot;, &quot;s3cret&quot;, &quot;Welcome, Ada&quot;),
                                (&quot;ada@example.com&quot;, &quot;wrong&quot;, &quot;Invalid credentials&quot;),
                                (&quot;&quot;, &quot;&quot;, &quot;Invalid credentials&quot;)]:
        context = browser.new_context()            # fresh, isolated state per test
        page = context.new_page()
        page.set_content(LOGIN_HTML)
        page.set_default_timeout(1000)
        expect.set_options(timeout=1000)
        lp = LoginPage(page).login(email, pw)
        check(f&quot;login({email or &#x27;empty&#x27;!r}, {pw or &#x27;empty&#x27;!r})&quot;, lambda: expect(lp.message).to_have_text(expected))
        context.close()
    # a deliberately wrong expectation shows how a failing check is reported
    context = browser.new_context(); page = context.new_page(); page.set_content(LOGIN_HTML)
    expect.set_options(timeout=500)
    check(&quot;expects wrong text&quot;, lambda: expect(LoginPage(page).login(&quot;ada@example.com&quot;, &quot;s3cret&quot;).message).to_have_text(&quot;Goodbye&quot;))
    browser.close()
</code></pre>

<p>The page object owns the locators and actions; tests only describe intent. Each case gets a new context for isolation, and expect() retries until the message appears. Verified by running the solution with Playwright for Python and Chromium.</p>
</details>

### Mock an API and read a table without sleeping

Implement run_case(browser, items): route the items endpoint to a canned JSON response, load the page, wait for #status to read "ready" with expect (no sleep), and read the table into dicts with read_table. Then report rows, the cheapest item, the total and the names.

**Starter code**

```python
from playwright.sync_api import sync_playwright, expect

PAGE = """<table id="t"><thead><tr><th>Item</th><th>Price</th></tr></thead><tbody id="b"></tbody></table><p id="status">loading</p>
<script>
setTimeout(() => fetch("http://shop.test/api/items").then(r => r.json()).then(items => {
  document.getElementById("b").innerHTML = items.map(i => `<tr><td>${i.name}</td><td>${i.price}</td></tr>`).join("");
  document.getElementById("status").textContent = "ready";
}), 200);
</script>"""

def read_table(page):
    # return a list of {"name": str, "price": float} dicts, one per row of #b tr
    return []

def run_case(browser, items):
    # new page; route http://shop.test/api/items to fulfill(json=items); set_content(PAGE);
    # wait with expect(...).to_have_text("ready") on #status (never sleep); read the table; close the page; return the data
    return []

with sync_playwright() as p:
    browser = p.chromium.launch()
    data = run_case(browser, [{"name": "Pen", "price": 2.5}, {"name": "Book", "price": 12}, {"name": "Bag", "price": 30.25}])
    print("rows:", len(data))
    print("cheapest:", min(data, key=lambda r: r["price"])["name"])
    print("total:", f"{sum(r['price'] for r in data):.2f}")
    print("names:", ", ".join(sorted(r["name"] for r in data)))
    empty = run_case(browser, [])
    print("empty rows:", len(empty))
    browser.close()
```

**Expected output**

```text
rows: 3
cheapest: Pen
total: 44.75
names: Bag, Book, Pen
empty rows: 0
```

<details><summary>Show solution and explanation</summary>

<pre><code>from playwright.sync_api import sync_playwright, expect

PAGE = &quot;&quot;&quot;&lt;table id=&quot;t&quot;&gt;&lt;thead&gt;&lt;tr&gt;&lt;th&gt;Item&lt;/th&gt;&lt;th&gt;Price&lt;/th&gt;&lt;/tr&gt;&lt;/thead&gt;&lt;tbody id=&quot;b&quot;&gt;&lt;/tbody&gt;&lt;/table&gt;&lt;p id=&quot;status&quot;&gt;loading&lt;/p&gt;
&lt;script&gt;
setTimeout(() =&gt; fetch(&quot;http://shop.test/api/items&quot;).then(r =&gt; r.json()).then(items =&gt; {
  document.getElementById(&quot;b&quot;).innerHTML = items.map(i =&gt; `&lt;tr&gt;&lt;td&gt;${i.name}&lt;/td&gt;&lt;td&gt;${i.price}&lt;/td&gt;&lt;/tr&gt;`).join(&quot;&quot;);
  document.getElementById(&quot;status&quot;).textContent = &quot;ready&quot;;
}), 200);
&lt;/script&gt;&quot;&quot;&quot;

def read_table(page):
    rows = page.locator(&quot;#b tr&quot;)
    return [{&quot;name&quot;: rows.nth(i).locator(&quot;td&quot;).nth(0).text_content(),
             &quot;price&quot;: float(rows.nth(i).locator(&quot;td&quot;).nth(1).text_content())}
            for i in range(rows.count())]

def run_case(browser, items):
    page = browser.new_page()
    page.set_default_timeout(3000)
    page.route(&quot;http://shop.test/api/items&quot;, lambda route: route.fulfill(json=items))
    page.set_content(PAGE)
    expect(page.locator(&quot;#status&quot;)).to_have_text(&quot;ready&quot;)     # waits for the async render, no sleep
    data = read_table(page)
    page.close()
    return data

with sync_playwright() as p:
    browser = p.chromium.launch()
    data = run_case(browser, [{&quot;name&quot;: &quot;Pen&quot;, &quot;price&quot;: 2.5}, {&quot;name&quot;: &quot;Book&quot;, &quot;price&quot;: 12}, {&quot;name&quot;: &quot;Bag&quot;, &quot;price&quot;: 30.25}])
    print(&quot;rows:&quot;, len(data))
    print(&quot;cheapest:&quot;, min(data, key=lambda r: r[&quot;price&quot;])[&quot;name&quot;])
    print(&quot;total:&quot;, f&quot;{sum(r[&#x27;price&#x27;] for r in data):.2f}&quot;)
    print(&quot;names:&quot;, &quot;, &quot;.join(sorted(r[&quot;name&quot;] for r in data)))
    empty = run_case(browser, [])
    print(&quot;empty rows:&quot;, len(empty))
    browser.close()
</code></pre>

<p>route.fulfill makes the data deterministic, expect waits for the asynchronous render, and locator.nth(i) reads the rows. Verified by running the solution with Playwright for Python and Chromium.</p>
</details>
