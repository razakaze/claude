# Claude Skills Bundle

Seven Claude Code skills plus a team-wide `CLAUDE.md`, tuned for a Java 17 reactive sensor pipeline stack and a .NET 10 / C# 14 / Visual Studio 2026 stack with Avalonia, ASP.NET Core, EF Core + SQLite, and Windows Service hosting. Plus a language-agnostic code-review skill that applies the CLAUDE.md rules as review criteria.

## Contents

```
claude-skills-bundle/
├── README.md                          (this file)
├── CLAUDE.md                          (team-wide coding rules)
└── skills/
    ├── java-sensor-pipeline/          (Java 17, Spring WebFlux, Reactor Netty)
    ├── csharp-sensor-pipeline/        (.NET 10, Kestrel + Channel<T>)
    ├── csharp-windows-service/        (.NET 10 Worker SDK, SCM hosting)
    ├── csharp-avalonia/               (Avalonia 11 + CommunityToolkit.Mvvm)
    ├── csharp-rest-aspnetcore/        (Minimal API + OAuth2 JWT)
    ├── csharp-efcore-sqlite/          (EF Core 10 + SQLite)
    └── code-review/                   (self-review + PR review, CLAUDE.md as criteria)
```

Each skill is a folder containing `SKILL.md` and a `references/` directory with deeper guidance loaded on demand.

---

## Where to unzip

### Skills

Claude Code loads skills from a scan directory. The location depends on whether you want the skills available across all your projects (global) or only in one (project-local).

**Global — recommended for most users:**

Linux and macOS:
```
~/.claude/skills/
```

Windows:
```
%USERPROFILE%\.claude\skills\
```

Unzip the bundle somewhere temporary, then move the contents of the `skills/` folder into your Claude Code skills directory. After moving, the layout should look like:

```
~/.claude/skills/
├── java-sensor-pipeline/
├── csharp-sensor-pipeline/
├── csharp-windows-service/
├── csharp-avalonia/
├── csharp-rest-aspnetcore/
└── csharp-efcore-sqlite/
```

**Project-local — when the skills apply to one repo only:**

```
<your-repo>/.claude/skills/
```

Same layout as above, just inside the repo.

The exact scan path can vary between Claude Code versions. If the skills don't trigger after install, check your Claude Code configuration (`claude --help` or your IDE settings) for the active skills directory and adjust.

### CLAUDE.md

`CLAUDE.md` goes at the root of each repository where you want the rules to apply. Claude Code reads it on session start and keeps the content in context throughout the session.

```
<your-repo>/CLAUDE.md
```

If you want the rules applied globally across every project, put a copy at `~/.claude/CLAUDE.md` (or the platform equivalent). Project-local beats global when both exist.

---

## How to apply (installation steps)

1. **Unzip** the bundle somewhere convenient: `unzip claude-skills-bundle.zip`.

2. **Copy the skills** to your scan directory:
   ```bash
   # Linux / macOS
   mkdir -p ~/.claude/skills
   cp -r claude-skills-bundle/skills/* ~/.claude/skills/

   # Windows PowerShell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills"
   Copy-Item -Recurse "claude-skills-bundle\skills\*" "$env:USERPROFILE\.claude\skills\"
   ```

3. **Copy `CLAUDE.md`** to the root of each repo where you want the rules applied:
   ```bash
   cp claude-skills-bundle/CLAUDE.md /path/to/your/repo/CLAUDE.md
   ```

4. **Restart Claude Code.** Skill frontmatter is loaded at session start; adding skills mid-session won't retroactively make them visible.

5. **Verify.** In a relevant repo, ask something that should trigger a skill (see "Verification prompts" below). If the skill's guidance shows up in the response, it's working.

---

## How skills are loaded (brief)

Each `SKILL.md` has a YAML frontmatter block with `name` and `description`. Claude Code loads only the frontmatter at session start — the body of `SKILL.md` and every `references/*.md` file stays on disk until Claude decides the skill applies to your current task.

This means:

- **Unused skills are effectively free.** Frontmatter is small; having ten installed costs little.
- **Triggering is driven by the `description`.** If a skill under-triggers for you, the description is the knob to tune.
- **References are lazy.** `SKILL.md` points to them ("see `references/transports.md`") and Claude reads them only when relevant.

---

## Verification prompts

After install, try each of these in a matching project. If the right skill loads, you should see its architectural vocabulary in the response without having prompted it.

