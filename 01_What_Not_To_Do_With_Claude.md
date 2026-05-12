# 01 — What Not to Do With Claude?
## How to Actually Think About Claude (and Why Most Developers Get It Wrong)

> **Series:** How Architects Use Claude  
> **Level:** Senior / Staff / Architect  
> **Core idea:** Claude is not a junior dev, not an oracle, not autocomplete. It is a literal contractor. Treat it accordingly.

---

## The Story

It is a Friday afternoon. Rahul is closing out a sprint. There is one ticket left — a "small" feature to add a new endpoint to the orders service. He estimates 4 hours. He opens Claude Code.

> *"Add a `POST /orders/refund` endpoint to the orders service that processes refunds. Accept an `order_id` and `amount`. Validate the user owns the order. Process the refund via the payment gateway."*

Claude responds in 12 seconds with a complete implementation: controller, service, repository update, payment gateway call, transaction handling, retries, an idempotency key, structured logging, three unit tests. It is 340 lines of code. It compiles. The tests pass.

Rahul scans the diff. The code looks clean. Naming matches the codebase. Idioms look familiar. He merges. CI passes. He deploys to staging. Smoke test works. He pushes to production at 5:30 PM and goes home.

At 11:47 PM, the on-call engineer is paged. The payments dashboard is on fire.

```
ALERT  payment_gateway_error_rate > 40%
       Affected service: orders-service
       Onset: 19:32 UTC
```

Customers are getting double-refunded. Some are getting refunded for orders they didn't own. One customer got refunded $0.00 and then the gateway flagged the account for fraud.

Three things went wrong. None of them are visible in the code.

**Bug 1.** The "idempotency key" was generated as `UUID.randomUUID()` inside the controller. Every retry from the client created a new key. Claude wrote the *shape* of idempotency without understanding what makes it actually idempotent — the key must be deterministic from the request, not random.

**Bug 2.** The ownership validation read from a stale read-replica of the database. Refund requests for orders created in the last 30 seconds frequently failed the ownership check, so Claude's "fix" was to retry the validation 3 times with backoff. The retries occasionally succeeded by sampling a different replica — including some that returned *other users'* matching orders due to a bug elsewhere in the system.

**Bug 3.** The structured logging included the full request body. Including the credit card token. Including the masked PAN. Including, for some legacy clients, the raw card number. PCI auditors found out about this 6 weeks later.

Rahul reviewed the code. He read every line. He saw nothing wrong. The code "looked correct."

That is the most important sentence in this blog post.

---

## The Diagnosis

Here is the question we have to ask: **how did three production-grade bugs ship past a senior engineer who actually read the code?**

The answer is not that Rahul was careless. He was operating with the wrong mental model of what Claude is.

Most developers, when they hear "AI pair programmer," picture something like this:

```
        Developer                    AI
           ↓                          ↓
     [thinks about it]          [helps think]
           ↓                          ↓
     [writes code]    ←→        [suggests code]
           ↓                          ↓
     [verifies it]              [reviews it]
```

This mental model implies a *collaboration*. Two minds, working together, each catching what the other misses. It is wrong. It does not describe what Claude is.

Here is the actual mental model:

```
       Developer                       Claude
          ↓                              ↓
   [thinks about it]            [does not think]
          ↓                              ↓
   [writes a spec]      →     [transforms spec into code]
          ↓                              ↓
   [verifies the code]          [moves on instantly]
```

Claude is not a collaborator. It is a **transformer**. You hand it a description; it produces a plausible artifact. It does not think about your system. It does not push back on bad ideas. It does not remember what was decided in last week's design review. It does not know your incidents, your compliance posture, your team's velocity, or your unspoken constraints.

It does not know your auth layer.  
It does not know your data classification policy.  
It does not know that your payments team has a runbook that says "always use deterministic idempotency keys."  
It does not know any of this — unless you put it in the prompt. Every time.

When you internalize this, three things change:

1. You stop being surprised when Claude misses something you thought was obvious
2. You stop reviewing Claude's code as if a peer wrote it
3. You start writing prompts that compensate for what Claude cannot know

---

## The Better Metaphor: The Contractor

The mental model that actually works is this:

> **Claude is an extraordinarily fast, impressively knowledgeable, and deeply literal contractor.**

Think about what it means to hire a contractor for your house.

