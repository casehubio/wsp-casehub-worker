# worker Workspace

**Name:** worker
**Project repo:** `/Users/mdproctor/claude/casehub/worker`
**Workspace:** `/Users/mdproctor/claude/public/casehub/worker`
**Workspace type:** public

## Session Start

Run `add-dir /Users/mdproctor/claude/casehub/worker` and `add-dir /Users/mdproctor/claude/public/casehub/worker` before any other work.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| java-update-design / update-primary-doc | `design/JOURNAL.md` (created by `work-start`) |
| adr | `adr/` |
| write-content | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md (auto-pruned, max 10)
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md
- `design/` — epic journal (created by `work-start` at branch start)

## Git Discipline

Two git repositories are active in every session:
- **Workspace** (`/Users/mdproctor/claude/public/casehub/worker`) — plans, blog (staging), snapshots, handover
- **Project repo** (`/Users/mdproctor/claude/casehub/worker`) — source code, ADRs (`docs/adr/`), specs

Never rely on CWD for git operations — the session may have started in either repo. Always use explicit paths:
```bash
git -C /Users/mdproctor/claude/public/casehub/worker ...    # workspace artifacts
git -C /Users/mdproctor/claude/casehub/worker ...           # project artifacts
```
The file path determines the repo: if the file lives under the workspace path, use the workspace; if under the project path, use the project.

## Rules

- All methodology artifacts go here, not in the project repo
- Promotion to project repo is always explicit — never automatic
- Workspace branches mirror project branches — switch both together

## Routing

| Artifact   | Destination | Notes |
|------------|-------------|-------|
| adr        | project     | lands in `docs/adr/` — promoted at work end |
| specs      | project     | lands in `docs/specs/` — promoted at work end |
| blog       | workspace   | staged here; published to mdproctor.github.io via publish-blog at work end |
| plans      | workspace   | stay in workspace permanently |
| design     | workspace   | epic journal stays in workspace |
| snapshots  | workspace   | stay in workspace permanently |
| handover   | workspace   | |

**Blog directory:** `/Users/mdproctor/claude/public/casehub/worker/blog/`

## Context Management

If the conversation is getting very long or you notice context pressure,
proactively suggest writing a handover before continuing.

---

# CaseHub Worker

Foundation-tier automated task primitives — Worker, WorkerFunction, Capability, execution policy.

## Project Type

type: java

## Build & Test

```bash
# Full build
mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml clean install

# Test only
mvn -f /Users/mdproctor/claude/casehub/worker/pom.xml test

# Single module
mvn -f /Users/mdproctor/claude/casehub/worker/<module>/pom.xml test
```

## Structure

Multi-module Maven project (Quarkus 3.32.2, Java 21):
- `api` — Worker, WorkerFunction, Capability, WorkerResult, WorkerOutcome
- `runtime` — WorkerExecutor with PolicyEnforcer + OTel tracing
- `testing` — MockWorkerExecutor + TestWorkerBuilder

## Repo

- Org: casehubio
- Repo: casehub-worker
- GitHub Packages for Maven artifacts

## Work Tracking

**Issue tracking:** enabled
**GitHub repo:** casehubio/casehub-worker
**Changelog:** GitHub Releases (run `gh release create --generate-notes` at milestones)

**Automatic behaviours (Claude follows these at all times in this project):**
- **Before implementation begins** — when the user says "implement", "start coding",
  "execute the plan", "let's build", or similar: check if an active issue or epic
  exists. If not, run issue-workflow Phase 1 to create one **before writing any code**.
- **Before writing any code** — check if an issue exists for what's about to be
  implemented. If not, draft one and assess epic placement (issue-workflow Phase 2)
  before starting. Also check if the work spans multiple concerns.
- **Before any commit** — run issue-workflow Phase 3 (via git-commit) to confirm
  issue linkage and check for split candidates. This is a fallback — the issue
  should already exist from before implementation began.
- **All commits should reference an issue** — `Refs #N` (ongoing) or `Closes #N` (done).
  If the user explicitly says to skip ("commit as is", "no issue"), ask once to confirm
  before proceeding — it must be a deliberate choice, not a default.

## Project Artifacts

Paths that are project content (not workspace noise). Skills use this to avoid
filtering or dropping commits that touch these paths.

| Path | What it is |
|------|------------|
| `CLAUDE.md` | Project conventions (build, test, naming) |
| `docs/adr/` | Architecture decision records |
| `docs/DESIGN.md` | Design document |
