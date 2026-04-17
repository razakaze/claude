# Common Findings — A Field Guide

Organized by CLAUDE.md rule. Each entry: what to look for, why it violates the rule, what to suggest.

---

## Rule 1 — Simplicity First

### Premature abstraction

**Signals:**
- Interface with a single implementation and no plan for more.
- Factory class for a type that never varies.
- Strategy pattern with one strategy.
- Generic method where the generic is never varied at the call site.

**Why it violates:** adds layers that cost cognition for no current benefit. "We might need it later" is not a feature request.

**Suggest:** delete the abstraction. When a second implementation actually appears, extract then.

### Just-in-case error handling

**Signals:**
- `catch (Exception ex) { throw; }` — rethrow-only handlers.
- `catch` for an exception the called code can't throw.
- Null checks on values that can't be null (constructor-enforced, `required`, etc.).
- Defensive copying that the language's type system already makes safe.

**Why it violates:** adds code that runs without purpose. Masks real bugs — the caller assumes the code handles something it doesn't.

**Suggest:** delete. Let the code crash if an impossible state happens — the stack trace will point at the real bug.

### Flexibility for imagined futures

**Signals:**
- Config options with only one valid value.
- Parameters defaulted to the only value callers use.
- `switch` with one case and an unreachable `default`.
- `TODO: support other formats` comments next to code that doesn't and doesn't need to.

**Why it violates:** YAGNI. Fictional requirements cost real complexity.

**Suggest:** hard-code the current single value. Revisit when a second need actually appears.

### Over-long methods

**Signals:**
- Methods over ~50 lines that do more than one thing.
- More than 3 levels of nesting.
- Cyclomatic complexity signals: many branches, many loops, many `&&`/`||` in conditions.

**Why it violates:** one method doing many things violates Single Responsibility and makes testing hard.

**Suggest:** extract methods along the seams. Early-return to flatten nesting.

---

## Rule 2 — Surgical Changes

### Adjacent improvements

**Signals:**
- Diff touches a method that wasn't part of the task description.
- Formatting changes across files that weren't otherwise modified.
- "While I was in there…" comments on the PR.

**Why it violates:** muddies the change's intent. The PR is no longer about what it says it's about. Reviewers can't tell what's the change and what's the noise.

**Suggest:** revert adjacent changes. Raise them as a separate PR or a `TODO:` note.

### Silent rename of pre-existing symbols

**Signals:**
- A public symbol renamed with no mention in the task description.
- Internal symbols renamed because "the old name was bad."

**Why it violates:** rule 2 says "don't rename pre-existing symbols." Rename is scope-creep even when the new name is better.

**Suggest:** revert the rename. Propose it as a separate change.

### Pre-existing dead code cleaned up silently

**Signals:**
- An unused method deleted as part of an unrelated change.
- A commented-out block removed in a diff that's supposed to add a feature.

**Why it violates:** pre-existing dead code is Rule 2's "don't touch" territory. Mention, don't clean.

**Suggest:** restore the dead code. Add a `TODO:` or flag in the PR description.

### Comment / documentation drift

**Signals:**
- Updates to unrelated docstrings or comments.
- `README.md` edits unrelated to the change.

**Why it violates:** scope creep. Same rule.

**Suggest:** revert. Doc cleanups are their own PR.

---

## Rule 3 — Naming

### Qualifier suffixes

**Signals to flag:** `*Data`, `*Info`, `*Manager`, `*Helper`, `*Util`, `*Service`, `*Handler`, `*Wrapper`, `*Processor`, `*Controller` (outside web frameworks), `*Impl`, `*Base`.

**Why it violates:** the qualifier signals the name doesn't describe the single responsibility. The name is a category, not an identity.

**Suggest per case:**
- `OrderDataManager` doing CRUD → `OrderRepository`.
- `PaymentProcessor` doing one specific payment flow → name the flow: `PaymentCapture`, `PaymentRefund`.
- `StringHelper` with three static methods → inline the methods at their call sites, delete the class. If the methods are used widely, name the actual operation: `Normalize`, `Tokenize`.
- `LoggerWrapper` wrapping `ILogger` → if it adds nothing, delete it. If it adds scoping, name the scope: `RequestScopedLogger`, `AuditLogger`.

### Meaningless variable names

**Signals:** `data`, `info`, `obj`, `item`, `result`, `temp`, `value` — outside loop counters and throwaway `_` discards.

**Why it violates:** the variable's role isn't self-describing. Readers have to track what `data` means through the method.

**Suggest:** rename to the actual concept. `data` holding a parsed config → `config`. `result` from a DB call → `customer` or `rows`.

### Inconsistent naming within one change

**Signals:** `userId` in one method, `user_id` in another, `UserID` in a third — all in code Claude just wrote.

**Why it violates:** consistency within a diff is table stakes.

**Suggest:** pick one convention, apply everywhere in the new code. Match existing code's convention if the file already has one.

---

## Rule 4 — Complexity check before editing

### Large change in tangled code without refactor proposal

**Signals:** a PR that rewrites 4+ functions across multiple files, where the original task was a single feature.

