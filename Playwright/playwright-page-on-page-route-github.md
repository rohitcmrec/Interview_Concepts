# Playwright: `page.on()`, `page.once()`, `page.route()`, and `waitForEvent()`

The easiest way to think about it is:

> "Hey Playwright, register this event. When it happens, run this callback."

That is the core idea behind `page.on()` and `page.once()`.

---

## 1) `page.on()` and `page.once()` are listeners

They are not functions that execute immediately. They are event handlers.

- `page.on('event', callback)`
  - keeps listening
  - runs callback every time the event happens

- `page.once('event', callback)`
  - listens only for the first time
  - then removes itself automatically

### Example: dialog

```javascript
page.once('dialog', async dialog => {
  console.log(dialog.message());
  await dialog.accept();
});

await page.getByRole('button', { name: 'Trigger Alert' }).click();
```

Here, the registration is instant. But the callback does not run immediately. It runs later when the browser triggers the `dialog` event.

---

## 2) Registration is sync, callback is async

This is the important part.

When you do this:

```javascript
page.on('console', msg => {
  console.log(msg.text());
});
```

You are only telling Playwright:

> "Store this callback in my listener list. If a console event happens later, call this function."

So:

1. The listener is registered immediately
2. The script continues running
3. Later, when the browser triggers the event, the callback executes

This is classic Node.js event-driven behavior.

So yes, your understanding is correct:

- the registration itself is synchronous
- the callback execution is asynchronous

---

## 3) `page.on()` vs `page.once()`

| Method | Behavior | Best use |
|--------|----------|----------|
| `page.on()` | Listens every time | Logging all console errors, tracking multiple dialogs |
| `page.once()` | Listens only once | One-off alert, first popup, single-time action |

### Example: multiple alerts

```javascript
page.on('dialog', async dialog => {
  await dialog.accept();
});
```

This handles every dialog that appears.

### Example: one alert only

```javascript
page.once('dialog', async dialog => {
  await dialog.accept();
});
```

This handles only the next dialog and then stops.

---

## 4) Why `page.route()` is different from `page.on()`

This is where many people get confused.

`page.on()` is mostly local to your script.

`page.route()` has to tell the browser:

> "Before this request goes through, intercept it and handle it my way."

So when you write:

```javascript
await page.route('**/api/fruits', async route => {
  await route.fulfill({ json: [{ name: 'Apple' }] });
});
```

Playwright sends a command to the browser to set up a network interception rule. The browser must confirm that it has set the rule before your test continues.

That is why `await` is needed here.

### Without `await`

```javascript
page.route('**/api/fruits', async route => {
  await route.fulfill({ json: [{ name: 'Apple' }] });
});

await page.goto('https://example.com');
```

This can be a race condition:

- browser starts loading page
- request may fire before route is fully set
- route not ready in time
- request hits real API instead of mock data

So `page.route()` is not just "registering a callback" in Node.js. It is setting up a browser-level rule, and the browser must acknowledge it.

---

## 5) `page.on()` is not waiting; `waitForEvent()` is waiting

This is another very important distinction.

### `page.on()` / `page.once()`

These are listeners.

They answer:

> "What should I do when this event happens?"

### `waitForEvent()`

This is a Promise-based wait.

It answers:

> "Wait until this event happens, then give me the event object."

### Example: popup

```javascript
const popupPromise = page.waitForEvent('popup');
await page.getByRole('button', { name: 'Open New Window' }).click();
const popupPage = await popupPromise;
```

This is useful when you want the popup page object so you can interact with it.

You can also do:

```javascript
const [popupPage] = await Promise.all([
  page.waitForEvent('popup'),
  page.getByRole('button', { name: 'Open New Window' }).click(),
]);
```

This is the cleanest pattern when you need to wait for the popup and trigger the action at the same time.

---

## 6) The real mental model

Imagine your test is split into two worlds:

1. Your Node.js script
2. The browser

`page.on()` is like saying:

> "In my script, keep this callback ready. When the browser sends an event, I will handle it."

`page.once()` is the same, but only once.

`page.route()` is like saying:

> "Browser, set up a network interception rule now. I need you to prepare before the app does its real request."

And `waitForEvent()` is:

> "Pause my test until this event is fired, then give me the result."

---

## 7) Very short summary

- `page.on()` = listen forever
- `page.once()` = listen once
- both are async callbacks triggered later by browser events
- `page.route()` needs `await` because it configures browser behavior
- `waitForEvent()` is a Promise that waits until the event occurs and returns the event object

---

## Quick examples

### Dialog handling

```javascript
page.once('dialog', async dialog => {
  await dialog.accept();
});

await page.getByRole('button', { name: 'Delete' }).click();
```

### Popup handling

```javascript
const [popupPage] = await Promise.all([
  page.waitForEvent('popup'),
  page.getByRole('button', { name: 'Open New Window' }).click(),
]);

await popupPage.getByRole('button', { name: 'Log In' }).click();
```

### Network mocking

```javascript
await page.route('**/api/fruits', async route => {
  await route.fulfill({ json: [{ name: 'Apple' }] });
});

await page.goto('https://example.com');
```

This is the simplest way to understand it: `on`/`once` are listeners, `waitForEvent` is a wait, and `route` is browser configuration that needs to be prepared before the app continues.
