---
name: code-review
description: Use when the user explicitly asks for a code review. Two modes, both manually triggered. MODE A — self-review of code Claude wrote in the current session; triggers on cues like "review what you just wrote", "check your last change", "audit that", "review your work", "self-review", or any request for Claude to critique its own recent output. MODE B — review of existing code the user brings in; triggers when the user pastes code or a diff and asks for a review, links to or mentions a PR/pull request, or asks to "review", "audit", "check", or "critique" a file, function, or change set with phrases like "is this good", "what's wrong with this", "any issues", "look at this". Both modes apply the team's CLAUDE.md rules (Simplicity First, Surgical Changes, Naming, Complexity Check, Verify don't declare, Documentation) as review criteria. Loads relevant language skills (csharp-avalonia, csharp-rest-aspnetcore, csharp-efcore-sqlite, csharp-windows-service, csharp-sensor-pipeline, java-sensor-pipeline) when file type or architectural cues indicate them. Do not fire this skill automatically — wait for an explicit review request.
---

# Code Review

Two modes, one checklist. CLAUDE.md rules are the standard. Language-specific skills extend the checklist when relevant.

**Both modes are user-triggered only.** Don't run a self-review after every coding turn; wait until the user asks for one. A review delivered without being asked is noise.

---

## Two modes

### Mode A — Self-review of Claude's recent output

Triggered when the user asks Claude to review code Claude itself wrote earlier in the session. Typical cues:

- "Review what you just wrote."
- "Check your last change against CLAUDE.md."
- "Self-review the last 30 minutes."
- "Audit what you built."
- "Any issues with what you did?"

**What Mode A produces:**

A focused critique of Claude's own recent code, held to the same standard as any other review. Structured report if there are findings; a short acknowledgement if the code passes the checklist.

**Why it's separate from Mode B:** Claude has session context the user doesn't need to paste — the recent edits, the reasoning, the file paths. Mode A uses that context directly. Mode B requires the user to supply the code because Claude doesn't have it.

**Bias to watch for:** Claude should hold its own code to the same standard as code from elsewhere. The temptation to go easy on recent work is a real failure mode — see the "Self-check for the reviewer" section at the end. If anything, Claude should be slightly harsher in Mode A, because the cost of finding issues now (while the context is fresh) is lower than finding them in Mode B days later.

### Mode B — Review of code the user brings in

Triggered when the user pastes code, shares a diff, links to a PR, or otherwise provides code for review.

**Cues:**

- Explicit verbs: "review", "audit", "check", "critique", "look at this", "what do you think of".
- PR/diff: user links to or pastes a diff; mentions "my PR", "this pull request", "these changes".
- Implicit when code is provided: user pastes >15 lines of code and asks a quality-framed question ("is this good", "any issues", "what's wrong", "thoughts").

**What Mode B produces:** a structured report (see template below).

---

## The checklist

Every review — Mode A or Mode B — applies the same criteria from CLAUDE.md. Ordered by severity: structural issues first, style issues last.

### 1. Simplicity (CLAUDE.md rule 1: Simplicity First)

- Does the code solve what was asked, and nothing more? No speculative flexibility, no unused parameters, no error handling for conditions that can't occur.
- If the code is >4× the length a senior engineer would write for the same task, flag it.
- More than 3 levels of nesting? Flag it — suggest early returns or extraction.

### 2. Surgical scope (CLAUDE.md rule 2: Surgical Changes)

- Every changed line traces to the stated request. Lines that don't are review findings.
- Adjacent code "improvements" not asked for — flag them as scope creep.
- Pre-existing symbols renamed — flag as violation of Surgical Changes.
- Pre-existing dead code cleaned up — should be mentioned in passing, not cleaned up silently.

### 3. Naming (CLAUDE.md rule 3: Naming)

- Any name containing a weasel qualifier: `Data`, `Info`, `Manager`, `Helper`, `Util`, `Service`, `Handler`, `Wrapper`. Flag it. Suggest specific alternatives.
- Names that need a qualifier to disambiguate — the abstraction is wrong. Flag and suggest re-scoping.
- Applies only to code introduced in this change. Pre-existing names are out of scope.

### 4. Complexity (CLAUDE.md rule 4: Complexity check before editing)

Run the three questions on each new or changed function:

