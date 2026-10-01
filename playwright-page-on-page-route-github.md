# Playwright: `page.on()`, `page.once()`, `page.route()`, and `waitForEvent()`

## Core Concepts

### Key Mental Model: Local vs. Remote Work

Your test runs in **two separate programs**:
1. **Node.js Script** – Your test code
2. **Browser (Chrome)** – The actual website

They communicate over a WebSocket connection.

---

## `page.on()` vs `page.once()`

Both are **event listeners** that register synchronously but execute asynchronously.

| Feature | `page.on()` | `page.once()` |
|---------|-----------|--------------|
| **Scope** | Persistent—stays active until `page.off()` | One-time—auto-removes after first trigger |
| **Use Case** | Continuous tracking (all errors, all requests) | Single contextual action (one dialog, one popup) |
| **Setup** | Synchronous, instant | Synchronous, instant |
| **Execution** | Asynchronous, event-driven | Asynchronous, event-driven |

### Understanding Async Behavior

When you register an event listener:
```javascript
page.once('dialog', async dialog => {
  console.log('Dialog arrived!');
  await dialog.accept();
});

// ← Code moves on instantly; callback executes later when event fires
await page.getByRole('button', { name: 'Trigger Alert' }).click();
```

**The execution order:**
1. Listener registration happens (synchronously)
2. Main code continues to next line
3. When browser fires the event, callback executes asynchronously
4. Callback completes independently of main test flow

---

## Why `page.route()` Requires `await` (But `page.on()` Doesn't)

### The Difference

**`page.on()` stays inside Node.js:**
```javascript
page.on('console', msg => console.log(msg.text()));
// Just registers a local handler. No wire needed. Instant.
```

**`page.route()` sends commands to the browser:**
```javascript
await page.route('**/api/fruits', async route => {
  await route.fulfill({ json: [] });
});
// Sends WebSocket message: "Browser, set up a network trap"
// Waits for browser response: "Trap is set. Ready."
```

### What Happens Without `await`

```javascript
// ❌ WRONG
page.route('**/api/fruits', route => route.fulfill({ json: [] }));
await page.goto('https://example.com');

// Race condition: Website loads before the trap is set up
// → Real API call gets hit instead of being mocked
```

**With `await`:**
```javascript
// ✅ CORRECT
await page.route('**/api/fruits', route => route.fulfill({ json: [] }));
await page.goto('https://example.com');

// Browser confirms trap is active before page loads
// → All matching requests get intercepted
```

---

## `once()`/`on()` vs `waitForEvent()`: When to Use Each

### The Distinction

- **`once()` / `on()`** → Answer: *"What should I do when this event happens?"*
  - Registers a callback handler
  - Callback executes asynchronously in the background
  - Main test flow continues independently

- **`waitForEvent()`** → Answer: *"I want my test to pause and wait for this event"*
  - Returns a Promise you can `await`
  - Main test pauses until event fires and you get the event object
  - Better for when you need the event object to continue testing

### Real-World Examples

**Dialog (Use `once()`):**
```javascript
page.once('dialog', async dialog => {
  // Callback handles the dialog—test doesn't need to wait
  await dialog.accept();
});

await page.getByRole('button', { name: 'Delete' }).click();
// Test continues; dialog is handled automatically
```

**Popup (Use `waitForEvent()`):**
```javascript
// Main test NEEDS the popup object to interact with it
const popupPage = await page.waitForEvent('popup');
await page.getByRole('button', { name: 'Open New Window' }).click();

// Now we have the popup page object
await popupPage.getByRole('button', { name: 'Log In' }).click();
```

Or with `Promise.all()` to avoid race conditions:
```javascript
const [popupPage] = await Promise.all([
  page.waitForEvent('popup'),
  page.getByRole('button', { name: 'Open New Window' }).click()
]);

await popupPage.getByRole('button', { name: 'Log In' }).click();
```

---

## Event Loop Execution Timeline

```javascript
await page.route('**/api/fruits', async route => {
  console.log('🔄 3. Network intercepted');
  await route.fulfill({ json: [{ name: 'Apple' }] });
});

await page.goto('https://example.com');
console.log('🌐 1. Page loaded');

await page.getByRole('button', { name: 'Load Fruits' }).click();
console.log('👆 2. Button clicked');

await expect(page.getByText('Apple')).toBeVisible();
console.log('✅ 4. Test passed');
```

**Execution order:**
1. `🌐 1. Page loaded` (Main thread)
2. `👆 2. Button clicked` (Main thread)
3. `🔄 3. Network intercepted` (Callback wakes up when API fires)
4. `✅ 4. Test passed` (Main thread, after mock response)

---

## Quick Reference

| Task | Solution | Why |
|------|----------|-----|
| Handle a **single dialog** | `page.once('dialog', ...)` | Dialog handled by callback; main test doesn't need the object |
| Handle **multiple dialogs** | `page.on('dialog', ...)` | Need persistent handler |
| Capture a **popup window** | `await page.waitForEvent('popup')` | Main test needs the popup `Page` object to interact with it |
| **Intercept network** | `await page.route(...)` | Browser needs confirmation before continuing |
| **Listen to console** | `page.on('console', ...)` | No `await` needed; happens locally in Node.js |
