# Technical Depth (Manual Perspective)



## 1. Testing "Invisible" Features: No UI? No Problem.

Most testers learn on UIs. Click a button, see a result, pass or fail. When the UI disappears — APIs, background jobs, data pipelines — many testers feel lost because their primary feedback mechanism is gone. The mindset shift required is this: **the UI was never the system. It was just a window into the system. Now you're looking through a different window.**

### Testing APIs Manually

An API is actually *easier* to test than a UI in many ways — it's more deterministic, less prone to rendering quirks, and gives you direct access to the system's behaviour without UI noise in the way.

**Your toolset** is tools like Postman, Insomnia, or even cURL in a terminal. But the tool is the least important part. What matters is your mental model of what you're testing.

For every API endpoint, you're really asking six questions:

**Does it do what it claims?** Send a valid request, verify the response body matches the contract — correct fields, correct data types, correct values. If the API documentation says `user_id` is an integer, verify it's never returned as a string. If it says `status` can only be `active` or `inactive`, verify no other value appears.

**Does it reject what it shouldn't accept?** This is where most scripted test suites are thin. Send malformed JSON. Send missing required fields. Send a string where an integer is expected. Send a negative number where only positives make sense. Send an empty string. Send a field value 10x longer than any reasonable maximum. A well-built API rejects these gracefully with meaningful error codes — not a 500, not silence, not a crash.

**Does it enforce authorisation correctly?** This is frequently the most critical gap. Can a standard user call an admin endpoint? Can User A retrieve User B's data by manipulating an ID parameter in the request? These are IDOR (Insecure Direct Object Reference) vulnerabilities, and they're found constantly in manual API testing by simply changing one numeric ID in a request and seeing what comes back.

**Does it behave correctly under sequencing?** APIs rarely exist in isolation. What happens if you call "cancel order" before "create order"? What if you call "confirm payment" twice? Sequence-based testing reveals state management bugs that single-request testing never surfaces.

**Are headers and status codes correct?** A 200 response with an error message inside the body is a bug. A 404 when the resource exists is a bug. A 401 vs 403 confusion is both a bug and potentially a security issue. HTTP semantics exist for a reason.

**Does pagination, filtering, and sorting work correctly?** These are consistently undertested. Send a `page=0` — does it crash or handle gracefully? Request 10,000 records per page — what happens? Sort ascending and descending — do the results actually change and are they actually in order?

### Testing Background Jobs

Background jobs are the dark matter of software. They run silently, often on a schedule, and produce side effects — emails sent, records updated, files generated, queues populated — rather than direct responses. You cannot click them. You have to **observe their footprints**.

Your testing approach is forensic: you examine what the job *did*, not what it *said*.

Before triggering a job, capture the **pre-state**: relevant database records, queue depths, files in a directory, external service states. Trigger the job. Then examine the **post-state** and compare. The delta should exactly match what the job was supposed to do — no more, no less.

Specific things to probe: Does the job run on schedule, or can it be triggered manually for testing? What happens if it runs twice (idempotency)? What happens if it crashes halfway — does it leave the system in a partially-processed state, or does it roll back cleanly? What happens if the data it's processing is malformed — does it skip gracefully, log an error, or take down the entire job? What happens when the job processes zero records versus thousands?

For email-sending jobs specifically, use tools like Mailtrap or Mailhog in non-production environments — these intercept outgoing emails so you can inspect them without spamming real addresses.

### Testing Data Pipelines

A data pipeline moves data from source to destination, often transforming it along the way. Your testing is fundamentally about **data integrity across the entire chain**.

You start at the source: is the input data what you expect? You end at the destination: is the output data correct? And you investigate every transformation in between.

The key testing techniques are **lineage tracing** (pick a specific record and follow it through every stage of the pipeline — does it arrive at the destination intact and correctly transformed?), **volume testing** (does the pipeline handle 10 records the same way it handles 10 million?), **boundary injection** (deliberately introduce records with null values, duplicate keys, maximum-length strings, special characters, and timezone edge cases — these are the records pipelines choke on), and **reconciliation** (the row count going in should match the row count coming out, adjusted for any intentional deduplication or filtering — a discrepancy is always worth investigating).



## 2. Exploratory Testing: Finding What Scripts Miss

Let me walk through a realistic scenario that illustrates the thought process, because the scenario matters less than the *thinking* behind it.

### The Setup

Imagine you're testing a multi-tenant SaaS platform — a project management tool. Scripted test cases cover the happy paths: create a task, assign it, complete it, archive it. Sprint after sprint, they pass. The feature appears stable.

During a regression cycle, you're given two hours of free exploratory testing time. No script. No checklist. You sit down and ask yourself: *"What do I believe about this system, and what happens if I'm wrong?"*

### The Thought Process

You start with **assumptions inversion**. The scripted tests assume a single, well-behaved user doing things in order. Real users don't behave that way. What happens with concurrent users? What happens when someone does things out of sequence?

You open two browser sessions — two different user accounts in the same organisation. In Session A, you open Task #101 and start editing the description. In Session B, you simultaneously open the same task and change its status to "Completed." You go back to Session A and save your description edit.

The task is now in a conflicted state: the status shows "Completed" but internally, the last write (the description save from Session A) has silently overwritten the status change from Session B, resetting the task to "In Progress" in the database, while the UI shows "Completed." No error. No warning. No conflict notification.

This is a **last-write-wins race condition** with invisible data corruption — and no scripted test found it because scripted tests, by nature, are sequential.

