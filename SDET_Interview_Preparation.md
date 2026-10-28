# SDET Interview Preparation Guide
### Comprehensive Q&A — Senior Level

---

## Table of Contents

1. [Strategic Planning & Risk Management](#1-strategic-planning--risk-management)
2. [Technical Depth (Manual Perspective)](#2-technical-depth-manual-perspective)
3. [Process & Collaboration (Agile/DevOps)](#3-process--collaboration-agiledevops)
4. [Scenario-Based Challenges](#4-scenario-based-challenges)
5. [Automation — Tool Selection & ROI](#5-automation--tool-selection--roi)
6. [Test Strategy vs Test Plan](#6-test-strategy-vs-test-plan)
7. [Test Data Management](#7-test-data-management)
8. [Behavioural & Situational Questions](#8-behavioural--situational-questions)
9. [Interview Introduction](#9-interview-introduction)
10. [Questions to Ask the Panel](#10-questions-to-ask-the-panel)

---

## 1. Strategic Planning & Risk Management

---

### Q1. How do you create a Test Strategy for a brand-new product with minimal or unclear documentation?

When documentation is sparse, a good test strategy doesn't require perfect documentation — it requires structured thinking and intelligent information gathering.

#### Phase 1: Intelligence Gathering

- **Stakeholder Interviews** — Talk to product owners, business analysts, developers, and support teams. Ask: *What does success look like? What would make a customer angry? What keeps you up at night about this release?*
- **Competitor and Domain Analysis** — Study competitors in the same market. Domain knowledge fills gaps where documentation doesn't exist.
- **Exploratory Sessions** — Run unscripted sessions on prototypes or early builds to map feature inventory and understand system behaviour.
- **Technical Artefacts** — API contracts, database schemas, infrastructure diagrams, and deployment scripts reveal behaviour that functional documents miss.

#### Phase 2: Strategy Structure

- **Scope** — What is in scope and explicitly out of scope. Stating out-of-scope items protects you later.
- **Risk Assessment** — Identify highest-risk areas: core user journeys, data integrity, external integrations, security boundaries.
- **Test Levels and Types** — Functional, integration, regression, performance, security, usability.
- **Entry and Exit Criteria** — Define what "done" looks like upfront. Without exit criteria, testing never officially ends.
- **Test Approach** — Scripted, exploratory, or hybrid. For unclear-documentation products, charter-based exploratory testing often works better early on.
- **Assumptions and Dependencies** — Document every assumption made due to documentation gaps. This is a critical protective measure.
- **Defect Management and Reporting** — How bugs are logged, prioritised, and communicated.

#### Phase 3: Iteration

A test strategy for a new product is a living document. Revise it weekly as the product evolves and documentation improves.

---

### Q2. If a release is due today and you only have 2 hours left but 50 test cases remaining, how do you decide what NOT to test?

The real question is: **what risk can we afford to ship with, and what risk can we not?**

#### The Mental Framework: Think in Failure Modes

Ask: *If something breaks in production tonight, what breaks the business?* That answer defines your priorities.

Group the 50 test cases into four buckets:

| Bucket | Description | Action |
|---|---|---|
| **Must Test** | Business critical — revenue loss, data corruption, security breach | Test first, completely |
| **Should Test** | High visibility, high frequency user flows | Test next, lighter coverage acceptable |
| **Could Test** | Edge cases, less-used features, cosmetic issues | Likely deferred |
| **Won't Test** | Formally acknowledged and documented | Explicitly communicated to stakeholders |

#### Practical Prioritisation Signals

- **Recent code changes** — anything touched in the last sprint is higher risk
- **Failed tests from previous cycles** — historically buggy areas
- **Integration points** — anything crossing system boundaries
- **User journey criticality** — broken checkout is catastrophic; broken avatar upload is not

#### What You're NOT Doing

You're not skipping tests randomly. You're not testing everything shallowly. You're not staying silent about the gaps. Every decision has a rationale, communicated before the release.

---

### Q3. How do you implement Risk-Based Testing (RBT) and justify it to stakeholders?

#### The Core Concept

RBT prioritises testing effort based on two axes for every feature or component:

- **Likelihood of Failure** — code complexity, developer experience, recency of change, historical defect density
- **Impact of Failure** — financial impact, reputational damage, regulatory exposure, number of users affected, reversibility

#### Building a Risk Register

Score each area on a 1–5 scale for both dimensions. Multiply them to get a **Risk Priority Number (RPN)**.

| Module | Likelihood (1-5) | Impact (1-5) | RPN |
|---|---|---|---|
| Payment Processing | 4 | 5 | 20 |
| User Profile Avatar | 2 | 1 | 2 |

Testing depth is proportional to RPN. High-RPN areas get comprehensive scripted suites, boundary value analysis, and exploratory sessions. Low-RPN areas get a single happy-path check.

#### Justifying to Stakeholders

Speak in business terms — revenue, reputation, users affected:

*"We have identified 47 testable areas. Based on likelihood and impact, 8 areas carry 80% of total risk. We are allocating 70% of our effort to those 8 areas. The remaining 39 areas carry 20% of risk. Here is the risk register so you can review and challenge our assumptions."*

Three things make this stakeholder-proof:
1. You speak in business terms, not technical ones
2. You're transparent rather than making scope decisions in a black box
3. You invite them to challenge risk assumptions — they own the decision jointly

#### Common Pitfalls

- Doing risk assessment once and never revisiting it — risk profiles change every sprint
- Treating RBT as justification to skip testing rather than allocate testing intelligently

---

## 2. Technical Depth (Manual Perspective)

---

### Q4. How do you manually test something with no UI — APIs, background jobs, or data pipelines?

#### Testing APIs Manually

For every API endpoint, ask six questions:

1. **Does it do what it claims?** — verify response body, field names, data types, values
2. **Does it reject what it shouldn't accept?** — send malformed JSON, missing fields, wrong types, empty strings, oversized values
3. **Does it enforce authorisation correctly?** — can a standard user call an admin endpoint? Can User A retrieve User B's data by manipulating an ID? (IDOR vulnerabilities)
4. **Does it behave correctly under sequencing?** — call "cancel order" before "create order"; call "confirm payment" twice
5. **Are headers and status codes correct?** — a 200 with an error message inside is a bug; 401 vs 403 confusion is a security issue
6. **Does pagination, filtering, and sorting work correctly?** — send page=0, request 10,000 records, sort ascending and descending

**Toolset:** Postman, Insomnia, cURL

#### Testing Background Jobs

Background jobs produce side effects — emails sent, records updated, files generated. You test them forensically:

- Capture **pre-state** before triggering the job
- Trigger the job
- Examine **post-state** and compare — the delta should exactly match what the job was supposed to do

Probe: Does it run on schedule or can it be triggered manually? Is it idempotent (what happens if it runs twice)? What if it crashes halfway? What if it processes zero records versus thousands?

**Tools for email jobs:** Mailtrap or Mailhog intercept outgoing emails without spamming real addresses.

#### Testing Data Pipelines

Key techniques:

- **Lineage tracing** — pick a specific record and follow it through every stage
- **Volume testing** — does the pipeline handle 10 records the same as 10 million?
- **Boundary injection** — null values, duplicate keys, maximum-length strings, special characters, timezone edge cases
- **Reconciliation** — row count going in should match row count coming out (adjusted for intentional deduplication)

---

### Q5. Describe your exploratory testing thought process and how it finds bugs scripted tests miss.

#### Why Scripts Miss Things

Scripted test cases are structurally incapable of discovering *unspecified* behaviour. Nobody wrote a requirement that said "concurrent edits should be handled gracefully" — so nobody scripted a test for it. But every user on the platform is a potential concurrent editor.

#### The Thought Framework

Good exploratory testing isn't random clicking. It's structured curiosity guided by **charters**: focused missions with a defined target and time box.

*"Explore the task editing feature with a focus on concurrent user interactions for 45 minutes."*

Within that charter, thinking follows **hypothesis and probe**:
- Form a hypothesis: *"I think concurrent edits might cause a conflict"*
- Design a quick experiment to test it
- Observe the result
- Let the result guide your next hypothesis

Also constantly ask: *What does this system trust that it shouldn't?* It trusts users edit one thing at a time. It trusts API calls arrive in order. Each trust assumption is a test target.

#### Assumptions Inversion

The scripted tests assume a single, well-behaved user doing things in order. Real users don't behave that way. Ask: *What do I believe about this system, and what happens if I'm wrong?*

---

### Q6. How do you verify business logic within large-scale, distributed databases or third-party software integrations manually?

#### The Fundamental Principle

**Never trust the UI to validate data correctness.** The UI shows what the application *thinks* the data is. The database shows what the data *actually is*. These can differ due to caching layers, ORM quirks, eventual consistency delays, and transformation errors.

Approach: **UI assertion followed by direct database verification.**

#### SQL as a Testing Tool

Key query types that matter in testing:

- **Existence checks** — does the record exist with correct values after a UI action?
- **Absence checks** — after deletion, is the record gone or correctly flagged (deleted_at timestamp)?
- **Aggregate validation** — independently calculate totals and compare to stored values
- **Referential integrity** — in distributed systems, verify corresponding records exist across services
- **Audit and history tables** — verify correct user, timestamp, and before/after values

#### Testing Third-Party Integrations

Focus on three boundaries:

1. **What you send** — inspect outbound requests using network interception (Charles Proxy, Fiddler, browser dev tools)
2. **What you receive** — simulate error responses, malformed responses, timeouts, empty responses using WireMock or Mockoon
3. **What you do with it** — verify your system processed the response correctly and stored it against the correct records

#### Handling Eventual Consistency

Use **polling with timeout**: query the target system at short intervals for a defined maximum wait period. Test the *guarantees* the system makes — if it claims "your dashboard updates within 30 seconds," that's a testable assertion.

---

## 3. Process & Collaboration (Agile/DevOps)

---

### Q7. What does Shift-Left Testing mean in a manual context, and how do you involve yourself in the requirement phase?

#### The Core Insight

The later you find a defect, the more it costs to fix. A requirement ambiguity caught in planning costs a five-minute conversation. The same ambiguity caught in production costs developer time, re-testing, and potentially customer impact.

#### What Involvement Looks Like

**Three Amigos Sessions** — Product owner, developer, and tester review a story together before development begins. Your role:
- Developers ask *how* to build something
- Product owners ask *what* to build
- Testers ask *what could go wrong, what's ambiguous, what happens in edge cases*

When a story says "users can reset their password," you ask:
- What if the user has no email address on file?
- What if they click the reset link twice?
- What if the link has already expired?
- What if someone else requests a reset for an account that isn't theirs?
- What happens to active sessions when a password is reset?

**Acceptance Criteria as Test Scenarios**

Help write acceptance criteria that are inherently testable:

*"Given a user with a verified email, when they request a password reset, then they receive a reset email within 60 seconds containing a link valid for 24 hours"* — testable.

*"Users should be able to reset their password"* — an intention, not a criterion.

**Reviewing Designs and Wireframes**

Look for: flows with no error states, forms with no validation shown, navigation paths that lead nowhere, screens that don't account for empty states, extremely long content, or high record counts.

---

### Q8. How do you handle a situation where a developer rejects a bug you believe is critical?

#### When a Developer Says "It Works on My Machine"

This is useful information — it tells you there's an **environment discrepancy**. Find it systematically:

1. **Reproduce it precisely** — document exact steps, data state, environment, and capture evidence (screenshot, screen recording, network logs, console errors)
2. **Compare environments explicitly** — OS, browser and version, build number, test data, user role, feature flags
3. **Request a screen share** — reproduce the issue together in real time
4. **Escalate environment parity as a systemic issue** if discrepancies are common

#### When a Developer Rejects as "Not a Bug"

1. **Assume good faith and seek understanding** — ask why they believe it's not a bug; their reasoning may reveal context you didn't have
2. **Anchor in user impact, not technical opinion** — "A user who hits this scenario loses their entire form submission with no warning, and this flow is triggered 200 times per day" requires a business response
3. **Bring in a third voice** — involve the product owner as a tie-breaker framed as: *"We have a difference of interpretation on expected behaviour"*
4. **Document your position regardless of outcome** — *"QA maintains this scenario represents a risk. Closing as Won't Fix is acknowledged; recommend monitoring post-release"*

---

### Q9. If a critical bug is found in production for a feature you signed off on, what are your first three steps?

#### Step 1: Contain and Communicate Immediately

Alert the right people immediately. Communicate: what the bug is, what users are affected, what the observable impact is, and whether there's a workaround. Don't speculate about cause. Don't minimise impact. Communicate facts clearly and quickly.

#### Step 2: Conduct a Personal Root Cause Analysis

Before the team retrospective, do your own honest investigation:

- Was this scenario in scope for testing? Did I test it?
- If yes — why did the test pass when the bug was present?
- If no — why was it out of scope?
- When was the code that introduced it written?
- Was there a test case that should have caught it?

Output: a clear, honest account of the gap — *"This scenario fell through because our coverage for third-party timeout handling was limited to happy-path responses."*

#### Step 3: Propose Systemic Fixes, Not Just an Additional Test Case

Adding one test case covers the specific bug but doesn't address why the process failed. Identify the **systemic gap**:

- Requirement ambiguity → strengthen acceptance criteria review
- Knowledge concentration → knowledge transfer sessions and documentation
- Inadequate coverage → targeted test design for the scenario category
- Technical debt → make the business case for refactoring using defect data

At the retrospective, present the analysis collaboratively. Frame it: *"What in our process makes it hard to get this right, and what would make it easier?"*

---

## 4. Scenario-Based Challenges

---

### Q10. How do you prevent the Pesticide Paradox in long-term projects?

The Pesticide Paradox: just as insects develop resistance to pesticides used repeatedly, software develops "resistance" to test cases run repeatedly. The same tests find fewer bugs over time — not because software is getting better, but because tests only look where they've already looked.

#### Strategy 1: Scheduled Test Case Retirement and Renewal

Quarterly review of existing test cases — not for correctness but for **continued relevance**:
- Is this area still active and unchanged?
- Is this scenario still a realistic user path?
- Has the underlying business rule changed?

Retire irrelevant tests. Redirect freed capacity to newer, higher-risk areas.

#### Strategy 2: Rotating Testers Across Modules

A fresh tester brings fresh assumptions and generates fresh test ideas. Build deliberate rotation into testing assignments. When rotating onto a new module, spend time in exploratory testing before running existing scripts.

#### Strategy 3: Exploratory Testing as a Formal Complement

Schedule formal exploratory sessions — time-boxed charters — as a recurring activity, not an ad hoc one. On long-running projects, the ratio of exploratory to scripted testing should shift over time as scripted tests stabilise.

#### Strategy 4: Using Defect History to Redirect Focus

- High historical defect density → warrants deeper, more varied testing
- Zero historical defects → could mean genuinely stable, or tests aren't good enough to find what's there; apply exploratory testing to verify

#### Strategy 5: Introducing Variability Into Existing Tests

Where you can't replace tests, vary them — use different test data combinations, different character sets, different value lengths. Vary execution sequence — order of operations sometimes matters in ways original test design didn't anticipate.

---

### Q11. How do you approach regression testing for a massive legacy system where original developers and testers are no longer with the company?

#### Phase 1: Archaeological Investigation

- **Existing test artefacts** — find every test case, plan, or script regardless of age; evaluate for continued relevance before running
- **Defect history** — which modules generated the most defects? Which integration points failed repeatedly?
- **Release notes and change logs** — cross-reference with defect history to build a risk map
- **Code history** — commit frequency by module reveals which areas are sensitive to change
- **Support and incident tickets** — real-world failure modes that scripted tests missed

#### Phase 2: Building a Functional Coverage Map

Create a structured inventory of every feature, workflow, and integration point. For each area assess: How critical is it? How frequently used? How often does the code change? Historical defect rate?

#### Phase 3: Identifying Critical Paths

Focus first on end-to-end flows representing core system value. Interview current users: *"If this system broke in a way that stopped your work for a day, what would that look like?"*

#### Phase 4: Building the Regression Suite Incrementally

Don't try to build a complete suite before testing starts. Build it sprint by sprint, change by change. Convert every production defect immediately into a regression test case.

#### Phase 5: Managing Uncertainty Honestly

Maintain a living document mapping test coverage against the functional map — showing not just what you test but explicitly what you don't test and why. Communicate coverage position to stakeholders before each release.

---

### Q12. When you see a pattern of repeated defects in the same module, how do you conduct a root cause analysis?

#### The Mindset Shift

Ask not *"what went wrong in the code?"* but *"what went wrong in the process that allowed this to be written and released without being caught?"*

#### Step 1: Aggregate and Pattern-Analyse the Defect Data

For each defect note: type, when introduced, when found, what triggered it. Look for patterns — are defects concentrated after a specific type of change? Found predominantly in production rather than testing? Concentrated in a specific functional area?

#### Step 2: Apply the 5 Whys

*The payment calculation module has produced five defects in three months.*
- Why? → Edge cases in discount stacking logic are consistently incorrect
- Why? → The discount stacking rules are complex and code is difficult to understand
- Why? → Business rules were never formally documented — they exist in scattered emails
- Why? → Requirement documentation for complex business rules was always treated as informal
- Why? → The team never established a standard for formalising complex business logic documentation

Root cause: absence of formalised documentation — not the code itself.

#### Step 3: Categorise the Root Cause

| Category | Fix |
|---|---|
| Requirement ambiguity | Strengthen acceptance criteria and Three Amigos review |
| Knowledge concentration | Knowledge transfer, documentation, pair programming |
| Inadequate test coverage | Targeted test design for the specific scenario category |
| Complexity without safeguards | Module-specific review and testing requirements |
| Technical debt | Business case for refactoring using defect frequency data |

#### Step 4: Conduct Analysis Collaboratively

Bring aggregated data to a retrospective. Present the pattern and invite the team to challenge and enrich it. Frame it: *"What in our process makes it hard to get this module right, and what would make it easier?"*

#### Step 5: Measure Whether the Fix Worked

Track defect rates in the affected module for three to six months after implementing the process fix. Did frequency decrease? Did the specific recurring defect type disappear?

---

## 5. Automation — Tool Selection & ROI

---

### Q13. How do you identify which test cases to automate?

Think of automation candidacy as a filter. Every test case passes through a series of questions.

#### Filter 1: Repetition Frequency

How often does this test need to execute? Tests running on every commit, every deployment, or multiple times per day are strong candidates. Tests run once per quarter are weak candidates.

#### Filter 2: Stability of the Feature

Don't automate something that has changed significantly in the last two to three sprints. Automation should follow stability, not chase it.

#### Filter 3: Clarity and Determinism of Expected Outcome

Tests where pass/fail is clear-cut are strong candidates. Tests requiring human judgment — usability, visual design, exploratory investigation — cannot be meaningfully automated.

#### Filter 4: Risk and Business Criticality

High-risk, business-critical flows warrant automation even if they run only once per sprint, because the cost of a regression in production is so high that the insurance value is significant.

#### Filter 5: Human Execution Cost

Complex test cases requiring multi-system coordination, specific data states, and cross-browser execution have higher automation ROI because the manual execution cost (denominator) is larger.

#### Filter 6: The Automation Paradox Check

If a test case has passed without a single failure in 18 months — is the feature genuinely stable, or is the test case not good enough to find bugs that might exist?

#### Scoring Model

Score each candidate 1–3 on each dimension and sum:

| Dimension | 3 | 2 | 1 |
|---|---|---|---|
| Execution frequency | Many times/year | A few times | Rarely |
| Feature stability | Stable 6+ months | Mostly stable | Frequently changing |
| Manual execution cost | Complex, time-consuming | Moderate | Quick |
| Business criticality | Mission critical | Important | Low impact |
| Outcome determinism | Clear binary pass/fail | Mostly clear | Requires judgment |

- **12–15:** Strong automation investment
- **8–11:** Conditional — examine weak dimensions
- **Below 8:** Poor automation candidate

#### What NOT to Automate

- One-time or infrequent tests
- Highly volatile features
- Tests requiring complex, unstable test data
- Tests requiring human judgment

---

### Q14. How do you identify the right automation tool?

Start with your context, let the tool follow — never the other way around.

#### Step 1: Understand Your System Under Test

- What layers need automation? (unit, component, integration, API, UI, end-to-end)
- What technology stack does the application use?
- What are the interface types? (REST API, GraphQL, message queue, database, web UI, mobile)
- What are the non-functional requirements? (performance, security)

#### Step 2: Evaluate Against Structured Criteria

- **Technical Fit** — supports your languages, frameworks, interfaces, and CI/CD pipeline
- **Team Skill Alignment** — the best tool is often the one your team can actually master and maintain
- **Community and Support** — large active communities mean faster problem resolution
- **Maintenance Overhead** — some tools are inherently more brittle; factor maintenance burden in
- **Licensing and Cost** — open-source vs commercial; both have real costs
- **Parallel Execution and Scalability** — can it run tests simultaneously across environments?
- **Reporting and Integration** — does it integrate with Jira, Azure DevOps, and your CI/CD dashboard?

#### Step 3: Run a Proof of Concept

Two to four week time-boxed PoC covering: one complex scenario end-to-end, CI/CD integration, simulation of maintenance when something breaks, honest assessment of actual time versus expected time.

#### Tool Landscape by Layer

| Layer | Tools |
|---|---|
| UI / End-to-End | Playwright, Cypress, Selenium WebDriver |
| API Testing | REST Assured, Karate, Postman/Newman |
| Messaging (Solace) | Native client libraries wrapped in TestNG/JUnit |
| Database | JDBC-based frameworks, DbUnit, direct SQL execution |
| Performance | JMeter, Gatling, k6 |
| Contract Testing | Pact |

---

### Q15. How do you calculate automation ROI?

#### The Full Cost Picture

- **Licence Cost** — annual or per-seat fees for commercial tools
- **Setup and Infrastructure** — CI/CD configuration, cloud execution environments, containerisation
- **Initial Framework Development** — building the framework itself before any business test is written (often 4–8 weeks)
- **Test Development Cost** — writing, debugging, and stabilising each test case (a 20-minute manual test may take 6–10 hours to automate)
- **Maintenance Cost** — UI suites: 25–30% of development cost per year; API suites: 10–15%
- **Training Cost** — upskilling team members on the tool and framework

#### The Full Benefit Picture

- **Manual Execution Time Saved** — manual execution time × annual run frequency × hourly cost
- **Frequency Uplift Value** — automation enables more frequent testing; earlier defect detection has quantifiable value
- **Parallel Execution Saving** — running across 5 browsers simultaneously vs sequentially
- **Defect Prevention Value** — estimated value of production incidents avoided
- **Release Velocity Improvement** — faster regression cycles enable faster releases

#### Worked Example

80 API test cases, 15 minutes each to execute manually, running 36 times per year, resource cost £45/hour:

| Item | Calculation | Cost/Value |
|---|---|---|
| Annual manual execution | 80 × 15min × 36 runs = 720hrs × £45 | £32,400/year |
| Framework build | 6 weeks × 40hrs × £45 | £10,800 one-time |
| Test development | 80 tests × 7hrs × £45 | £25,200 one-time |
| Annual maintenance | 15% of development | £3,780/year |
| Infrastructure | — | £1,200/year |
| **Total Year 1 cost** | | **£41,000** |
| **Total Year 2 cost** | | **£4,980** |
| **Break-even** | | **Mid Year 2** |

Adding frequency uplift (200 deployments vs 36 manual runs) and prevented production incidents (2–3 per year at £8,000–£15,000 each) typically pulls break-even into Year 1.

#### Presenting ROI to Stakeholders

- Speak in business outcomes: *"We will reduce regression cycle from 3 days to 4 hours"*
- Be honest about Year 1 — automation almost never delivers positive ROI in Year 1
- Show the compounding curve — Years 2, 3, and 4 is where automation's real value is realised

---

## 6. Test Strategy vs Test Plan

---

### Q16. What is the difference between a Test Strategy and a Test Plan?

#### The One-Line Distinction

- **Test Strategy** answers: *"How do we approach testing?"*
- **Test Plan** answers: *"How do we execute testing for this specific release?"*

The strategy is the philosophy. The plan is the schedule.

#### Test Strategy

A high-level, long-lived document defining your overall approach to quality. It doesn't change sprint to sprint. Think of it as your **testing constitution**.

**Contains:** Testing scope and objectives, testing types and levels, risk-based approach, tools and environments, roles and responsibilities, defect management philosophy, entry and exit criteria definitions, metrics and reporting approach.

**Who writes it:** QA Lead / Test Manager
**Lifespan:** Months to years

#### Test Plan

A specific, time-bound document tied to a particular release, sprint, or project phase. Think of it as your **mission brief**.

**Contains:** Scope for this specific release, test cases or scenarios to be executed, specific entry and exit criteria, resource allocation, timeline and milestones, test environment details, known risks and dependencies, sign-off requirements.

**Who writes it:** QA Engineer / QA Lead
**Lifespan:** Days to weeks

#### Side-by-Side Comparison

| Dimension | Test Strategy | Test Plan |
|---|---|---|
| Question answered | How do we approach testing? | How do we execute this cycle? |
| Scope | Product or programme level | Release or sprint level |
| Lifespan | Long-lived (months/years) | Short-lived (days/weeks) |
| Specificity | Broad principles | Specific tasks and timelines |
| Author | QA Lead / Test Manager | QA Engineer / QA Lead |
| Audience | Stakeholders, whole team | Execution team |
| Changes when | Process fundamentally shifts | Every release |
| Contains test cases? | Never | Yes |
| Contains timelines? | No | Always |

#### The Analogy

A military campaign: your **strategy** says we will fight a guerrilla war, prioritise supply line disruption, measure success by territory held over six months. Your **plan** for Tuesday's operation says Unit A moves to grid X at 0600, the objective is the communications tower, abort if resistance exceeds Y. The plan must be consistent with the strategy — but the strategy never gets into Tuesday's specifics.

---

## 7. Test Data Management

---

### Q17. How do you manage test data in general?

**Core philosophy:** Test data is not a precondition for testing — it is part of testing. The discipline you apply to designing test cases should be applied equally to designing the data those test cases run against.

#### Production-Mirrored Data

**Strengths:** Reflects real-world complexity, edge cases, and volume characteristics synthetic data rarely replicates.

**Weaknesses:** Contains sensitive information requiring careful masking. Masking introduces risk. Reflects current production state, which may not include scenarios needed for new feature testing.

**When to use:** Performance testing, complex business rule regression, production incident reproduction.

#### Synthetic Data

**Strengths:** Fully controlled, no sensitive information exposure, can be generated on demand, covers scenarios that don't yet exist in production.

**Weaknesses:** Only as good as the thinking that created it. Rarely replicates the messy combinations real users generate.

**When to use:** Functional testing of new features, edge case and boundary testing, security and negative testing, any scenario involving sensitive data categories.

#### Combination Approach

Use **synthetic data as default** for functional and regression testing. Supplement with **production-mirrored data** where real-world complexity genuinely matters — performance testing, complex integration scenarios, incident reproduction.

#### Handling Sensitive Data

- **Data Masking** — replace sensitive values with realistic but fictitious equivalents; masked data must preserve format and structure
- **Data Subsetting** — extract a representative subset rather than copying entire databases
- **Tokenisation** — replace high-sensitivity fields with tokens that map back only in production
- **Synthetic Generation** — for the most sensitive categories, generate from scratch using tools like Faker or Mockaroo

#### Test Data Consistency Across Distributed Systems

Strategies:

- **Centralised Test Data Store** — single source of truth that all services reference
- **Data Setup as Part of Test Execution** — each test creates its own data and tears it down; most reliable approach
- **Contract-Based Data Agreements** — data created to satisfy service contracts
- **Environment Refresh Cadence** — daily refresh for active environments, weekly for less active
- **Service Virtualisation** — stub upstream dependencies with controlled responses
- **Automated Data Validation Checks** — run integrity checks before each test session

#### When Test Data Gets Corrupted Mid-Cycle

**Immediate response:**
1. Identify scope of corruption before taking action
2. Suspend testing in affected areas — continuing produces unreliable results
3. Do not manually patch corrupted data unless absolutely necessary

**Recovery options (in order of preference):**
1. **Snapshot and restore** — restore to last known good snapshot
2. **Scripted data restoration** — rebuild from clean baseline using version-controlled scripts
3. **Partial restoration with impact assessment** — restore specific affected areas, document what remains uncertain

**Prevention:**
- Test data isolation — each test operates on its own data set
- Read-only shared reference data — locked from test modification
- Transactional test execution — wrap tests in transactions that roll back after completion
- Automated data validation checks at the start of each CI pipeline run

#### Test Data Ownership

- **QA owns the strategy** — approach, standards, tooling, governance
- **Individual testers own their test data** — creating, validating, cleaning up
- **Shared baseline is collectively owned** — changes require communication and agreement
- **Test data debt is tracked** — stale records and orphaned data sets are periodically cleaned

---

## 8. Behavioural & Situational Questions

---

### Q18. Tell me about a time you had a disagreement with your manager.

#### What the Panel Is Assessing

- Can you form and defend an independent professional opinion?
- Do you handle it through dialogue, not escalation or passive aggression?
- Do you know when to concede and when to hold firm?
- Can you separate the disagreement from the relationship?

#### Sample Answer

*"There was a situation during a release cycle where a release was approaching and my test manager wanted to sign off and move forward. I had concerns about resiliency testing not being complete — specifically, we hadn't validated system recovery behaviour under a messaging failure scenario, which I considered high risk given the 24x7 nature of the system.*

*My manager's position was that the deadline was fixed, the business had committed to the release date, and the scenario I was concerned about was edge-case enough to accept as a risk.*

*Rather than just flagging the concern verbally, I put together a short written summary — the specific scenario, the potential failure mode, the estimated time to complete the test, and what a partial mitigation might look like. I kept it factual and framed it as a risk communication rather than a challenge to the decision.*

*Once my manager saw the failure impact documented — a silent data loss scenario that wouldn't surface immediately but would corrupt downstream data — he agreed the risk wasn't acceptable to ship with.*

*We negotiated a middle ground: delayed the release by one day, ran a targeted test covering the specific scenario, it passed, and we released with confidence.*

*What I took from it was that disagreeing upward works best when you come with evidence and a solution, not just a concern. And that my manager wasn't wrong to push for the deadline — he was balancing constraints I didn't have full visibility into. The conversation worked because we both stayed focused on the right outcome rather than our respective positions."*

---

### Q19. How do you handle escalation when it is directed at you?

#### What the Panel Is Assessing

- Do you get defensive or do you get curious?
- Do you own your decisions?
- Do you separate the escalation from your ego?
- Do you resolve it or just manage the noise?

#### Core Philosophy

*"My first response to any escalation against me is to listen fully before responding. My instinct isn't to defend — it's to understand. Is the concern valid? Is there something I missed? Is there a communication gap that created a wrong expectation? The answer to those questions determines everything that follows."*

#### The Framework

**Step 1: Receive It Without Reacting**
Don't interrupt, don't immediately justify, don't minimise. Let the person finish. Your composure in that moment is itself a signal to everyone watching.

**Step 2: Acknowledge Before You Respond**
*"I understand why this has been raised and I take it seriously. Let me look at this properly and come back to you with a full picture."*
This isn't capitulation — it's professionalism.

**Step 3: Honest Self-Assessment**
Is the concern valid? Did I make an error? Was there a communication gap? A process failure?
- If yes — own it immediately, completely, and without hedging
- If no — respond with clarity and evidence, not emotion

**Step 4: Respond With Facts, Not Feelings**
*"Here is what the situation was. Here is the information I had at the time. Here is the decision I made and the reasoning behind it."*

**Step 5: Propose the Path Forward**
End with a forward-looking proposal. What are you going to do about it? What changes will prevent recurrence? This shifts the conversation from what went wrong to what happens next.

#### What You Never Do

- Never throw someone else under the bus
- Never minimise the concern to make it go away
- Never over-apologise without substance
- Never let it make you risk-averse in subsequent decisions

#### The One-Line Answer

*"When I'm escalated against, my first move is to listen and understand before I respond, my second is to be honest about whether the concern is valid, and my third is to focus the conversation on what happens next — not on defending what already happened."*

---

### Q20. What are your strengths and weaknesses?

#### Strengths

**Risk Thinking in a High-Stakes Domain**
*"One of my core strengths is risk-based thinking — developed working on a system where a missed defect isn't just a quality issue, it's a financial one. I've developed a strong instinct for where risk actually lives — not just where the most code is, but where the consequences of failure are highest."*

**Bridging Manual Depth with Automation Judgment**
*"My strongest technical differentiator is that I don't approach automation as a separate discipline from testing — I approach it as an extension of good testing thinking. My strength is knowing where that boundary sits — what deserves automation, at which layer, and how to build it so it stays maintainable."*

**Testing Distributed and Messaging-Driven Systems**
*"In distributed systems and messaging architectures, I've built strong capability in areas many testers find difficult: resiliency testing, recovery validation, contract testing, and messaging automation."*

#### Weakness

*"A weakness I've recognised in myself is that under release pressure, my instinct is to want to test more rather than make a clean scope decision and own it. I find it genuinely uncomfortable to sign off when I know there are areas I haven't fully covered, even when the risk is objectively low.*

*I've worked on this by getting more disciplined about risk quantification — forcing myself to document residual risk explicitly and communicate it, rather than trying to eliminate it through more testing time. That shift from 'test everything' to 'own the risk decision' has made me more effective and more trusted by the teams I work with."*

---

## 9. Interview Introduction

---

### Opening Statement

*"I have over 8 years of experience in quality engineering, working across the full testing spectrum — from early-stage component validation through to system-wide integration and release. Currently, I work on RPE — a Real-time FX Pricing Engine — a mission-critical, distributed system built for 24x7 availability with high resilience and high throughput capacity. In FX markets, there is zero tolerance for downtime or mispriced data, which means every testing decision carries direct business and financial risk.*

*The system is distributed in nature and communicates across services through Solace, an enterprise messaging platform — which means we're not just validating application logic, but also message delivery, ordering, and system behaviour under failure conditions.*

*My approach to testing this kind of system is structured across multiple layers. It starts at the component level — validating individual services and modules in isolation to ensure core logic is sound before any integration begins. From there, I move into component integration testing, where I verify that services interact correctly with each other — catching interface mismatches and contract violations early, before they compound into larger system issues. We also practice contract testing, ensuring that each service honours the agreements it has with its consumers and providers, which is particularly critical in a distributed, messaging-driven architecture like ours.*

*Once the integration layer is stable, we conduct System Integration Testing end-to-end — validating complete business flows across all components, databases, and messaging layers working together as a whole.*

*Alongside this structured progression, I apply risk-based testing to continuously prioritise coverage where business impact is highest, and exploratory testing to surface edge cases and failure modes that scripted test cases don't anticipate. Given the resilience requirements of the system, resiliency and recovery testing is also a core part of what I do — deliberately simulating failures to confirm the system degrades gracefully and recovers correctly.*

*On the automation side, I follow a deliberate triage process — starting with a well-designed manual suite, then splitting it into what can and cannot be automated, and from the automatable set, prioritising what is business-critical. That drives our backend automation strategy, which spans API-level testing, database validation, and Solace messaging automation — giving us deep, reliable coverage at every layer of the stack.*

*What I bring to an SDET role is that combination of structured testing methodology — knowing what to test, at which layer, and why — paired with the engineering discipline to automate it intelligently and build something the team can trust and maintain over time."*

---

## 10. Questions to Ask the Panel

---

### Question 1: Onboarding and Knowledge Transfer

*"If I were to join this team, what would you recommend I familiarise myself with during my notice period — whether that's domain knowledge, tooling, or architecture concepts — so that the initial knowledge transfer is more productive for both sides?"*

**Why it works:** Signals you're already thinking about contributing from day one. Demonstrates self-driven professionalism. Respects the team's time during KT.

---

### Question 2: AI in Testing

*"AI in testing is something the industry is actively exploring right now — from test generation to anomaly detection to intelligent coverage analysis. I'm curious where this team stands on it — are you actively experimenting with any AI-assisted approaches, and if so, what's been the most tangible impact you've seen so far?"*

**Why it works:** Shows you're thinking beyond current practices. Demonstrates awareness of the broader industry direction. Opens a richer conversation rather than inviting a yes/no answer.

---

### Question 3: Current Challenges (Optional — use if time and energy allow)

*"What's the most complex testing challenge this team is currently facing that hasn't been fully solved yet?"*

**Why it works:** Gives genuine insight into what you're walking into. Positions you as someone interested in solving hard problems rather than just filling a seat.

---

### Sequencing

1. Lead with the **onboarding question** — personal, practical, sets a collaborative tone
2. Follow with the **AI question** — strategic and forward-looking, strong closing note
3. Use the **challenge question** only if time allows and the room invites it

---

## Entry and Exit Criteria Reference

---

### Entry Criteria (When to Start Testing)

| Category | Criteria |
|---|---|
| Build readiness | Successfully deployed, stable, no critical installation errors |
| Smoke test | Basic smoke test passed confirming fundamental functionality |
| Code completeness | Feature fully developed, not partially implemented |
| Test artefacts | Test cases written, reviewed, approved; test data prepared |
| Requirement sign-off | Acceptance criteria agreed and stable |
| Defect baseline | Open critical/high defects from previous cycle within acceptable threshold |

### Exit Criteria (When to Stop Testing)

| Category | Criteria |
|---|---|
| Test execution | 95–100% of planned test cases executed |
| Pass rate | Minimum pass rate threshold met (defined upfront) |
| Critical/high defects | Zero open, or formally risk-accepted with documentation |
| Regression suite | Automated suite passed at defined stability threshold |
| Non-functional | Performance, security, resiliency benchmarks met |
| Test summary report | Produced covering scope, results, defects, coverage, risks |
| Stakeholder sign-off | QA lead, product owner, release manager formally signed off |

### Entry/Exit Criteria Across Testing Phases

| Phase | Entry Criteria | Exit Criteria |
|---|---|---|
| Component Testing | Code complete, unit tests passing | All component tests executed, no critical defects |
| Component Integration | Component testing exit met, integration environment ready | Interface contracts verified, integration defects resolved |
| SIT | Integration testing exit met, full system environment stable | End-to-end flows validated, performance benchmarks met |
| UAT | SIT exit met, business scenarios documented | Business stakeholders satisfied, go/no-go decision made |
| Production Release | UAT exit met, release plan approved | Deployment verified, smoke test passed, monitoring active |

---

*Document prepared for SDET interview preparation — all scenarios based on real-world QA practices in distributed, high-availability systems.*