| Prompt | Expected skill |
|---|---|
| "Add a new UDP-based temperature sensor to the pipeline" | `java-sensor-pipeline` (in a Java project) or `csharp-sensor-pipeline` (in a .NET project) |
| "Create a BackgroundService that polls an API every 30 seconds" | `csharp-windows-service` |
| "Add a settings view to my Avalonia app" | `csharp-avalonia` |
| "Add a PUT endpoint for updating orders with validation" | `csharp-rest-aspnetcore` |
| "Add a migration for an IsArchived column on Orders" | `csharp-efcore-sqlite` |
| "Review this pull request" or pasting a diff with "any issues?" | `code-review` |

If the wrong skill triggers, or no skill triggers, open the skill's `SKILL.md` and tighten the `description` — then restart your Claude Code session.

---

## What each skill covers (one line each)

- **java-sensor-pipeline** — Java 17 / Spring WebFlux / Reactor Netty with a five-stage pipeline (integration → protocol → normalization → alarm → final), YAML-seeded SQLite config, per-pipeline context cache, in-process Orchestration Manager hand-off.

- **csharp-sensor-pipeline** — Mirrors the Java skill architecturally. Kestrel `ConnectionHandler` + `System.IO.Pipelines` for TCP, `Channel<T>` for boundaries that need decoupling, direct method calls everywhere else.

- **csharp-windows-service** — .NET 10 Worker SDK, `BackgroundService` patterns, Event Log, `sc.exe` install scripts, graceful shutdown, recovery configuration.

- **csharp-avalonia** — Avalonia 11 with `CommunityToolkit.Mvvm` partial properties (8.4+), compiled bindings mandatory, DI via `Microsoft.Extensions.DependencyInjection`, `ViewLocator` through DI.

- **csharp-rest-aspnetcore** — Minimal API as default with an explicit decision rule for when to switch to Controllers. `TypedResults`, `AddValidation()` (new in .NET 10), `ProblemDetails`, JWT bearer, URL path versioning.

- **csharp-efcore-sqlite** — EF Core 10 with SQLite-specific guidance (decimal-as-TEXT, WAL mode, table-rebuild limits), `IDbContextFactory<T>` for long-running hosts, `AsNoTracking` default for reads, SQLite in-memory for tests (not the InMemory provider).

- **code-review** — Two modes in one skill. Mode A fires automatically after any coding turn — a silent final-pass check against CLAUDE.md rules, with a short flag if anything's off. Mode B fires on explicit request (paste + "review this", PR links, "any issues?") and produces a structured report with Blocking/Suggested/Noted severity tiers. Loads language-specific skills when the file type indicates them.

---

## Notes

### The two sensor pipeline skills are deliberately parallel

`java-sensor-pipeline` (Java) and `csharp-sensor-pipeline` (.NET) share the same five stages, the same YAML → SQLite rule, and the same per-pipeline context cache scoping. A team working across both stacks can read one skill and recognize the architecture in the other.

### Some guidance is repeated across the C# skills

C# idioms (primary constructors, `required` members, collection expressions, `field` keyword) appear briefly in multiple skills rather than being extracted into a shared base skill. This is deliberate — triggering precision beats deduplication. If one idiom becomes obsolete or wrong, search and update across all relevant skills.

### CLAUDE.md scope

The bundled `CLAUDE.md` is language-agnostic — prompt rules and code rules that apply across Java, C#, and any other language. It is not .NET or Java specific. Skills carry the language-specific guidance.

### Iteration

Skills that under-trigger or over-trigger are common in the first week. The fix is almost always editing the `description` in `SKILL.md` — make it more specific (name the libraries, name the concepts), or broader (add synonyms). Restart Claude Code after edits.

---

## Troubleshooting

**Skills don't appear to load at all.**
Check your Claude Code skills directory. The default can vary; consult your Claude Code version's documentation. Also confirm the skill folders are directly under the scan path (not nested in an extra wrapper directory).

**Wrong skill triggers for a given prompt.**
Edit the losing skill's `description` to include the missing keyword, or the winning skill's `description` to exclude the unrelated phrasing. Restart your session.

**The skill loads but the guidance feels wrong for my codebase.**
The skills encode architectural opinions. If a specific opinion doesn't fit your project, edit the `SKILL.md` — these are meant to be team-owned, not static. Keep the pattern (main file + references) but substitute your conventions.

**`CLAUDE.md` feels too opinionated or doesn't match your team.**
Treat it as a starting draft. Every rule in it was picked for a reason that should be visible when you read it; adjust or drop rules that don't fit. The split between PROMPT rules and CODING rules is the main structural idea — keep that, vary the content.
