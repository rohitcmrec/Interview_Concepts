# Process & Collaboration (Agile/DevOps)



## 1. Shift-Left Testing: Getting Involved Before a Line of Code is Written

The traditional testing model looks like a waterfall: requirements flow to development, development flows to testing, testing flows to release. Testing sits at the right end of the timeline, acting as a filter before production. Shift-Left is the recognition that **this model is expensive, slow, and structurally backwards**.

The core insight is simple: the later you find a defect, the more it costs to fix — in time, in rework, in team friction, and sometimes in customer impact. A requirement ambiguity caught in a planning meeting costs a five-minute conversation. The same ambiguity caught in testing costs a bug report, a developer context-switch, a fix, a re-test, and potentially a delayed release. The same ambiguity caught in production costs all of that plus customer damage and incident management.

Shift-Left means moving your involvement — and therefore your defect-finding — as far left on that timeline as possible.

### What This Looks Like in a Manual Testing Context

In a scripted-testing-only world, testers receive a finished build and execute pre-written test cases against it. In a Shift-Left world, by the time a build arrives in your hands, you've already been involved for days or weeks. Here's what that involvement actually looks like:

**Requirement Reviews and Three Amigos Sessions**

The Three Amigos model — product owner, developer, and tester reviewing a story together before development begins — is the most practical implementation of Shift-Left that exists. Your role in this meeting is not to validate that requirements are complete. Your role is to **ask the questions nobody else is asking**.

Developers tend to ask *how* to build something. Product owners tend to ask *what* to build. Testers ask *what could go wrong, what's ambiguous, and what happens in the edge cases nobody thought about*.

When a story says "users can reset their password," you're the person asking: What if the user doesn't have an email address on file? What if they click the reset link twice? What if the link has already expired — what do they see? What if someone else requests a reset for an account that isn't theirs? What happens to active sessions when a password is reset? Each of these questions is either clarifying a real requirement or identifying a scenario that needs a decision before development starts.

These aren't test cases yet. They're **requirement gaps**, and surfacing them now means the developer builds the right thing the first time.

**Acceptance Criteria as Test Scenarios**

One of the most valuable contributions a tester can make during the requirement phase is helping write acceptance criteria in a way that is inherently testable. Vague acceptance criteria produce vague development and vague testing. Precise acceptance criteria produce precise development and clear pass/fail conditions.

"Given a user with a verified email, when they request a password reset, then they receive a reset email within 60 seconds containing a link valid for 24 hours" — that's a testable statement. "Users should be able to reset their password" — that's an intention, not a criterion.

When you help craft acceptance criteria this way, you're not just improving the requirement. You're already writing the skeleton of your test scenarios before development has started.

**Reviewing Designs and Wireframes**

UI designs and wireframes are requirements in visual form. They contain the same gaps and assumptions that written requirements do. When you review a wireframe, you're looking for flows that have no error states, forms that have no validation shown, navigation paths that lead nowhere, and states the design never accounts for — what does this screen look like when the data is empty? When there are 10,000 records? When the user's name is 200 characters long?

Finding these in a wireframe review is a 10-minute conversation with a designer. Finding them in a finished build is a bug report, a design revision, a development rework cycle, and potentially a missed sprint.

### The Cultural Shift Required

Shift-Left only works if testers are welcomed into early conversations rather than treated as a downstream function. That welcome doesn't always come automatically — you sometimes have to earn it by demonstrating that your early involvement prevents problems rather than creating overhead.

The way you earn it is by showing up to planning meetings with prepared questions, by turning those questions into documented decisions, and by making it visible when an early question prevented a late defect. Over time, teams that have experienced this stop thinking of testing as a phase and start thinking of quality as a continuous responsibility shared from day one.



## 2. Developer Conflicts: "It Works on My Machine"

This is one of the most common and most emotionally charged situations in a QA-developer relationship. Handled badly, it creates a culture of adversarial friction. Handled well, it builds credibility and mutual respect.

The first thing to internalise is this: **a developer rejecting your bug is not a personal attack on your testing**. It is either a genuine disagreement about severity, a legitimate environment discrepancy, or a misunderstanding of expected behaviour. Your job is to resolve which one it is — not to win an argument.

### When a Developer Says "It Works on My Machine"

This phrase, frustrating as it is, is actually useful information. It tells you there is an **environment discrepancy** somewhere. Your job is to find it, because it's almost certainly real.

The systematic approach looks like this:

**Reproduce it yourself, precisely.** Before any conversation with a developer, make sure you can reproduce the bug consistently with a documented, step-by-step path. "The page sometimes breaks" is not a reproducible bug. "When logged in as a non-admin user with an expired subscription, clicking Settings > Billing on a mobile viewport produces a 403 error with no user-facing message" is reproducible. Document the exact steps, the exact data state, the exact environment, and capture evidence — screenshot, screen recording, network logs, console errors.

**Compare environments explicitly.** Create a simple side-by-side comparison: your environment versus the developer's. Operating system, browser and version, application version/build number, test data used, user role and permissions, feature flags enabled. Nine times out of ten, the discrepancy lives in one of these variables. Maybe the developer is testing on Chrome and you're on Safari. Maybe you're using a test account with a specific data configuration the developer's account doesn't replicate. Find the variable and you've either confirmed the bug or explained the discrepancy — both are valuable outcomes.

**Request a screen share.** Rather than going back and forth asynchronously, suggest you reproduce the issue together in real time. Walk the developer through your exact steps on your machine while they watch. This eliminates interpretation errors and is almost always faster than a ticket comment chain.

**Escalate environment parity as a systemic issue.** If environment discrepancies are common, the problem isn't the individual bug disagreement — it's that your test environments aren't consistent enough. That's a process issue worth raising, not as a complaint, but as a risk to release quality.

