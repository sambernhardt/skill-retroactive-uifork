---
name: retroactive-uifork
description: Build a retroactive UIFork timeline — take existing git commit history for a component and materialize it as browsable UIFork versions (vs main or a discovered base), so past milestones are reviewable in-context without checking out old SHAs.
triggers:
  - retroactive uifork
  - uifork from git history
  - commit history to uifork
  - browsable ui timeline from commits
  - turn commits into uifork versions
  - replay branch ui in uifork
requires: git, uifork npm package (`pnpm` / `npm` in the target app), pnpm workspace with target app; optional UIFork agent skill (see below)
---

# Retroactive UIFork (git history → browsable versions)

## Why this exists

**UIFork** is usually **forward-looking**: you fork components to explore **new** ideas before promoting one. This skill covers the **retroactive** case: the work **already landed** as a sequence of commits on a branch, and you want that **history** to be **navigable** in the running app—same route, same props, same data—by flipping versions in the UIFork widget instead of reading `git log`, raw diffs, or checking out each commit.

**Use it when** you need design or eng review of **how a UI evolved**, stack-ranking polish steps, or a walkthrough of a feature branch **as it was built**—not when you are only sketching new variants from scratch (use the **UIFork** tooling and agent skill below on its own).

**What you get:** each chosen commit (plus an optional baseline at `BASE`) becomes a **real file snapshot** (`git show <ref>:path`) wired into **`*.versions.ts`** with **labels** derived from commit subjects, so the timeline is **browsable** and **labeled**, not a pile of opaque `v3` ids.

Implementation details (CLI, `ForkedComponent`, watch server) still follow **UIFork** upstream; this document is the **git + curation + snapshotting** workflow on top.

---

## Companion — UIFork agent skill (optional)

This retroactive workflow assumes you can look up **UIFork** CLI usage, init layout, and app wiring. Many setups already have the **UIFork** skill installed globally for agents.

**If it is not available**, add it from the upstream registry:

```bash
npx skills add sambernhardt/uifork
```

Use that skill for `init`, `watch`, `ForkedComponent`, Next.js root mounting, and promotion. The **`uifork`** package itself is still installed in the app (`pnpm add uifork` / workspace dependency) regardless of the agent skill.

---

## Reference — UIFork contract (from upstream)