**Why it violates:** rule 4 says large change in tangled code should have been preceded by a refactor-scope proposal that the user accepted.

**Suggest:** break the PR into two. One that proposes and applies the refactor; one that adds the feature on top.

### Function doing multiple things

**Signals:** function name contains `And`, or the description requires multiple sentences, or the function has a comment like "// Step 1:" / "// Step 2:".

**Why it violates:** rule 4's first heuristic question fails — "describe this function's job in one sentence without 'and'."

**Suggest:** split along the "and" boundary. Each part becomes its own function.

### High-coupling signature change

**Signals:** a signature change where the call sites span unrelated modules — auth, reporting, UI, tests.

**Why it violates:** rule 4's second heuristic — if more than two unrelated modules need edits, coupling is too high.

**Suggest:** find the seam. Often the solution is a thin adapter: keep the old signature in one place, have it delegate to the new one. Migrate callers over time.

---

## Rule 5 — Verify, don't declare

### "Fixed" without a test

**Signals:** a bug-fix commit message or PR description that says "fixes X" with no accompanying test change — in a codebase that has a test framework in use.

**Why it violates:** rule 5 says "if the repo has a test framework in use, reproduce the bug as a failing test before fixing."

**Suggest:** add a reproducing test. If the test setup is genuinely hard, mention that as a blocker in the PR rather than skipping.

### Claim of completion without verification output

**Signals:** "I've fixed the bug" / "I've implemented the feature" without showing what was verified — no test output, no reproduction steps, no screenshot.

**Why it violates:** rule 5 — "Don't declare success. Show the verification result."

**Suggest:** run the test, paste the output. Or run the repro, paste the output. Or note that manual QA is needed if the change can't be verified automatically.

### Multi-step task with no verify step per step

**Signals:** a PR that does three logical things, with one combined test or no test at all.

**Why it violates:** rule 5's planning requirement.

**Suggest:** one test per logical step. Or an explicit justification in the PR description for why shared verification is sufficient.

---

## Rule 6 — Documentation

### What-comments instead of why-comments

**Signals:**
```csharp
// Increment the counter
counter++;

// Loop over users
foreach (var user in users) { ... }
```

**Why it violates:** rule 6 — "Never comment what the code does."

**Suggest:** delete. If the intent is unclear, improve the name or extract a method whose name explains the intent.

### Duplicated business rules

**Signals:** the same validation, transformation, or calculation appears in two or more places. Often with slight variations.

**Why it violates:** rule 6's duplication rule. One of the copies will drift.

**Suggest:** extract to a single source. But first, verify the two copies really mean the same thing — rule 6's warning about identical code meaning different things. If `customer.CalculateDiscount` and `product.CalculateDiscount` happen to look the same but follow different domain rules, they stay separate.

### Stale comments after a rename

**Signals:** comment references `OrderManager` but the class was renamed to `OrderRepository`.

**Why it violates:** rule 6 — docs that contradict the code are worse than no docs.

**Suggest:** update or delete the comment.

---

## Cross-rule patterns

### The "cleanup" PR

A PR titled "Cleanup and minor refactors" or "Code quality improvements" that touches dozens of files.

**What's wrong:** violates Surgical (rule 2) — no single tractable change to review. Violates Complexity (rule 4) — large cross-module change without a specific refactor proposal.

**Suggest:** split into one PR per refactor. Each PR has a named, justified goal.

### The "framework" PR

A PR that adds a new internal framework or abstraction layer for future features, not for anything in the current task.

**What's wrong:** violates Simplicity (rule 1) and Surgical (rule 2). The feature the user asked for doesn't require the framework.

**Suggest:** remove the framework. Add it when the second use case appears and requires it.

### The "TODO" PR

A PR full of `TODO:` and `// TODO: come back to this` comments instead of actually finishing the work.

**What's wrong:** violates rule 5's "don't declare success prematurely." A `TODO` is an undone task in shipped code.

**Suggest:** either finish the TODOs in this PR, or move them to a tracking issue and remove the comment. Never merge a `TODO` into main unless it's a genuinely deferred non-blocker tracked elsewhere.

---

## When to flag as Blocking vs Suggested vs Noted

The tier assignment is the review's most consequential choice. Default heuristics:

**Blocking:**
- Violates rule 2 (scope creep, silent renames, adjacent improvements).
- Violates rule 5 (claims done without verification, in code that needs it).
- Security issue (SQL injection, hardcoded secret, missing auth).
- Obvious correctness bug — not a style issue, an actual wrong result.

**Suggested:**
- Naming violations (rule 3) — in new code.
- Simplicity violations (rule 1) in code that's short enough to inline-fix.
- Documentation issues (rule 6).

**Noted:**
- Pre-existing issues Claude is forbidden from cleaning up (rule 2).
- Opinions that don't trace to a specific CLAUDE.md rule.
- Deferred architectural concerns.

If in doubt between Blocking and Suggested, downgrade. A loud review with many Blockings trains reviewers to ignore the tier. A concise review with Blocking reserved for real blockers keeps the signal.
