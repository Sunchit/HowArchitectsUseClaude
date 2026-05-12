# How Architects Use Claude

> **A field guide for developers, senior engineers, and architects who want to extract real leverage from AI coding tools — without inheriting their failure modes.**

Most developers use Claude wrong. Not because they prompt poorly. Because they think of Claude as an autocomplete, an oracle, or a junior developer. It is none of those things.

This series is for engineers who have moved past the "wow, this is fast" phase and are now asking: **"Why does this keep going wrong?"**

---

## Who Is This For?

| You Are | What You Will Get |
|---|---|
| **Developer using Claude daily** | The failure modes you are accumulating without realizing it, and how to catch them before they ship |
| **Senior engineer** | A vocabulary and process for reviewing AI-generated code at scale |
| **Engineering Manager** | Team-level practices to manage AI-augmented workflows without sacrificing quality |
| **Architect** | A mental model for using Claude as a force multiplier on your judgment rather than a substitute for it |
| **Tech Lead** | A `CLAUDE.md` template and review checklist your team can adopt this week |

---

## The Premise

> *Claude is an extraordinarily fast, impressively knowledgeable, and deeply literal contractor who will build exactly what you describe — and sometimes things you didn't describe — with full confidence and zero second-guessing.*

If you don't manage the engagement, it will manage itself. And that is where things go wrong.

This series walks through every phase of the developer lifecycle — design, implementation, testing, review, deployment, maintenance — and surfaces the **specific, non-obvious failure modes** that emerge when Claude is your primary coding collaborator.

Not "AI bad." Claude is genuinely transformative. But unsupervised Claude at scale will teach you hard lessons that this series can front-load.

---

## What Makes This Series Different

There is no shortage of AI-coding content. Most of it is one of two things:

1. **Cheerleading** — "AI will 10× your productivity!"
2. **Doom-mongering** — "AI will hallucinate your codebase to death!"

Neither is useful in production. This series is the third option: **practical engineering discipline for working alongside AI**, written from the perspective of architects who have seen what works, what fails, and what looks like progress but isn't.

Every post is a **story** — a real-world scenario where a developer used Claude, hit a wall, and learned something the hard way. Followed by the architectural pattern that prevents it.

---

## The Series

### Part 1 — The Foundations

| # | Post | Core Idea | Link |
|---|------|-----------|------|
| 01 | The Contractor Who Never Says No — How to Actually Think About Claude | Claude is not a junior dev, not an oracle, not autocomplete. It is a literal contractor. Treat it accordingly. | [Read](./01_The_Contractor_Who_Never_Says_No.md) |
| _more coming_ | | | |

### Part 2 — The Lifecycle, Phase by Phase

| # | Post | Core Concept | Link |
|---|------|--------------|------|
| 02 | Requirements & Design — The Underspecification Spiral | How vague prompts produce plausible-looking output that's wrong in the details that matter | _coming soon_ |
| 03 | Implementation — Scope Integrity, Hallucination Taxonomy, Context Window Cliff | The five types of Claude hallucinations and how to catch each one | _coming soon_ |
| 04 | Testing — The Mocked-Tests-Pass Problem | Why high coverage on Claude-generated tests can be worse than no tests | _coming soon_ |
| 05 | Code Review — Why Claude PRs Need More Review, Not Less | The familiarity illusion and the adversarial review posture | _coming soon_ |
| 06 | Security Review — The OWASP Top 10 in AI-Generated Code | Where Claude introduces vulnerabilities through pattern-matching | _coming soon_ |
| 07 | Deployment — Claude Does Not Think About Rollouts | Feature flags, canaries, observability — all invisible to Claude unless you ask | _coming soon_ |
| 08 | Maintenance — Archaeological Debt and CLAUDE.md as a Living Document | How AI-generated codebases accumulate invisible technical debt | _coming soon_ |

### Part 3 — Organizational Practices

| # | Post | Core Concept | Link |
|---|------|--------------|------|
| 09 | Designing CLAUDE.md — The Contract Between Your Team and Claude | A template and the seven sections every CLAUDE.md needs | _coming soon_ |
| 10 | Code Review Checklists for AI-Generated Code | Specific items your team should add to every PR template | _coming soon_ |
| 11 | Incident Post-Mortems for AI-Caused Bugs | The post-mortem question that turns incidents into CLAUDE.md updates | _coming soon_ |
| 12 | When to NOT Use Claude — A Tier-by-Tier Risk Model | The three signals that mean you should write the code yourself | _coming soon_ |

---

## What You Will Learn

- How to spot Claude's failure modes before they cost you a sprint
- The five categories of hallucination and how to catch each one
- Why your scope-control discipline matters more with Claude than without it
- How to write a `CLAUDE.md` that actually changes Claude's behavior
- Why "100% test coverage" can be meaningless on AI-generated code
- The single most important review question to ask after every Claude session
- How to build a team-level workflow that captures Claude's speed without inheriting its risks

---

## What This Series Is Not

- **Not a prompting guide.** Prompting is a tactical skill. This series is about systemic engineering practice.
- **Not anti-AI.** Every post assumes you are going to use Claude. The question is how to do it well.
- **Not generic AI safety.** This is about software engineering specifically — code, reviews, deployments, incidents.
- **Not vendor-specific to Claude.** The patterns apply to any modern coding LLM. Examples are Claude-flavored because that is what most teams are using.

---

## Structure of Each Post

```
1. The Story        — A real scenario where Claude was used and things broke
2. The Diagnosis    — What went wrong, and why it was invisible
3. The Pattern      — The architectural practice that prevents it
4. The Checklist    — Concrete things to do this week
5. Interview Q&A    — How to discuss this in senior/staff engineering interviews
```

---

## The Architect's Mindset Shift

| Developer Thinking | Architect Thinking |
|--------------------|-------------------|
| "How do I prompt Claude better?" | "How do I verify what Claude produced?" |
| "Claude said this should work." | "What is Claude assuming that I haven't verified?" |
| "The tests pass, ship it." | "Do the tests actually catch the bugs that would surface in production?" |
| "Claude is fast, I'm productive." | "Am I shipping faster, or just generating faster?" |
| "I'll review the diff quickly." | "I'll read every line, and I'll especially read what was removed." |

The developer treats Claude as an oracle. The architect treats Claude as an instrument — powerful, precise, and only as good as the hand that wields it.

---

## Author

**Sunchit Dudeja**  
Engineering Leader · System Design Educator  
Building this series for developers who refuse to outsource their judgment to a tool that does not have any.

📺 [YouTube — CodeWithSunchitDudeja](https://www.youtube.com/@CodeWithSunchitDudeja)  
📸 [Instagram — @sunchitdudeja](https://www.instagram.com/sunchitdudeja/)

---

> *"A developer uses Claude to write code faster. An architect uses Claude to make better decisions faster — and knows the difference."*