When executing the implementation phases, use the **UIFork** skill or [UIFork on GitHub](https://github.com/sambernhardt/uifork) / npm for:

- **CLI:** `init`, `watch`, `new`, `fork`, `rename`, `delete`, `promote`
- **File layout after init:** wrapper entry file, `*.versions.ts`, `Component.vN.tsx` version files
- **Next.js App Router:** `<UIFork />` at app root; **dev-only** (do not ship the widget to production users)
- **Watch server:** regenerates `*.versions.ts` when version files change; run after adding or renaming `*.vN.tsx` files manually

This **retroactive** workflow adds **git archaeology**, **commit curation**, **`git show` extraction**, **labels**, and **timeline docs**; UIFork itself is generic.

---

## Goals and success criteria

Reviewing work only through `git log` or file diffs is slow and loses **in-context** comparison (same page, same props, same data).

**Goal:** Turn **past** commits into a **browsable, chronological set of real component files** inside the running app so people flip milestones **without** checking out branches—**retroactively** wiring **UIFork** as the switcher.

**What “good” looks like:**

- **v1** (optional but common) shows the **baseline** at `BASE`, usually `git show BASE:$TARGET` (e.g. `main` before the arc).
- Later versions each match **`git show <commit>:$TARGET`** for the chosen commits—**faithful snapshots**, not hand-replayed diffs.
- The floating UIFork UI lists each step with a **short, recognizable `label`** (from the commit message or a tightened phrase), not only opaque ids like `v4`.
- A small **`*.UIFORK-TIMELINE.md`** beside the component documents purpose, hashes, labels, and reproducible `git show` commands.

---

## What you need from the user (gather early)

Ask or infer; don’t block on perfection—**refine after Phase 1**.

| Input | Why |
| ----- | --- |
| **Base branch / `BASE`** | Usually `main`; sometimes the branch was cut from another branch or is stacked. Default: `main`. |
| **Target** | One file (e.g. `Button.tsx`) or a small set. If unknown, derive from diff (Phase 2). |
| **Optional: commit filter** | All commits on the branch vs only commits touching the target file vs a hand-picked milestone list. |

---

## Phase 1 — Establish what to compare (due diligence)

**Start with:** current branch vs **`main`** (or the user’s stated base).

```bash
# Commits reachable from HEAD but not from main
git log main..HEAD --oneline

# How many files changed?
git diff main...HEAD --stat

# Merge base (where this branch forked from main’s history)
git merge-base main HEAD
```

**Interpret:**

- **Small / focused diff** (few files, clear feature area): comparing to **`main`** is probably correct. Proceed.
- **Huge diff** (hundreds of files, unrelated areas), or **merge-base looks wrong** for the story the user tells: the work may not “start” at `main` in a meaningful way—e.g. branch stacked on another feature branch, or long-lived integration branch.

**If comparison to `main` is misleading, investigate (read-only):**

```bash
# First commit on this branch (oldest unique commit)
git rev-list --reverse main..HEAD | head -1
git show -1 --stat <that-hash>

# Sometimes helpful: log with graph
git log --oneline --graph --decorate -20 main HEAD
```

**Ask the user** when automation isn’t enough:

- “Was this branch cut from **`main`**, or from another branch (e.g. a parent feature)?”
- “What **merge-base** or **parent PR** should we treat as ‘before my changes’?”

**Deliverable of Phase 1:** a **comparison ref** `BASE` (often `main`, sometimes `origin/main`, sometimes a commit hash or another branch) such that `BASE..HEAD` is the history you care about for *this* story.

---

## Phase 2 — Scope the UI / file target

**If the user already named a file or component:** use that path.

**Else, derive from diff:**

```bash
git diff BASE...HEAD --stat
git log BASE..HEAD --oneline -- path/to/suspected/file.tsx
```

- **One file dominates** the narrative (many commits, large churn): propose it as the UIFork target.
- **Many files with similar weight:** **stop and ask**: “Which component or file should the timeline track?” Offer top candidates from `--stat` sorted by churn.

**Multi-file UI:** Prefer **one primary file** for UIFork (the shell consumers import). Related layout can stay outside the fork; document that in `*.UIFORK-TIMELINE.md`.

**Stable path for `git show`:** always use the **repo-relative path** as Git stores it (same string for `git show <ref>:path`):

```bash
REL='apps/<app>/.../YourComponent.tsx'
git show "BASE:$REL"
git show "<commit>:$REL"
```

**Deliverable of Phase 2:** a **stable repo-relative path** `TARGET`.

---

## Phase 3 — Curate commits for the timeline

**Full chronological list:**

```bash
git rev-list --reverse BASE..HEAD
```

**Commits that actually touched the target file:**

```bash
git log BASE..HEAD --oneline -- "$TARGET"
```

Decide with the user:

- **Every commit on the branch** (strict timeline; some steps may not change `TARGET` → duplicate snapshots for “state after commit *k*”).
- **Only commits touching `TARGET`** (fewer versions; each step changes that file).
- **Milestone subset:** user picks which hashes matter (e.g. only “sidebar / buttons / spacing” commits); optional **v1 = baseline** at `BASE`, then **v2…vN** = end state after each selected commit.

**Naming:**

- **v1** often = **baseline** at `BASE`: `git show BASE:$TARGET`.
- **v2…vN** = `git show <commit>:$TARGET` at each chosen commit (tree **after** that commit).

Record a **ledger**: version id, full or short hash, commit subject, **label** for the widget, and optional note if a commit didn’t touch `TARGET` (snapshot identical to previous).

**Caveat — baseline vs branch:** `git show BASE:$TARGET` may **differ sharply** from later versions (structure, imports, props, exports). The forked **wrapper** must still satisfy what parent files import (`Component`, `Skeleton`, types). Options: thin shims, re-export types from the **shipping** version file, or treat baseline as **preview-only** if types cannot align.

---

## Phase 4 — Implement with UIFork

### Numbered steps (matches a typical project run)

1. **Ledger doc** — Write `Component.UIFORK-TIMELINE.md` (or section in a shared doc): purpose, `BASE`, table of versions → hash → **label** → commit subject, and copy-pastable `git show` examples using `REL`.

2. **Initialize UIFork** — From the app package (e.g. `apps/vercel-site`):

   ```bash
   pnpm exec uifork init "path/from/app/root/to/Component.tsx" --export ComponentName
   ```

   Add **`v1`…`vN`** via `uifork new` / `fork` or by creating `Component.vN.tsx` files. **Preserve public exports** importers rely on (often more than one named export).

3. **Fill versions from git** — For each version row:

   ```bash
   git show "BASE:$REL"        # v1 when baseline = BASE
   git show "<hash>:$REL"      # v2…vN
   ```

   Prefer **no hand-replayed diffs** unless a snapshot fails typecheck or build.

4. **Sync `*.versions.ts`** — Run **`pnpm exec uifork watch`** from the appropriate directory after adding/renaming version files, **or** edit `*.versions.ts` by hand if your workflow matches (preserve `@uifork-export` and any **manual labels** the tool would overwrite—re-run watch only when safe).

5. **Apply labels** — Set human-readable **`label`** on each version in `*.versions.ts` (`VersionType` in `uifork@0.0.14`). Optionally use **`pnpm exec uifork rename`** for version ids (e.g. sub-versions `v1_1`). **Mirror** the same labels in the TIMELINE markdown so repo readers see the mapping without running the app.

6. **Wrapper / exports** — If the CLI only forks one export, restore the full public API: extra components (e.g. **skeleton**), **types**. If the UI has both **card and skeleton**, you may need **two** `ForkedComponent` trees with the **same** `id` (e.g. `bot-traffic-card`) so the widget keeps them on the same selected version—see implementation notes in your app. Use **`ForkedComponent`** generics / `as unknown as` only where uifork’s `Record<string, unknown>` props require it.

7. **App wiring** — Ensure **`UIFork`** is mounted at the app root (often `app/layout.tsx`) via a small client component (e.g. `uifork-root.tsx`). Depend on **`uifork`** in that app’s `package.json`.

### Version labels (commit message → widget copy)

1. **CLI:** `pnpm exec uifork rename <path-or-name> <fromId> <toId>` when renaming version ids (sub-versions like `v2_1` may display as “V2.1” per upstream docs).
2. **Config file:** After `init` / `watch`, inspect **`*.versions.ts`**. Set **`label`** (and **`description`** if present) per version; verify against the installed `uifork` types.
3. **Timeline doc:** Duplicate labels in the markdown ledger.

**Example label pattern** (tighten per project):

| Version | Example label | Tied to |
| ------- | ------------- | ------- |
| v1 | Main (baseline) | `git show BASE:$TARGET` |
| v2… | Short phrase | Commit subject or trimmed variant |

### Production gating

Do **not** render `<UIFork />` in production for end users. Typical pattern:

```tsx
if (process.env.NODE_ENV === 'production') return null;
return <UIFork />;
```

Apply in the root `UIFork` provider component (e.g. `uifork-root.tsx`).

---

## Phase 5 — Verify

- **Typecheck:** `pnpm type-check -F <app>` or `turbo -F <app> type-check` (monorepo convention).
- **Manual:** Run dev, navigate to the feature page, open the UIFork widget, walk **v1 → vN** in order; confirm **labels** and visual steps match the ledger.
- **Baseline:** If **v1** is from `BASE` and diverges structurally from **vN**, expect possible type friction; document in TIMELINE or add minimal shims.

---

## Quick command reference

| Goal | Command |
| ---- | ------- |
| Commits not in base | `git log BASE..HEAD --oneline` |
| Oldest first | `git rev-list --reverse BASE..HEAD` |
| File at commit | `git show <hash>:path/to/File.tsx` |
| File at base | `git show BASE:path/to/File.tsx` |
| Who touched file | `git log BASE..HEAD --oneline -- path/to/File.tsx` |

---

## Appendix — Example case study (non-normative)

**Bot traffic card** (`vercel-site`): timeline **v1** = `main` snapshot of `bot-traffic-card.tsx`; **v2–v7** = six milestone commits on a polish branch (sidebar/labels, buttons, others category, dot→line, spacing, button transition). Each row used **`git show <hash>:$REL`**; labels like “Sidebar & labels”, “Dot → line”, “Button transition”. Wrapper uses **`ForkedComponent`** for both **`BotTrafficCard`** and **`BotTrafficCardSkeleton`** with the same **`id`** so the picker stays in sync; types re-exported from the shipping version file (`v7`). **`UIForkRoot`** gated for production. See the component directory for `bot-traffic-card.UIFORK-TIMELINE.md` and `*.versions.ts` as a concrete reference.

---

## When not to use this

- Production feature flags or long-lived A/B tests (use real flags).
- **Multiple unrelated components** in one fork—split timelines or separate forked entrypoints.
- **Greenfield exploration** with no meaningful commit history to replay—use **UIFork** alone (agent skill or package docs).

---

## Revision notes (draft)

- Formerly drafted as **`uifork-commit-timeline`**; renamed to emphasize **retroactive** use of UIFork.
- Canonical source for this skill: standalone repo **`skill-retroactive-uifork`** (`skills/retroactive-uifork/SKILL.md`).
- Tighten **triggers** once this ships.
- Point appendix to a stable internal path or remove if the example moves.
- Track **uifork** semver notes (`*.versions.ts` hand-edits vs `watch`).
