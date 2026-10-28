# Senior QA — Test Strategy, Prioritisation & Risk-Based Testing

## 1. Creating a Test Strategy for a New Product with Minimal Documentation

When documentation is sparse or unclear, most testers panic. The reality is this scenario is more common than the ideal, and a good test strategy doesn't require perfect documentation — it requires **structured thinking and intelligent information gathering**.

### Phase 1: Intelligence Gathering (Before Writing a Single Test Case)

Your first job is to become a detective. You mine every available source of truth:

* **Stakeholder Interviews** — Talk to the product owner, business analyst, developers, and even sales/support teams. Ask questions like:

  * *What does this product need to do to be considered a success?*
  * *What would make a customer angry?*
  * *What keeps you up at night about this release?*

  These conversations surface implicit requirements that no one bothered to write down.

* **Competitor and Domain Analysis** — If it's a new product in an existing market, study competitors. A payment gateway, for example, must handle certain flows regardless of whether anyone documented them, because the domain demands it.

* **Exploratory Sessions** — Even with a prototype or early build, run unscripted exploratory testing sessions. Your goal isn't to find bugs yet — it's to *understand the system*. Map what you find into a rough feature inventory.

* **Technical Artefacts** — API contracts, database schemas, infrastructure diagrams, and deployment scripts often reveal behaviour that no functional document mentions. A schema with a `is_premium_user` flag tells you there's a tiered access model that needs testing, even if it's never in a spec.

### Phase 2: Defining the Strategy Structure

Once you have enough understanding, your test strategy document should cover:

* **Scope** — What is in scope and, critically, what is explicitly out of scope. Stating out-of-scope items protects you later. For a minimal-documentation product, scope is based on your gathered intelligence, not a signed-off spec.

* **Risk Assessment** — Identify the highest-risk areas first. For a new product, risk typically concentrates around core user journeys, data integrity, integrations with external systems, and security boundaries.

* **Test Levels and Types** — Define which types of testing apply: functional, integration, regression, performance, security, usability, and so on. For an early-stage product, you probably won't run exhaustive performance tests — but you should at minimum have a position on it.

* **Entry and Exit Criteria** — What conditions must be met before testing begins, and what does "done" look like? Without clear exit criteria, testing never officially ends and stakeholders fill that vacuum with arbitrary deadlines.

* **Test Approach** — Will you use scripted test cases, exploratory charters, or a hybrid? For unclear-documentation products, a **charter-based exploratory approach** often works better early on, with scripted cases added as behaviour stabilises.

* **Assumptions and Dependencies** — Document every assumption you're making because of the documentation gaps. This is a critical protective measure. If you assume that only logged-in users can access the checkout, write that assumption down. If it's wrong, you've created a paper trail that shows the information was never provided.

* **Defect Management and Reporting** — How will bugs be logged, prioritised, and communicated? What severity classifications will you use?

### Phase 3: Iteration

A test strategy for a new product is a living document. Expect to revise it as the product evolves and documentation improves. Set a cadence — say, weekly — to review and update the strategy as you learn more. This isn't a weakness; it's intellectual honesty about working in uncertainty.

---

## 2. Prioritisation Under Pressure: 2 Hours, 50 Test Cases Left

This is one of the most important real-world skills a tester can have, and the answer is never "test as many as possible as fast as possible." Speed-running through 50 test cases in 2 hours produces low-quality coverage and false confidence.

The real question is:

> **What risk can we afford to ship with, and what risk can we not?**

### The Mental Framework: Think in Failure Modes

Before touching your test case list, ask yourself:

> *If something breaks in production tonight, what breaks the business?*

That answer defines your priorities.

Group your 50 test cases into four buckets:

### Must Test — Business Critical

These are flows where failure means revenue loss, data corruption, security breach, or legal exposure.

For an e-commerce app:

* Completing a purchase
* Processing payment
* User authentication

These get tested first, completely, no exceptions.

### Should Test — High Visibility / High Frequency

These are features that a large proportion of users will touch, or that support will get flooded with calls about.

Examples:

* Homepage load
* Core navigation
* Search functionality

Test these next, but you may accept slightly lighter coverage — one representative path rather than every edge case.

### Could Test — Lower Risk, Lower Frequency

Examples:

* Edge cases
* Less-used features
* Cosmetic issues

In 2 hours with 50 cases outstanding, these likely don't get touched.

### Won't Test — Formally Acknowledged

This is the most professional thing you can do: explicitly document what you are **not** testing and why.

Send a message to your manager and the release owner:

> "Given time constraints, the following areas have not been tested in this cycle: [list]. Risks associated with this decision: [list]. Recommend monitoring these post-release."

This transforms an uncomfortable situation into a managed, communicated risk rather than a silent gap.

### Practical Prioritisation Signals

When sorting your remaining 50 cases, use these signals:

1. **Recent code changes** — Anything the developer touched in the last sprint is higher risk than stable, unchanged code. Ask for a diff or a summary of what changed.

2. **Failed tests from previous cycles** — Areas that have historically been buggy are higher risk than areas that have been clean for months.