### When a Developer Rejects a Bug as "Not a Bug" or "Won't Fix"

This is a different situation. Here the disagreement is about severity or validity, not reproducibility.

**Start by assuming good faith and seeking understanding.** Ask the developer why they believe it's not a bug or why it's lower priority than you assessed. Sometimes their reasoning reveals context you didn't have — maybe the behaviour is intentional, maybe there's a workaround in the design you weren't aware of, maybe a similar issue was previously reviewed and accepted as a known limitation. Understanding their reasoning either changes your view or helps you make a stronger counter-argument.

**Anchor the conversation in user impact, not technical opinion.** "I think this is critical" is an assertion that invites counter-assertion. "A user who hits this scenario loses their entire form submission with no warning or recovery path, and our support logs show this flow is triggered approximately 200 times per day" is a business impact statement that requires a business response. Translate the technical defect into user and business language whenever there's disagreement.

**Bring in a third voice when genuinely stuck.** If you and a developer have an honest disagreement about severity that can't be resolved between you, the right move is not to escalate as a conflict — it's to bring the product owner or business analyst in as a tie-breaker. Frame it as: *"We have a difference of interpretation on expected behaviour and would like your view on the intended user experience."* This keeps it professional, focuses on the right question (what was intended), and prevents the disagreement from becoming personal.

**Document your position regardless of the outcome.** If a bug is closed as "Won't Fix" over your objection, note your disagreement in the ticket. Not aggressively — simply and factually. *"QA maintains this scenario represents a risk to users in X situation. Closing as Won't Fix is acknowledged; recommend monitoring post-release."* This isn't about being right. It's about creating a record that protects both you and the team if the issue surfaces in production later.

### The Relationship Underneath the Conflict

The developers who trust their testers most are the ones who've experienced testers being fair, precise, and right consistently over time. Every interaction you have in a bug disagreement is either building or eroding that trust. Being meticulous about reproduction steps, honest when you've made a mistake, and collaborative rather than adversarial in disagreements compounds over time into a relationship where your critical bug reports are taken seriously immediately — because your track record warrants it.



## 3. Defect Leakage: A Critical Bug Reaches Production That You Signed Off On

This is the scenario that most testers dread and most teams handle badly. The wrong response is defensiveness, deflection, or silence. The right response is structured, professional, and genuinely investigative.

### Step One: Contain and Communicate Immediately

The first priority is not explanation — it's damage control. A critical bug in production is an active incident, and your first action is to ensure the right people know immediately. Depending on your organisation, this might mean alerting your lead, the product owner, the on-call engineer, or triggering your incident response process.

Communicate clearly: what the bug is, what users are affected, what the observable impact is, and whether you have any initial sense of a workaround. Don't speculate about cause yet — you don't know enough. Don't minimise the impact to reduce pressure — that destroys trust. Just communicate the facts as clearly and quickly as possible so the people who can mitigate the damage can start doing so.

Your role in this phase is to support the resolution — which might mean helping reproduce the bug in production to confirm scope, testing any emergency hotfix as fast as possible, or helping identify whether a rollback is safe.

### Step Two: Conduct a Personal Root Cause Analysis Before the Team Retrospective

Before the team retrospective or formal post-mortem, do your own honest investigation. This is not about assigning blame — it's about genuinely understanding how the bug moved through the process undetected.

Ask yourself: Was this scenario in scope for testing, and did I test it? If yes, why did the test pass when the bug was present — was there an environment difference, a data difference, or did I miss something in my analysis? If no, why was it out of scope — was it a deliberate risk decision, a coverage gap I didn't identify, or a scenario that emerged from a combination of factors no individual requirement anticipated?

Trace the bug's history: when was the code that introduced it written? Was there a requirement that covered this behaviour? Was there a test case that should have caught it? Did the test case exist and pass incorrectly, or did the test case never exist? Each of these questions leads to a different systemic fix.

The output of this analysis should be a clear, honest account of the gap: *"This scenario fell through because our test coverage for third-party timeout handling was limited to happy-path responses, and we had no test cases for partial response scenarios."* That's actionable. *"I don't know how I missed it"* is not.

### Step Three: Propose Systemic Fixes, Not Just an Additional Test Case

The least useful response to a production defect is adding one test case to cover the exact scenario that failed. That's closing the barn door after the horse has left — it prevents the same specific bug from reoccurring, but it doesn't address why the process failed to catch it in the first place.

The most useful response is identifying the **systemic gap** and proposing a fix at that level.

If the bug existed because a specific integration scenario was never tested, the fix might be creating a formal integration testing checklist for all third-party connections. If the bug existed because an edge case data state was never replicated in the test environment, the fix might be improving test data management so that production-representative data states are always available. If the bug existed because a requirement was ambiguous and you made an incorrect assumption, the fix might be strengthening acceptance criteria review in the Three Amigos process.

At the retrospective, come with this analysis and these proposals. Don't be defensive, don't over-apologise, and don't be silent. The worst thing a tester can do after a production defect is shrink. Your team needs you to be the person who understands what happened and is driving the improvements that prevent recurrence — not the person who is either making excuses or flagellating themselves.

### The Harder Truth About Defect Leakage

Production defects are almost never purely a testing failure. They're a **process failure** — and the process involves requirements, development, testing, and release management collectively. A critical bug that reaches production usually had multiple opportunities to be caught and wasn't, which means multiple process steps failed, not just one tester.

Owning your piece of that honestly — without owning the entire failure alone — is the mark of a mature QA professional. You're accountable for the coverage decisions you made and the gaps in your testing approach. You're not solely accountable for every defect that exists in a system you tested. The distinction matters, and framing it correctly in a retrospective protects both your professional standing and the team's ability to have an honest conversation about what actually needs to change.
