# Playwright Automation: `page.on()` vs. `page.route()`

While both methods use callbacks, their **initialization behavior is different**.

The easiest way to understand the difference is:

> **`page.on()` = Register a listener and wait for something to happen later.**
>
> **`page.route()` = Set up a network interception rule before the request happens.**

---

## Quick Reference Comparison

| Method | Purpose | What happens? | Requires `await`? | Simple reason |
|---|---|---|---:|---|
| **`page.on()`** | Listen to browser events such as console messages, errors, or dialogs | Registers an event listener in the Playwright/Node.js side | ❌ **No** | Listener registration is synchronous |
| **`page.route()`** | Intercept or mock network requests | Asynchronously registers a routing rule in Playwright | ✅ **Yes** | The routing setup must complete before continuing |
| **`page.goto()`** | Navigate to a URL | Performs browser navigation | ✅ **Yes** | Navigation is asynchronous |
| **`page.click()`** | Click an element | Performs an action in the browser | ✅ **Yes** | The browser action is asynchronous |

---

# 1. Understanding the Architecture

Think of your Playwright test as having two sides:

```text
┌──────────────────────────────┐
│     Your Test Script         │
│          Node.js             │
│                              │
│  test()                      │
│  page.on()                   │
│  page.route()                 │
│  page.goto()                  │
│  page.click()                 │
└──────────────┬───────────────┘
               │
               │ Playwright communication
               │
               ▼
┌──────────────────────────────┐
│          Browser             │
│    Chrome / Chromium         │
│                              │
│  Website                     │
│  DOM                         │
│  Network requests            │
└──────────────────────────────┘
```

The important point is that **not every Playwright method requires the same type of operation**.

Some methods simply register something in the test process.

Other methods need Playwright to perform an asynchronous operation involving the browser.

---

# 2. Why `page.on()` Does NOT Need `await`

`page.on()` is used to register an event listener.

For example:

```javascript
page.on('console', msg => {
  console.log(`Browser log: ${msg.text()}`);
});
```

You are essentially saying:

> **"If a console event happens later, run this function."**

The listener is registered immediately.

Conceptually:

```text
Your Test Script

page.on('console', callback)
        │
        ▼
Register callback
        │
        ▼
Continue immediately
        │
        │
        │   Later...
        ▼
Browser produces console event
        │
        ▼
Callback executes
```

So you don't need:

```javascript
await page.on(...); // ❌
```

Instead:

```javascript
page.on('console', callback); // ✅
```

### Simple analogy

Imagine you tell someone:

> "If the delivery person arrives, call me."

You don't wait for the delivery person to arrive.

You simply register the instruction and continue with your work.

That's essentially what an event listener does.

---

# 3. Why `page.route()` Uses `await`

`page.route()` is used to intercept network requests.

For example:

```javascript
await page.route('**/api/v1/fruits', async route => {
  await route.fulfill({
    json: [{ name: 'Apple' }]
  });
});
```

Here, you are telling Playwright:

> **"Whenever a request matching this URL occurs, intercept it and return my mock response."**

The important part is that the **routing setup itself is asynchronous**.

Therefore, you should wait for the operation to complete before starting something that could trigger the request.

```javascript
await page.route(...);

await page.goto('https://example.com');
```

The sequence is:

```text
1. Register route
       ↓
2. Wait for route setup to complete
       ↓
3. Navigate to website
       ↓
4. Website makes API request
       ↓
5. Playwright intercepts request
       ↓
6. Mock response is returned
```

---

# 4. Why `await` Matters Here

Think of `await` as:

> **"Don't move to the next line until this asynchronous operation is complete."**

So:

```javascript
await page.route(...);
```

means:

```text
Set up the route
       ↓
Wait until setup is complete
       ↓
Continue
```

Without `await`:

```javascript
page.route(...);
await page.goto(...);
```

you are starting the route setup but immediately moving to the next operation.

---

# 5. What Happens If You Omit `await`?

Consider:

```javascript
// ❌ No await
page.route('**/api/fruits', route => {
  route.fulfill({ json: [] });
});

await page.goto('https://example.com');
```

Conceptually, you now have two operations being started without the required sequencing:

```text
Your Test

Start route setup
       │
       ├───────────────┐
       │               │
       ▼               ▼
Continue          Route setup
to goto()         still completing
       │
       ▼
Website loads
       │
       ▼
API request happens
       │
       ▼
Does the route exist yet?
       │
       ├── YES → Mock response
       │
       └── NO  → Request may not be intercepted
```

This creates a potential **race condition**.

The exact internal implementation is handled by Playwright, but the important testing concept is:

> **You should not rely on the route being ready while its asynchronous setup is still in progress.**

---