3. **Integration points** — Anything that crosses system boundaries (APIs, third-party services, database writes) fails in more interesting and damaging ways than self-contained UI behaviour.

4. **User journey criticality** — A broken "forgot password" flow is annoying; a broken "checkout" flow is catastrophic.

### What You're NOT Doing

You're not:

* Skipping tests randomly.
* Testing everything shallowly.
* Staying silent about the gaps.

Every decision has a rationale, and that rationale is communicated **before the release, not after something breaks**.

---

## 3. Risk-Based Testing (RBT): Implementation and Stakeholder Justification

Risk-Based Testing is the most mature and defensible approach to test prioritisation.

It's the answer to the perennial question:

> *"Why aren't you testing everything?"*

And the answer is:

> *"Because not everything has equal potential to cause harm, and resources are finite."*

### The Core Concept

RBT prioritises testing effort based on **two axes for every feature or component**:

#### 1. Likelihood of Failure

How probable is it that this area contains a defect?

Factors include:

* Code complexity
* Developer experience
* How recently the code was written or changed
* How well the requirements were understood
* Historical defect density

#### 2. Impact of Failure

If this area does fail, how bad is it?

Factors include:

* Financial impact
* Reputational damage
* Regulatory exposure
* Number of users affected
* Reversibility — can it be rolled back easily?

### Building a Risk Register

The practical implementation starts with a **Risk Register** — a structured list of your application's functional areas mapped against these two dimensions.

You score each area — typically on a **1–5 scale** for both likelihood and impact — and multiply them to get a **Risk Priority Number (RPN)**.

Example:

| Functional Area            | Likelihood | Impact |    RPN |
| -------------------------- | ---------: | -----: | -----: |
| Payment processing         |          4 |      5 | **20** |
| User profile avatar upload |          2 |      1 |  **2** |

A payment processing module might score 4 on likelihood (complex, frequently changed) and 5 on impact (direct revenue), giving an RPN of 20.

A user profile avatar upload might score 2 on likelihood and 1 on impact, giving an RPN of 2.

### Testing Depth Based on RPN

Your testing depth is then proportional to the RPN.

**High-RPN areas** get:

* Comprehensive scripted test suites
* Boundary value analysis
* Negative testing
* Exploratory sessions

**Low-RPN areas** might get:

* A single happy-path check
* Or be deferred entirely

### What RBT Looks Like in Practice

For a medium-sized product, your risk register might identify:

| Risk Level  |   Approx. Areas | Testing Effort |
| ----------- | --------------: | -------------: |
| High Risk   |             5–8 |            70% |
| Medium Risk |           10–15 |            25% |
| Low Risk    | Remaining areas |             5% |

This is a deliberate, defensible allocation — not neglect.

RBT also informs **test case design**.

**High-risk areas** get richer test cases:

* More equivalence partitions
* More boundary values
* More negative scenarios
* More state-based variations

**Low-risk areas** get thinner test cases:

* One or two representative paths

### Justifying RBT to Stakeholders

This is where RBT's real power lies.

Non-technical stakeholders often believe that:

> **"More testing = more quality"**

and will push back on any scope reduction.

RBT gives you a language they understand:

> **Business risk**

A stakeholder conversation might sound like this:

> "We have identified 47 testable areas in this release. Based on likelihood of defect and business impact, 8 areas carry 80% of the total risk. We are allocating 70% of our testing effort to those 8 areas. The remaining 39 areas carry 20% of the risk. We will provide lighter coverage there and accept residual risk in those zones. Here is the risk register so you can review and challenge our assumptions."

Three things make this approach stakeholder-proof:

1. **Business language** — You're speaking in terms of revenue, reputation, and users affected, not technical ones.
2. **Transparency** — You're being transparent rather than making scope decisions in a black box.
3. **Shared ownership** — You're inviting stakeholders to **challenge your risk assumptions**. If they disagree with a prioritisation, they can say so explicitly and own that decision jointly.

This fundamentally shifts the dynamic from:

> **"QA didn't test X"**

to:

> **"We collectively decided X was lower risk."**

### Common Pitfalls in RBT

#### Pitfall 1: Never Revisiting Risk

The biggest mistake is doing risk assessment once and never revisiting it.

Risk profiles change with every sprint.

A low-risk module that gets a major refactor overnight becomes high-risk.

**Recommendation:** Build risk review into your sprint ceremonies — even five minutes in sprint planning to re-evaluate the risk register pays significant dividends.

#### Pitfall 2: Using RBT to Skip Testing

The second pitfall is treating RBT as a justification to skip testing rather than a framework to **allocate testing intelligently**.

Low-risk areas still get tested — just not as deeply.

The goal is:

> **Optimal coverage, not minimal coverage.**

---

## Key Takeaway

These three competencies:

1. **Building strategy from ambiguity**
2. **Prioritising under pressure**
3. **Managing risk systematically**

are what separate a reactive tester from a strategic QA professional.

The common thread across all three is that you're always making:

> **Explicit, communicated, justified decisions rather than implicit ones.**

That transparency is what builds credibility with your team and stakeholders.