- They will build exactly what you describe — including the parts you got wrong
- They will work fast, but they will not know that the inspector cares about a specific local code
- They will produce work that looks professional, because they have done a thousand of these before
- They will not tell you "this is a bad design" because that is not what they were hired for
- They will replace your custom-built window with a standard one if it gets in their way, because *to them* the standard one is equivalent
- They will leave on Friday at 5 PM regardless of whether the work is right

A contractor is **enormously useful**. They are also a **liability without an architect**. The architect is the one who:

- Decides what to build (not just describes it)
- Catches the contractor's local decisions that violate the larger plan
- Knows which "improvements" the contractor will suggest are actually regressions
- Walks the site, looks at the work, and finds the problem before the building inspector does

When you use Claude, **you are the architect**. There is no second architect on the other end of the keyboard.

---

## The Three Properties That Make Claude Dangerous (and Useful)

To use Claude well, you have to understand the three specific properties that make it so productive — and so risky.

### Property 1: Uniform Confidence

Claude's output confidence is determined by how locally coherent it can make the response, not by whether the response is correct.

A subtly wrong database query and a completely correct one look identical. Same tone. Same comments. Same structure. You cannot use the output's *style* as a signal for its *quality*.

Compare with a human:

```
Developer (uncertain):
   "I think this should work, but I'm not 100% sure about the locking semantics —
    can you double-check the section where we hold the lock across the network call?"

Claude (always):
   "Here is the implementation. It handles concurrency correctly using optimistic
    locking and includes appropriate retry logic for transient failures."
```

Both might be wrong about the locking. Only one tells you to look.

Your brain is wired to use speaker confidence as a quality signal. With humans, this is a useful heuristic — confident speakers are usually competent. With Claude, it is a trap. You have to consciously override this instinct **every time**.

### Property 2: Pattern Matching, Not Reasoning

Claude does not reason about your code. It pattern-matches against billions of code examples.

When Claude says *"this should integrate cleanly with your existing auth layer,"* it is not reasoning from knowledge of your auth layer. It is pattern-matching to: *"code structured like this typically integrates cleanly with auth layers that typically look like the ones in my training data."*

This matters in two specific ways:

**(a) Your system may not be typical.** Your auth layer may have a quirk specific to your business. Your idempotency requirements may be tighter than the average codebase. Your scale may put you in failure modes most codebases never hit. Claude has not seen any of this — and will not warn you when its patterns don't apply.

**(b) Claude infers structure even when it doesn't have the information.** Ask Claude about a service that doesn't exist in your codebase. It will not say "I don't know about that service." It will infer what such a service probably looks like and respond as if it knows. This is the source of every "hallucinated dependency" bug you have ever debugged.

### Property 3: No Persistent Memory

Claude does not remember anything between sessions.

Last Tuesday you spent an hour explaining to Claude why your team uses event sourcing for audit logs. Today you ask Claude to add a new audit event. It will not remember the event sourcing context. It will write the code using whatever pattern looks most reasonable based on the file it sees right now — possibly direct database inserts.

Even within a long session, Claude's effective memory degrades. The context window has a limit, and as the conversation grows, earlier context gets deprioritized. You told Claude in message 3 that *"all external API calls must have a 5-second timeout."* By message 40, Claude is happily generating untimed `RestTemplate` calls.

This is not a bug. This is how the tool works. Your job is to design your workflow around it — not to expect it to behave like a human collaborator who remembers what was decided.

---

## What This Means for How You Work

These three properties — uniform confidence, pattern matching, no memory — combine into a specific operational discipline you have to adopt.

### Discipline 1: Specificity Is the Skill

The vague prompt is the enemy.

```
Vague:
  "Add a refund endpoint to the orders service."

Specific:
  "Add a POST /orders/{id}/refund endpoint to the orders service.
  
  Constraints:
  - Idempotency key MUST be deterministic — derived from order_id + refund_amount.
    NEVER use UUID.randomUUID() — we had an incident with that pattern in payments.
  - All database reads in this flow must use the primary, not the read replica,
    because of cross-replica staleness during high-write windows.
  - Logging must NOT include any field from the request body — request bodies in
    this service contain card data. Log only order_id and outcome.
  - Validate ownership via OrderOwnershipService.verify(userId, orderId), which
    handles the auth context correctly. Do not write a new validation check.
  - The payment gateway client is GatewayClient.refund(idempotencyKey, amount).
    It already retries internally — do not add a retry loop around it."
```

