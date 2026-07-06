---
layout: post
title: "The SNAPSHOT That Wasn't There"
date: 2026-07-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-worker]
tags: [foundation-tier, maven, ci, dependency-resolution]
---

# The SNAPSHOT That Wasn't There

**Date:** 2026-07-06
**Type:** phase-update

Worker CI has been red since July 3. I hadn't noticed because the last session ended with a clean `work-end` — the code compiled locally, tests passed, the branch merged. CI was an afterthought.

The error was `cannot find symbol` for `TimeoutPolicyException` and `InterruptedPolicyException` in `DefaultWorkerExecutor`. Both classes exist. Both compile. Both are in my local `.m2`. The imports are correct, the dependency version is correct (`0.2-SNAPSHOT`), and `mvn clean install` passes without complaint.

The problem is that `0.2-SNAPSHOT` means different things in different places. My local Maven cache has the build from July 1 where I added the typed exception subclasses to `casehub-platform-governance`. GitHub Packages has an older build — SNAPSHOT #19, also from July 1 but from earlier that day, before the exception classes existed. CI pulled the published version. The classes literally weren't in the jar.

Platform CI auto-deploys on push to main and even triggers downstream repos via `repository_dispatch`. The machinery works. The gap was ordering: I pushed worker before platform's exception refactor had been deployed. By July 5 platform had deployed a newer SNAPSHOT that includes the classes — the worker CI failure was already stale. A re-run fixed it.

I considered adding a pre-push SNAPSHOT freshness check — `mvn dependency:resolve -U` to verify every SNAPSHOT dependency is available remotely before allowing a push. It would have caught this. But it adds 5–10 seconds per push and blocks intentional work-ahead, where you push a consumer knowing the upstream deploy is minutes behind. The discipline fix — push upstream first — costs nothing and matches the dependency flow. If this becomes a pattern, the automation is there to add.

The session also turned up some housekeeping. The `issue-2` project branch was missing its closure stamp — `EPIC-CLOSED.md` existed on the workspace branch but the project branch's last commit was a docs sync, not the `chore: branch closed` marker. Stamped it. The second blog entry (`mdp02 — Closing the Result Model`) was missing YAML frontmatter and had never been published to `casehub-notes`. Added frontmatter, updated INDEX.md, published to both destinations. Small gaps that accumulate when wrap hygiene skips a step.

The real lesson from the CI failure is how convincingly `cannot find symbol` points you at the wrong thing. Nothing in the error says "version mismatch." The class name is right, the package is right, the dependency is declared. Local builds pass. You have to know that SNAPSHOT resolution differs between your machine and the remote registry to even consider looking there.