- Can I describe this function's job in one sentence without "and"?
- If the signature changed, do more than two unrelated modules need edits?
- Is the change local, or does understanding it require reading the whole file?

Route by change size:
- Small change in tangled code → note the tangle, accept the change.
- Large change in tangled code → **refactor should have been proposed first**. Flag that the process was skipped.
- Any change in clean code → fine.

### 5. Verification (CLAUDE.md rule 5: Verify, don't declare)

- Bug fix claimed but no reproducing test (when test framework exists) — flag.
- Multi-step task without a verify step per step — flag.
- Any claim of "fixed" or "done" without a verification result shown — flag.

### 6. Documentation (CLAUDE.md rule 6: Documentation)

- Comments that describe *what* code does, not *why* — flag, delete or rename instead.
- Duplicated logic or knowledge across two places — flag, verify whether it's genuinely the same concept.
- Missing docs in `/docs/*.md` for a major new concept or module — flag as follow-up work, not a blocker.

### 7. Language-specific standards

If the code is in a language with a matching skill, load that skill's rules and extend the checklist:

- **`.cs` in an Avalonia project** → `csharp-avalonia` (ViewLocator patterns, compiled bindings, no code-behind logic)
- **`.cs` ASP.NET Core endpoints** → `csharp-rest-aspnetcore` (Minimal API vs Controllers rule, TypedResults, ProblemDetails)
- **`.cs` with DbContext** → `csharp-efcore-sqlite` (AsNoTracking default, Select projections, transaction boundaries)
- **`.cs` BackgroundService** → `csharp-windows-service` (stoppingToken honored, PeriodicTimer over Task.Delay, scope per unit of work)
- **`.cs` sensor pipeline code** → `csharp-sensor-pipeline` (five-stage contract, transport recipes, channel-only-when-needed rule)
- **Java sensor pipeline code** → `java-sensor-pipeline` (same stage contract, Reactor idioms, no boundedElastic on ingress)

Detection is by file content, not filename extension alone. A `.cs` file with `public class X : BackgroundService` is a Windows Service concern even if the project also has Avalonia views elsewhere.

**If no matching skill is available**, apply generic principles from CLAUDE.md only. Don't invent language rules Claude isn't confident in.

---

## Output format

### Mode A output