# 6. Correct Version

Use:

```javascript
await page.route('**/api/v1/fruits', async route => {
  await route.fulfill({
    json: [{ name: 'Apple' }]
  });
});

await page.goto('https://example.com');
```

Now the intended order is clear:

```text
Route setup
     ↓
await
     ↓
Route ready
     ↓
page.goto()
     ↓
Website loads
     ↓
API request
     ↓
Route intercepts it
     ↓
Mock response
```

---

# 7. Important: `await` Inside the Route Callback

You may also see:

```javascript
await page.route('**/api/v1/fruits', async route => {
  await route.fulfill({
    json: [{ name: 'Apple' }]
  });
});
```

There are actually **two different `await`s** here.

### First `await`

```javascript
await page.route(...)
```

This waits for the **route registration** to complete.

### Second `await`

```javascript
await route.fulfill(...)
```

This waits for the **mock response operation** to complete.

So:

```javascript
await page.route('**/api/v1/fruits', async route => {
  await route.fulfill({
    json: [{ name: 'Apple' }]
  });
});
```

can be understood as:

```text
await page.route()
       │
       ▼
Wait for route registration
       │
       ▼
Route is ready
       │
       ▼
Later, API request occurs
       │
       ▼
Callback executes
       │
       ▼
await route.fulfill()
       │
       ▼
Wait for mock response operation
```

---

# 8. `page.on()` vs `page.route()` — The Key Difference

The most useful way to remember this is:

### `page.on()`

```javascript
page.on('console', callback);
```

Means:

> **"Listen for this event."**

It registers a listener.

You don't need to wait for the event to happen.

---

### `page.route()`

```javascript
await page.route('**/api/fruits', callback);
```

Means:

> **"Set up this network interception rule."**

The setup is asynchronous, so you wait for it to complete.

---

# 9. Real QA Example

Suppose your application displays a list of fruits.

Normally:

```text
UI
 │
 ▼
GET /api/fruits
 │
 ▼
Real Backend
 │
 ▼
Real response
 │
 ▼
UI displays fruits
```

During a test, you want to mock the API:

```text
UI
 │
 ▼
GET /api/fruits
 │
 ▼
Playwright route
 │
 ▼
Mock response
 │
 ▼
UI displays test data
```

Your test could be:

```javascript
test('should display mocked fruit', async ({ page }) => {

  await page.route('**/api/fruits', async route => {
    await route.fulfill({
      json: [
        { id: 1, name: 'Apple' }
      ]
    });
  });

  await page.goto('https://example.com');

  await expect(page.getByText('Apple')).toBeVisible();
});
```

The important thing is that the route is configured **before navigation**.

That way, when the application starts making its API calls, the interception rule is already in place.

---

# 10. A Better Mental Model

Don't think:

> "`page.on()` is synchronous and `page.route()` talks directly to Chrome over a WebSocket."

That is unnecessarily implementation-specific.

Instead, remember the practical Playwright rule:

```text
page.on()
   ↓
Register listener
   ↓
No await

page.route()
   ↓
Register asynchronous route
   ↓
await

page.goto()
   ↓
Asynchronous browser navigation
   ↓
await

page.click()
   ↓
Asynchronous browser action
   ↓
await
```

This mental model is easier to use in day-to-day automation and interviews.

---

# 11. Interview-Friendly Answer

If an interviewer asks:

> **"Why does `page.on()` not require await while `page.route()` does?"**

A good answer is:

> "`page.on()` is used to register an event listener, and the listener registration itself is synchronous, so it doesn't return a Promise that needs to be awaited. `page.route()`, on the other hand, performs asynchronous route registration, so we use `await` to ensure the interception rule is set up before the test continues and potentially triggers the request."

### Short version

> **`page.on()` = register a listener → no `await`.**
>
> **`page.route()` = asynchronously register a network route → use `await`.**

---

# 12. Final Cheat Sheet

```text
┌────────────────────────────────────────────┐
│              page.on()                     │
├────────────────────────────────────────────┤
│ Listen for an event                        │
│                                            │
│ page.on('console', callback);              │
│                                            │
│ No await                                   │
│                                            │
│ "Tell me when something happens."          │
└────────────────────────────────────────────┘


┌────────────────────────────────────────────┐
│             page.route()                   │
├────────────────────────────────────────────┤
│ Intercept/mock a network request           │
│                                            │
│ await page.route('**/api/**', callback);   │
│                                            │
│ Use await                                  │
│                                            │
│ "Set this interception up before          │
│  I continue."                              │
└────────────────────────────────────────────┘
```

## One Line to Remember

> **`page.on()` listens for something that may happen later; `page.route()` sets up something that needs to be ready before the request happens.**