### Why the Script Missed It

Scripted test cases are excellent at verifying *specified* behaviour. They are structurally incapable of discovering *unspecified* behaviour, because you have to know what you're looking for to write a script for it. Nobody wrote a requirement that said "concurrent edits should be handled gracefully" — so nobody scripted a test for it. But every user on the platform is a potential concurrent editor.

Exploratory testing finds these bugs because it follows **curiosity and risk intuition** rather than a predetermined path. The thought process is: *this system has multiple users, data has state, state changes can collide — what happens when they do?*

### The Broader Thought Framework for Exploratory Sessions

Good exploratory testing isn't random clicking. It's structured curiosity, guided by **charters**: focused missions with a defined target and time box. A charter sounds like: *"Explore the task editing feature with a focus on concurrent user interactions for 45 minutes."*

Within that charter, your thinking follows a pattern of **hypothesis and probe**. You form a hypothesis (*"I think concurrent edits might cause a conflict"*), design a quick experiment to test it, observe the result, and let the result guide your next hypothesis. If the system handles concurrent edits fine, your next hypothesis might be: *"What about concurrent edits across different organisations — could data leak between tenants?"*

You're also constantly asking: *what does this system trust that it shouldn't?* It trusts that users edit one thing at a time. It trusts that API calls arrive in order. It trusts that the client clock matches the server clock. Each of those trust assumptions is a test target.



## 3. Complex Database Testing: Verifying Business Logic Manually

Large-scale distributed databases and third-party integrations are where business logic goes to hide. The UI might show you a result, but you have no idea if the underlying data is correct unless you look directly.

### The Fundamental Principle

**Never trust the UI to validate data correctness.** The UI shows you what the application *thinks* the data is. The database shows you what the data *actually is*. These can differ in subtle and dangerous ways — caching layers, ORM quirks, eventual consistency delays, and transformation errors all create gaps between what the screen shows and what's stored.

Your approach: **UI assertion followed by direct database verification.** Every meaningful test completes a user action in the UI, then drops to the database level to verify the correct records were created, updated, or deleted with the correct values.

### SQL as a Testing Tool

For relational databases, you need working SQL — not at a developer level, but enough to query, join, and aggregate. The queries that matter most in testing are:

**Existence checks** — after creating a record through the UI, does it exist in the correct table with the correct values? `SELECT * FROM orders WHERE order_id = 12345` should return exactly one row, and every column should match what you submitted.

**Absence checks** — after deleting a record (or soft-deleting it), is it actually gone (or correctly flagged)? Many systems have a `deleted_at` timestamp column — verify it's populated rather than assuming the record disappeared.

**Aggregate validation** — if a business rule says "an order total is the sum of line items minus applicable discounts plus tax," write a query that calculates this independently and compare it to what the application stored. Discrepancies are business logic bugs.

**Referential integrity** — in a distributed system, creating a record in one service should create corresponding records in related services. Query both sides to verify the relationship exists and is consistent.

**Audit and history tables** — many enterprise systems maintain a full audit log of every change. Verify that every action in the UI produces the correct audit record — correct user, correct timestamp, correct before/after values. Audit trail bugs are often invisible at the UI level and catastrophic during compliance reviews.

### Testing Third-Party Integrations

Third-party integrations introduce a category of complexity that pure database testing doesn't cover: **you don't own the other side**.

Your testing strategy for integrations focuses on three boundaries:

**What you send** — inspect the outbound request to the third party. Is it correctly formed? Does it contain the right data? Use network interception tools (Charles Proxy, Fiddler, or built-in browser dev tools for REST) to capture exactly what leaves your system. Compare it against the third party's API specification.

**What you receive** — simulate different responses from the third party, including success responses, error responses, malformed responses, timeouts, and empty responses. Most integration bugs live in error handling: systems that work perfectly when the third party responds normally often collapse catastrophically when it responds unexpectedly. Tools like WireMock or Mockoon let you stub third-party responses to test these scenarios without hitting live external systems.

**What you do with it** — after receiving a response from the third party, verify your system processed it correctly. If a payment gateway returns a successful charge token, verify it's stored in your database against the correct order. If a shipping API returns a tracking number, verify it's associated with the correct shipment record.

### Handling Eventual Consistency in Distributed Systems

This is the trickiest scenario in database testing. In a distributed system, a write to one service doesn't immediately appear in another — there's propagation delay through queues, event streams, or replication. A test that checks data immediately after an action may fail not because there's a bug, but because the data hasn't arrived yet.

Your approach here is **polling with timeout**: after triggering an action, query the target system at short intervals for a defined maximum wait period. If the data arrives within the expected window, the test passes. If it doesn't arrive, or arrives with incorrect values, you have a genuine defect.

More importantly, you're testing the *guarantees* the system makes. If the system claims "your dashboard will update within 30 seconds," that's a testable assertion — trigger an event, wait 30 seconds, check the dashboard. If it claims "eventual consistency with no time guarantee," your test is about correctness of the data when it *does* arrive, not about timing.

For third-party integrations specifically, always test with **both live sandbox environments and mocked responses**. Sandboxes verify real connectivity and real protocol behaviour. Mocks verify your error handling. You need both — a sandbox that always returns success will never tell you how your system handles a timeout.



The common thread across all three of these areas is **moving from surface-level observation to system-level understanding**. Invisible features, exploratory discovery, and database verification all require a tester who thinks about systems — not just screens. That depth is what elevates testing from verification to genuine quality intelligence.
