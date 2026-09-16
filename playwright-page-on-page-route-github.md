just like page.on(), page.route() also asynchronous

14 September 2026

**Yes,** **`page.route()`** **is an asynchronous operation in Playwright that returns a Promise, meaning you should use** **`await page.route(...)`** **in async/await environments.**

**Key Differences and Behavior**

- **Registration vs Execution**: Calling `await page.route()` registers the network interceptor asynchronously. However, the internal handler callback itself is triggered dynamically whenever a matching network request in-flight occurs on the page. [(https://mergify.com/blog/playwright-route-handlers-after-page-goto),(https://www.linkedin.com/pulse/mastering-real-time-browser-interactions-playwrights-pageon-v-xt20c)]
- **Handling Promises inside Routes**: The callback function provided to `page.route()` can also be marked as `async`, allowing you to call asynchronous methods on the `route` object (such as `await route.fulfill()`, `await route.continue()`, or `await route.fetch()`). [(https://github.com/microsoft/playwright/issues/26689),(https://playwright.dev/docs/api/class-route),(https://qaskills.sh/blog/playwright-route-fulfill-mock-api-guide)]
- **Comparison with** **`page.on()`**: Unlike `page.on()`, which merely attaches an event listener synchronously to stream notifications (like console logs or request lifecycles), `page.route()` actively alters network traffic and requires proper asynchronous setup before triggering navigations or actions that fire those requests. [(https://www.linkedin.com/pulse/when-use-pageroute-pageon-playwright-shilpashree-v-6lnnc),(https://www.linkedin.com/pulse/mastering-real-time-browser-interactions-playwrights-pageon-v-xt20c),(https://github.com/microsoft/playwright/issues/26689),(https://mergify.com/blog/playwright-route-handlers-after-page-goto)]

## You said: what is page.on() or page.once () which is used to handle alerts, window pop-up etc

what is page.on() or page.once () which is used to handle alerts, window pop-up etc

14 September 2026

**`page.on()`** **and** **`page.once()`** **are event listeners** used in Playwright and Puppeteer to listen for specific browser events like alerts, dialogs, console messages, or new popup windows.

The primary difference lies in **how many times** they trigger:

- **`page.on()`** **(Persistent Listener):** Listens for the specified event **every single time** it happens during the lifecycle of the page.
- **`page.once()`** **(One-Time Listener):** Listens for the specified event **only the first time** it happens, and then automatically removes itself.

---

**How They Apply to Dialogs and Popups**

**1. Handling Browser Alerts (Dialogs)**

By default, Playwright automatically dismisses dialogs (like `alert`, `confirm`, or `prompt`) so your tests don't freeze. If you want to accept or type text into them, you must register a listener **before** the action that triggers the dialog.

- **Using** **`page.once()`** **(Recommended for single alerts):**

  **javascript**
  ```
  // Listen for the next dialog, accept it, then stop listening
  page.once('dialog', async dialog => {
    console.log(dialog.message());
    await dialog.accept();
  });

  await page.getByRole('button', { name: 'Trigger Alert' }).click();

  ```

- **Using** **`page.on()`** **(If a page throws multiple alerts):**

  **javascript**
  ```
  // This will handle every dialog that pops up on this page
  page.on('dialog', async dialog => {
    await dialog.accept();
  });

  ```

**2. Handling Window Popups (New Tabs/Windows)**

When clicking a link opens a `target="_blank"` popup window, you can use the `popup` event to capture the new page object.

Since a specific button click usually spawns exactly one popup, **`page.once()`** or a Promise-based approach is ideal:

**javascript**

```
// Start waiting for the popup event
const popupPromise = page.waitForEvent('popup');

// Click the button that opens the new tab
await page.getByRole('button', { name: 'Open New Window' }).click();

// Await the popup page object
const popupPage = await popupPromise;
await popupPage.waitForLoadState();
console.log(await popupPage.title());

```

---

**Summary Comparison**

| **Feature****`page.on()page.once()`** |                                                                                        |                                                                                      |
| ------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **Lifespan**                          | Stays active until manually turned off using `page.off()`.                             | Automatically destroys itself after firing once.                                     |
| **Best Used For**                     | Continuous tracking (e.g., logging all `console` errors or tracking network requests). | One-off contextual actions (e.g., handling a specific `dialog` or a single `popup`). |
| **Execution**                         | Synchronously attaches the listener.                                                   | Synchronously attaches the listener.                                                 |

## You said: but they are also asynchronous and after registering event, code moves on and in event of the trigger, callback get executed

but they are also asynchronous and after registering event, code moves on and in event of the trigger, callback get executed

14 September 2026

You are **exactly right**. Your understanding of how node-style event emitters work is spot on.

When you run `page.on()` or `page.once()`, the registration itself happens instantly (synchronously), but **the execution of your callback function is asynchronous and event-driven**. The main test code registers the listener, immediately moves on to the next lines of code, and whenever the browser eventually triggers that event (like an alert popping up), your callback function wakes up and executes.

Because of this asynchronous, decoupled nature, handling events can sometimes lead to **race conditions** in tests if not handled carefully.

---

**The Common Pitfall: Race Conditions**

If you register a listener and then immediately trigger an action, a race condition can occur if the action happens faster than the event loop can process it, or if your code moves on before the callback finishes.

Look at this **problematic example**:

**javascript**

```
// 1. Register the listener
page.once('popup', async newPage => {
  await newPage.getByRole('button', { name: 'Log in' }).click(); // Fast browser action
});

// 2. Code moves on instantly and triggers the popup
await page.getByRole('button', { name: 'Open Window' }).click();

// 3. CRITICAL FLAW: The test might end right here before the callback inside step 1 finishes!

```

In the code above, your test runner might mark the test as "Passed" and close the browser the split second step 2 finishes, completely cutting off the asynchronous callback inside step 1 before it gets to click the "Log in" button.

---

**The Solution: Combining Events with** **`Promise.all()`**

To ensure your test code waits for both the triggering action *and* the asynchronous callback to finish, the standard best practice in modern Playwright is to use `Promise.all()` with `page.waitForEvent()`.

`page.waitForEvent()` is built on top of `page.once()`, but it returns a Promise that you can actually `await`.

**javascript**

```
// This waits for BOTH the click to happen AND the asynchronous popup to be captured
const [popupPage] = await Promise.all([
  page.waitForEvent('popup'),                                    // Waits for the event trigger
  page.getByRole('button', { name: 'Open Window' }).click()       // Triggers the event
]);

// Now the code is forced to pause here until both are ready.
// You can safely interact with the new page:
await popupPage.getByRole('button', { name: 'Log in' }).click();

```

**Summary of what happens in the Event Loop:**

1. **`page.on(...)`** tells the system: *"Hey, put this callback in the background. Don't run it now."*
2. **The main script** keeps running sequentially line-by-line.
3. **The browser** encounters a trigger (e.g., a user click opens a window).
4. **The event loop** pauses the current execution context, pushes your callback into the queue, and executes your asynchronous handler.

## You said: When you run page.on() or page.once(), the registration itself happens instantly (synchronously), but the execution of your callback function is asynchronous and event-drivenkeeping this in mind, how about await page.route('\*/\*\*/api/v1/fruits', async route => {

When you run page.on() or page.once(), the registration itself happens instantly (synchronously), but the execution of your callback function is asynchronous and event-drivenkeeping this in mind, how about await page.route('\*/\*\*/api/v1/fruits', async route => {

14 September 2026

You have connected the dots perfectly. The concept you described for `page.on()` applies **exactly the same way** to `page.route()`, but with one additional layer regarding *network interception network setup*.

When you run this line:

**javascript**

```
await page.route('*/**/api/v1/fruits', async route => { ... });

```

**The Breakdown of What Happens:**

1. **The Registration (****`await page.route`****) is a setup step:**
   When your code hits this line, it tells the browser/Playwright engine: *"Hey, set up a network filter. If you ever see a request matching this URL pattern, pause it and hand it over to my callback."* The `await` here ensures that this filter is **successfully established** in the browser's network layer before the code moves to the next line.
2. **The Code Moves On Instantly:**
   Once the filter is registered, your main test script immediately moves on to the next line (e.g., clicking a button, navigating). It does **not** wait for any network requests to actually happen yet.
3. **The Callback is Asynchronous and Event-Driven:**
   Your `async route => { ... }` callback function doesn't execute right away. It sits idle in the background. Only when your application later fires a network request that matches `*/**/api/v1/fruits`, the browser intercepts it, pauses the request, and triggers your callback.

---

**A Visual Timeline of Execution**

Imagine this test script:

**javascript**

```
// Line 1: Register the route rule
await page.route('*/**/api/v1/fruits', async route => {
  console.log('🔄 3. Network Intercepted! Mocking data...'); // Callback execution
  await route.fulfill({ json: [{ name: 'Apple' }] });
});

// Line 2: Main thread continues instantly
await page.goto('https://example.com'); 
console.log('🌐 1. Page loaded');

// Line 3: Triggering the action that makes the API call
await page.getByRole('button', { name: 'Load Fruit List' }).click(); 
console.log('👆 2. Clicked the button');

// Line 4: Check the UI
await expect(page.getByText('Apple')).toBeVisible();
console.log('✅ 4. Test passed');

```

**The chronological order of logs will be:**

1. `🌐 1. Page loaded` (Main code moving forward)
2. `👆 2. Clicked the button` (Main code triggering the event)
3. `🔄 3. Network Intercepted! Mocking data...` (The asynchronous callback wakes up because the API call went out)
4. `✅ 4. Test passed` (The UI receives the mocked data, and the test finishes)

---

**Why** **`page.route()`** **needs** **`await`** **but** **`page.on()`** **does not**

You might wonder: *If both use asynchronous callbacks, why do we write* *`await page.route()`* *but just* *`page.on()`**?*

- **`page.on()`** is pure Node.js architecture. It just appends a function pointer to a local array in memory. This is completely synchronous and instantaneous.
- **`page.route()`** actually has to send a WebSocket command over to the browser instance (Chromium/Firefox) to tell its underlying network engine to set up an active traffic interception rule. This cross-process communication takes a few milliseconds, so Playwright forces you to `await` it to guarantee the rule is active *before* you start navigating or clicking.

## You said: still didn't understand

still didn't understand

14 September 2026

Let's strip away all the technical jargon. Let's look at **where the work actually happens**.

Your test is split into two completely separate programs on your computer:

1. **Your Script (Node.js)** – Where your test code lives.
2. **The Browser (Chrome)** – Where the actual website opens.

They talk to each other over a virtual wire.

---

**1. Why** **`page.on()`** **does NOT need** **`await`**

`page.on()` stays entirely inside **Your Script (Node.js)**. It does not send any messages down the wire to Chrome.

When you write:

**javascript**

```
page.on('console', msg => console.log(msg.text()));

```

You are simply telling Node.js: *"Hey, if Chrome ever decides to send a console message over the wire, pass it to this function."*

Because you are just setting a rule inside your own script's memory, it is instant. No wire is used. No waiting is required. Therefore, **there is nothing to** **`await`**.

---

**2. Why** **`page.route()`** **MUST use** **`await`**

`page.route()` forces **The Browser (Chrome)** to change how its internal engine works.

When you write:

**javascript**

```
await page.route('**/api/fruits', route => route.fulfill({ json: [] }));

```

This *instantly* sends a message down the wire to Chrome: **"Stop! Modify your internal network settings right now. Set up a trap for this URL."**

Chrome has to receive that message, set up the network trap, and then reply back over the wire: **"Okay, the trap is set."**

**The** **`await`** **forces your script to pause and wait for Chrome to say "Okay, the trap is set."**

---

**What happens if you forget** **`await`** **on** **`page.route()`****?**

If you don't use `await`, your script sends the message to Chrome but **refuses to wait for Chrome's response**. It moves straight to the next line:

**javascript**

```
// ❌ WRONG: No await
page.route('**/api/fruits', route => route.fulfill({ json: [] })); 

// Your script instantly runs this next line:
await page.goto('https://example.com'); 

```

Because your script didn't wait, a race happens:

1. The website loads and immediately fires the `/api/fruits` request.
2. Chrome is *still* busy setting up the network trap and hasn't finished yet.
3. The request slips right past the trap because it wasn't ready. Your mock fails, and the real website API gets hit instead.

**Summary:**

- `page.on()` is just taking a note in your local script. **No waiting needed.**
- `page.route()` is sending an order to a separate program (Chrome) to change its network hardware settings. **You must wait (****`await`****) for Chrome to confirm it's ready.**