Same structured report as Mode B. The difference is *what* is reviewed (Claude's own recent output, using session context) and sometimes *scope* (a few files from the recent turns, not a whole PR). Format:

```
## Self-review: <what was reviewed>

Code reviewed: [brief summary — e.g., "OrderService.cs and OrderController.cs, changes from the last three turns"]

### Blocking
- ...

### Suggested
- ...

### Noted
- ...

### Summary
N blocking, N suggested, N noted.

Want me to apply the Blocking and Suggested fixes?
```

If there are zero findings, say so directly — don't pad:

> Self-reviewed `OrderService.cs` and `OrderController.cs` against CLAUDE.md. No findings. The change stayed within stated scope, names are direct, no pre-existing symbols renamed, and the bug-fix commit has a reproducing test.

A clean-review acknowledgement like this is more useful than a forced finding. Don't invent issues to look thorough.

### Mode B output

Structured report. Severity prefixes. Grouped by rule.

```
## Review: <file or PR>

### Blocking (must fix before merge)
- **Surgical scope** (line X): <change> modifies `UserRepository.Save`, which is outside the stated task of
  adding a new endpoint. Per CLAUDE.md rule 2, every changed line must trace to the request.
  → Revert this change, or raise it as a separate PR.

### Suggested (should fix)
- **Naming** (line Y): `OrderDataManager` contains two weasel qualifiers (`Data`, `Manager`).
  The class actually does order persistence. Rename to `OrderRepository`.
- **Simplicity** (line Z): the `try/catch/rethrow` wrapper adds no value — the caller handles
  the same exception type. Remove.

### Noted (informational, not actionable)
- **Pre-existing dead code** (lines A-B): the `legacyHandler` method appears unused but predates
  this change. Out of scope per CLAUDE.md rule 2.

### Summary
N blocking, N suggested, N noted.

Want me to apply the Blocking and Suggested fixes?
```

Severity tiers:

- **Blocking** — violates a load-bearing rule (Surgical, Simplicity, Verification for bug fixes). Merging without fixing creates tech debt or breaks CLAUDE.md compliance visibly.
- **Suggested** — should fix, but reasonable people might disagree. Naming smells, minor simplifications.
- **Noted** — observed but not actionable in this change. Pre-existing issues, scope-external concerns.

No fourth tier. "Optional nice-to-have" is the same as "noted" — if it's not worth fixing, it's not worth listing as an action.

### After the report

Ask once: *"Want me to apply the Blocking and Suggested fixes?"*

If yes → apply them in one go, one commit/change per finding where possible, then summarize what was applied.

If no, or if user wants to triage → offer the diff of the first Blocking fix for discussion. Don't push.

**Never apply fixes without asking.** Even in Mode A. The user might have context that makes the "issue" not an issue.

---

## What a review is not

- **Not a style guide audit.** Formatting (indentation, line length, brace placement) follows existing code in the file — CLAUDE.md rule 2 says match existing style. Don't flag these unless they're obviously wrong.
- **Not a test for completeness.** If the task said "add a CREATE endpoint" and the code does that, missing PUT/DELETE is not a review finding — it wasn't asked for.
- **Not a rewrite.** A review observes and suggests. If half the file would need to change to meet the standard, the right finding is "this should have been proposed as a refactor" (rule 4), not a 200-line rewrite buried in the fixes.
- **Not security-complete.** Obvious security issues (SQL injection via string concat, hardcoded secrets, missing auth checks) are called out in a `### Security` section if found. Systematic security review is a separate discipline.
- **Not performance-complete.** Obvious performance issues (N+1 queries, sync-over-async, blocking on the UI thread) are called out. Systematic profiling is a separate discipline.

---

## Common findings — a field guide

See `references/common-findings.md` for a longer catalogue. Highlights:

**"Improving adjacent code"** — the most frequent Surgical Changes violation. Claude (or a developer) sees an ugly method next to the one they're changing and cleans it up. Flag it as a separate PR.

**"Just in case" error handling** — catching exceptions that can't happen, or returning nulls the caller will never see. Violates Simplicity. Delete it; let it crash if it crashes.

**Helper / Manager / Util / Service suffix** — rename time. Ask: what does it actually do? If the answer is "it helps", the abstraction is wrong.

**N+1 queries** — `foreach (var x in entities) { await repo.GetDetail(x.Id); }`. Suggest `Include`, `Select` projection, or a batch API. Flag as Blocking in hot paths, Suggested elsewhere.

**Sync-over-async** — `.Result` or `.Wait()` inside an async context. Blocking, Blocking.

**Premature abstraction** — interface with one implementation, factory for a type that never varies, strategy pattern for a single strategy. Flag under Simplicity.

**Duplicated logic that "looks the same"** — CLAUDE.md rule 6's nuance. Before flagging as duplication, verify both copies actually mean the same thing. Same code shape, different domain concept → stays separate.

---

## Interactions with language skills

When a language skill is loaded alongside this one, the review checklist extends. The language skill's "Rules" and "What not to do" sections become additional Blocking/Suggested items.

**Example: reviewing an Avalonia ViewModel.**

Base checklist finds: naming (rule 3), simplicity (rule 1), surgical scope (rule 2). The `csharp-avalonia` skill adds: inherits from `ObservableObject`?, `[ObservableProperty]` partial properties (not legacy fields)?, `[RelayCommand]` for commands (not hand-rolled `ICommand`)?, any `Window` / `Dispatcher` references in the VM (violation)?

The review report groups findings by CLAUDE.md rule number, not by which skill contributed the rule. The reader doesn't care where the rule came from — they care what's wrong.

---

## Self-check for the reviewer

Before producing any Mode B report, Claude asks itself:

1. **Am I being consistent with my own output this session?** If Claude wrote the code three turns ago and is now reviewing it, Claude should hold itself to the same standard, not go easier on its own work.
2. **Am I inventing rules not in CLAUDE.md or the loaded language skill?** If a finding doesn't trace to a specific CLAUDE.md rule or language-skill rule, either skip it or clearly mark it as "reviewer opinion, not a rule."
3. **Am I flagging things for completeness rather than importance?** Three high-signal findings beat ten where most are noise. The Noted section is where low-importance findings go — if everything is in Noted, the review is a wall of static.

If any of these is off, rewrite the review before producing it.