The second prompt is 10× longer. It produces code that is 10× more likely to be correct. **Every constraint you fail to state is a constraint Claude will violate.** Not maliciously — by default.

This is not "prompt engineering." It is requirements engineering. The same discipline you have always needed to write good specs, applied to a contractor who will follow them with perfect literalness.

### Discipline 2: Verify, Don't Validate

There is a difference between **validating** code and **verifying** code.

Validation is: *"I read it, it looks right, the tests pass, ship it."*

Verification is: *"Here is a specific thing this code is supposed to do. Here is the evidence that it actually does it."*

For Claude-generated code, validation is insufficient. The code will *look* right. The tests *will* pass — because Claude wrote both the code and the tests, and they share the same assumptions.

You have to verify. Specifically:

- **Verify every import.** Open the file, grep the import, confirm the library is in `pom.xml` or `package.json` at the expected version.
- **Verify every external call.** If Claude calls `PaymentGateway.refund(...)`, open the actual `PaymentGateway` class and confirm the method exists with that signature.
- **Verify every config key.** If Claude references `payment.gateway.timeout-ms`, open the actual config and confirm the key is defined.
- **Verify the behavior, not the description.** Claude will tell you *"this returns null if not found."* Run the code and confirm. Read the code path and confirm. Don't trust the description.

If this sounds tedious, it is. It is also the difference between shipping good software and apologizing in your next post-mortem.

### Discipline 3: Re-Anchor, Don't Trust the Context Window

For any non-trivial Claude session, you need to manage context like an operator, not like a user.

Two practices that experienced teams adopt:

**(a) The session contract.** A short, plain-text document with your non-negotiable constraints. You paste it at the start of every new task within a session, not just at the start of the session.

```
SESSION CONTRACT — Orders Service
─────────────────────────────────
1. Idempotency keys MUST be deterministic.
2. Reads in write paths use the primary DB. Not read replicas.
3. Logs must not contain request bodies.
4. External APIs use timeouts. No exceptions.
5. Code follows the existing repository pattern. No ORMs.
6. Errors are returned as Result<T, E>. No throwing.
```

You paste this. Every. New. Task.

**(b) The compliance check.** After Claude produces non-trivial code, ask explicitly: *"Does this implementation respect each of the constraints in the session contract? Go through them one at a time."* Claude is reasonably good at checking its own compliance when asked. It is bad at remembering to check on its own.

### Discipline 4: Scope Discipline

Claude is helpful. Being helpful means doing more than asked. This is a problem.

You ask Claude to fix a bug in `processPayment()`. Claude:
- Fixes the bug ✓
- Refactors the error handling (not asked)
- Renames `processPayment` to `executePayment` for clarity (not asked, breaks callers)
- Adds a helper utility (not asked)
- "Improves" the logging (silently removes a log line your monitoring depends on)

Every one of these "improvements" is a change you didn't ask for, didn't review carefully, and didn't communicate to your team. Every one is a potential regression.

**Rule:** After every Claude session, `git diff` the output against what you asked for. Anything outside scope is a risk. Each out-of-scope change is a deliberate decision: keep it, revert it, or split it into a separate ticket. Never merge an out-of-scope change because "it looks fine."

The discipline is exactly the same as you would apply to a contractor who replaced your custom window with a standard one. "Looks fine" is not how you accept work.

---

## A Concrete Example: The Right Way to Use Claude

Let us return to Rahul's refund endpoint. Here is how the same task would have gone with the architect's discipline applied.

### Step 1 — Constraints first

Before opening Claude, Rahul writes:

```
Functional:
- POST /orders/{id}/refund accepting { amount: number }
- Returns 200 with { refund_id, status } on success
- Returns 400 on invalid amount, 403 on ownership failure, 422 on already-refunded

Constraints:
- Idempotency key = SHA256(order_id + amount). NEVER random.
- Validation via existing OrderOwnershipService.verify() — single source of truth.
- DB reads in this path use primary (DataSource.PRIMARY).
- No request body fields in logs. Log order_id, user_id, outcome only.
- Payment gateway client (GatewayClient.refund) handles its own retries — no wrapper retry.

Out of scope:
- Refund partial/full logic (use amount as-is for now)
- Notification on refund (separate service handles this)
- Refund history endpoint (separate ticket)

Acceptance:
- Same request sent twice returns the same refund_id.
- Ownership check failure does NOT trigger gateway call.
- Logs in staging do not show any field except order_id and outcome.
```

