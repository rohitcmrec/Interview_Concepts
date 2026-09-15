# Playwright: `page.on()` vs `page.route()`

Let's strip away all the technical jargon. Let's look at **where the work actually happens**.

Your test is split into two separate programs:

1. **Your Script (Node.js)** – Where your test code lives.
2. **The Browser (Chrome)** – Where the actual website opens.

They talk to each other over a virtual wire.

---

## 1. Why `page.on()` Does NOT Need `await`

### Example

```javascript
page.on('console', msg => console.log(msg.text()));
```

`page.on()` stays entirely inside **Your Script (Node.js)**. It does not send a setup message down the wire to Chrome.

You are simply telling Node.js:

> "Hey, if Chrome ever sends a console message, pass it to this function."

Because you are just registering a listener in your script's memory, the setup is immediate.

**No waiting is required. Therefore, there is nothing to `await`.**

---

## 2. Why `page.route()` MUST Use `await`

### Example

```javascript
await page.route('**/api/fruits', route =>
  route.fulfill({ json: [] })
);
```

`page.route()` needs Playwright to set up a network interception rule for the browser.

Conceptually, your script tells the browser:

> "For this URL, intercept the request and return this mock response."

The browser needs to receive that instruction and complete the route setup.

**`await` forces your script to wait until the route setup is complete before continuing.**

---

## What Happens If You Forget `await` on `page.route()`?

```javascript
// ❌ No await
page.route('**/api/fruits', route =>
  route.fulfill({ json: [] })
);

await page.goto('https://example.com');
```

Without `await`, your script does not wait for the asynchronous route setup to finish.

This can create a **race condition**:

1. The route setup starts.
2. The script immediately continues to `page.goto()`.
3. The website loads and may immediately fire `/api/fruits`.
4. If the route is not ready yet, the request may not be intercepted as intended.

So the important point is:

> **The route must be ready before the application starts making the request.**

---

## Summary

### `page.on()`

```javascript
page.on('console', msg => console.log(msg.text()));
```

**Why no `await`?**

It simply registers a listener locally in the test script.

> **"Tell me when this event happens."**

No asynchronous setup needs to be completed before the script continues.

### `page.route()`

```javascript
await page.route('**/api/fruits', route =>
  route.fulfill({ json: [] })
);
```

**Why `await`?**

The route registration is asynchronous. The test should wait for the interception rule to be set up before continuing to an action such as `page.goto()` that can trigger the API request.

> **"Set up this interception first, then continue."**

### One line to remember

> **`page.on()` = listen for an event → no `await`**
>
> **`page.route()` = set up a network interception → use `await`**