This took 8 minutes to write. It would have prevented all three production bugs.

### Step 2 — Provide context

```
"I am adding a refund endpoint to the orders service.
 The codebase uses Spring Boot, the repository pattern (no ORM), and 
 returns errors as Result<T, E>.
 Existing services you should know about:
 - OrderOwnershipService.verify(userId, orderId): Result<Owner, AuthError>
 - GatewayClient.refund(idempotencyKey: String, amount: Money): Result<RefundResult, GatewayError>
 - PrimaryDataSource for any reads in this write path.
 
 Constraints (see SESSION CONTRACT):
 [paste constraints from Step 1]
 
 Please produce a design before any code."
```

### Step 3 — Design before code

Claude responds with a design. Rahul reads it. Asks one specific question:

> *"What assumptions are you making in this design? What would need to be true about my system for this to work correctly?"*

Claude lists 4 assumptions. Two of them are wrong for this codebase. Rahul corrects them. The corrected design is now grounded in reality.

### Step 4 — Implementation

Claude generates the code. Rahul opens it and:

- Greps for every import → all exist
- Opens `GatewayClient` → confirms `refund(String, Money)` signature
- Opens `OrderOwnershipService` → confirms `verify(UUID, UUID)` exists
- Searches logs for any reference to request body → finds none
- Searches for `UUID.randomUUID` → finds none
- Reviews the idempotency key generation → SHA256-based, deterministic ✓

### Step 5 — Tests with real assertions

```
"Write tests for this endpoint. Here is the unhappy path checklist:
1. Same request twice → same refund_id, gateway called only once.
2. Different request with same idempotency key (poison) → 422.
3. Ownership check fails → gateway NEVER called, 403 returned.
4. Gateway returns transient error → no retry from our side, error propagates.
5. Gateway returns permanent error → 502, no DB write.
6. Negative amount → 400, no gateway call.
7. Amount exceeds order total → 422, no gateway call.

For each test, the assertion must verify SPECIFIC state, not just 'no exception thrown'.
Specifically assert: response status, response body fields, gateway call count, 
DB state after the operation."
```

Claude produces tests that actually catch bugs.

### Step 6 — Out-of-scope check

Rahul runs `git diff` and finds Claude has added a new utility class `MoneyFormatter` that he didn't ask for. He reverts it — and creates a separate ticket to discuss whether `MoneyFormatter` should exist as part of the broader money-handling refactor.

### Step 7 — Review with adversarial posture

Before merging, Rahul reads every line — added and removed — asking *"what could a malicious or unusual user do to break this?"* He spots that the controller does not have rate limiting. He adds rate limiting via the existing `@RateLimited` annotation.

### Outcome

The endpoint ships. There is no incident at 11:47 PM. The total time spent was 6 hours instead of 4 — but the time spent on rework, on-call response, the post-mortem, and the PCI audit follow-up would have been 6 *days*.

This is what it means to use Claude as an architect rather than as a developer.

---

## The Architect's Mental Model — Summarized

```
┌─────────────────────────────────────────────────────────────────┐
│                    Claude Is a Contractor                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Properties                                                     │
│    • Uniform confidence — style ≠ correctness                   │
│    • Pattern matching — not reasoning about your system         │
│    • No persistent memory — across or within sessions           │
│                                                                 │
│  Implications                                                   │
│    • You are the architect. There is no second architect.       │
│    • Every constraint you don't state will be violated.         │
│    • Verification is your job, not Claude's.                    │
│    • Out-of-scope work is a risk, not a bonus.                  │
│                                                                 │
│  Disciplines                                                    │
│    • Specificity is the skill                                   │
│    • Verify, don't validate                                     │
│    • Re-anchor, don't trust the context window                  │
│    • Scope discipline — diff every session                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## The Discomfort Test

The single most reliable signal that something is wrong with Claude-generated code is this:

> *"I'm not entirely sure why this works, but it seems to."*

That sentence, said out loud or thought silently, is the discomfort test. If you say it and ship anyway, you have failed the test. Code you don't understand is code that will wake you up at 3 AM.

The discomfort is the signal. Investigate it. Either you discover Claude made a wrong assumption, or you genuinely understand the code at a deeper level. Both are wins. Suppressing the discomfort and shipping is the only losing move.

---

## Checklist — Before You Merge Any Claude-Generated Code

Cut this out. Tape it to your monitor.

```
□ I wrote constraints before Claude wrote code
□ I provided existing-system context in the prompt
□ I asked Claude what assumptions it was making
□ Every import has been verified to exist
□ Every external call has been verified against the actual API
□ Every config key has been verified against the actual config
□ I ran `git diff` and decided about every out-of-scope change
□ I read every removed line, not just every added line
□ I read every line — read, not scanned
□ Tests have specific assertions, not just "no exception thrown"
□ Tests include edge cases from MY checklist, not just Claude's
□ Mocks (if any) match the real dependency's actual behavior
□ Logs do not contain sensitive data
□ Observability requirements are met
□ I do not have the "I'm not sure why this works" feeling
```

15 items. ~10 minutes per non-trivial PR. The number of incidents this prevents is enormous.

---

## Interview Q&A

**Q1: What is the difference between using Claude as a developer and using Claude as an architect?**
> A developer asks Claude for code and reviews what comes back. An architect specifies the constraints, the context, the existing systems, the out-of-scope items, and the acceptance criteria — then reviews Claude's output as evidence against those specifications. The developer trusts the tool. The architect verifies the tool's output against a known specification.

**Q2: What is the most common mistake teams make when adopting Claude?**
> Treating it as a junior developer that will get better over time. Claude is not learning from your codebase. It is not building intuition about your team's preferences. Every session is fresh. Teams that expect cumulative improvement are accumulating cumulative debt instead. The fix is to externalize the intelligence — capture it in `CLAUDE.md`, prompts, and team-level checklists — rather than expect Claude to internalize it.

**Q3: How do you know when NOT to use Claude?**
> Three signals: (1) The task requires deep knowledge of your specific system that Claude cannot have. (2) The cost of a wrong answer is high and the wrong answer would be invisible — security code, data migrations, distributed coordination. (3) The task is small enough that the prompt-and-verify cycle takes longer than just writing it. For these tasks, write the code yourself or pair with a colleague.

**Q4: What is a `CLAUDE.md` file and why does your team need one?**
> A `CLAUDE.md` is a configuration document at the repo root that contains your team's actual conventions — patterns adopted, patterns rejected, libraries available, libraries prohibited, security and observability requirements. Claude reads it at the start of every session. It is the mechanism that prevents session-to-session drift. Without it, each Claude session produces internally consistent but mutually inconsistent code, and the codebase drifts.

**Q5: How do you train a junior engineer to use Claude well?**
> Counter-intuitively, junior engineers should use Claude *less* than seniors, not more. Junior engineers are still building the architectural judgment required to specify constraints, verify output, and catch missing pieces. Claude amplifies whatever judgment you bring to it — so a junior with Claude amplifies junior-level judgment, producing junior-level mistakes at senior-level speed. Teach juniors to write specifications and review code first. Give them Claude when they can already do those things well.

---

## The Closing Thought

There is a temptation — especially under deadline pressure — to treat Claude as an oracle. Ask the question, accept the answer, ship the code. Move on.

This works for a while. It works until the production incident. Then it stops working very loudly.

The developers who will thrive in the AI-augmented future are not the ones who prompt best. They are the ones who:

- **Think clearest** about their systems
- **Communicate constraints** most precisely
- **Verify output** most rigorously
- **Maintain the bar** for what ships, regardless of who or what wrote the first draft

Claude is a tool that amplifies judgment. It does not substitute for it. The judgment is yours. The verification is yours. The decision to merge is yours.

The code that ships under your name was reviewed by you. Not by Claude.

That is what it means to be the architect.

---

## What's Next in This Series

The next post (**02 — Requirements & Design: The Underspecification Spiral**) walks through the specific failure mode where Claude's eagerness to start coding causes developers to skip the design phase, and how to use Claude as a design sparring partner without letting it substitute for the design phase itself.

Subscribe / follow:

📺 [YouTube — CodeWithSunchitDudeja](https://www.youtube.com/@CodeWithSunchitDudeja)  
📸 [Instagram — @sunchitdudeja](https://www.instagram.com/sunchitdudeja/)

---

*01 · How Architects Use Claude · What Not to Do With Claude — Series introduction and foundational mental model*
