---
source: dam-agents/dam
commit: d34c21a008d3b868fc260838374836ac88fb0807
files: [packages/agent-runtime-api/src/modules/skills/source-roots.ts, packages/agent-runtime/src/modules/skills/infrastructure/local-skill-repository.ts, packages/agent-runtime/src/modules/skills/services/scan.ts, packages/api-server/src/modules/skills/infrastructure/public-archive-scanner.ts, docs/architecture/skills.md]
updated: 2026-06-24
---

# Skill resolution — which folders DAM scans for skills

When you point DAM at a git source to import skills (`scan`), or install a named
skill from one (`install`), the [agent-runtime](../sources/agent-runtime.md)
`skills/` module decides which directories count as skills. A directory is a
skill iff it contains a `SKILL.md`
(`packages/agent-runtime/src/modules/skills/infrastructure/local-skill-repository.ts:277 @8fa9761`).

## Source roots — three deliberate roots, top-level only as fallback

As of [#1745](https://github.com/dam-agents/dam/pull/1745) (`@8fa9761`), the set
of locations a source repo may hold skills under is a single shared, ordered
constant — `SKILL_SOURCE_ROOTS`:

1. `skills/`
2. `.claude/skills/`
3. `.agents/skills/`

(`packages/agent-runtime-api/src/modules/skills/source-roots.ts:1 @8fa9761`,
re-exported from the `agent-runtime-api` package at `src/index.ts:48 @8fa9761`).

`findSkillDirsInClone` **unions** the skill dirs found across all three roots, in
order, and only falls back to scanning the repo root (`<repo>/*`) when none of
the three deliberate roots yielded anything
(`local-skill-repository.ts:253-260 @8fa9761`). So a repo that organises under
`skills/` is never polluted by unrelated top-level dirs that happen to carry a
`SKILL.md`, while a repo mixing layouts surfaces every skill. Discovery is one
level deep per root — there is no recursive walk
(`local-skill-repository.ts:262-282 @8fa9761`).

**Dot-directories are skipped *within* each root.** `skillDirsUnder` does
`if (!ent.isDirectory() || ent.name.startsWith(".")) continue`
(`local-skill-repository.ts:274 @8fa9761`), so a hidden child folder is never
entered — but the dot-prefixed roots `.claude/skills` / `.agents/skills` are
reached fine, because they are joined onto `repoDir` as the root path to read,
not enumerated as children.

## Dedupe by name — first root wins

The union may contain same-name skills from different roots (e.g. a repo that
symlinks `.claude/skills` → `.agents/skills`). `runScan` collapses them with
`dedupeByName`: it keeps the **first** occurrence in root order and drops later
same-name entries, logging each drop
(`packages/agent-runtime/src/modules/skills/services/scan.ts:44-50 @8fa9761`,
`source-roots.ts:12-25 @8fa9761`). Name — `name` frontmatter, else the directory
basename — is the identity because the on-pod store is a flat name-keyed
directory and the catalog row is keyed on name, so two same-named dirs are not
separately representable. A `skills/foo` therefore shadows a `.claude/skills/foo`
(`docs/architecture/skills.md:96-102 @8fa9761`). Only real directories are
scanned, so per-skill symlinks are still skipped.

## Install (resolve one named skill) — same root order

`resolveSkillDirInClone` walks the same precedence: it tries
`skills/<name>/`, `.claude/skills/<name>/`, `.agents/skills/<name>/`, then the
top-level `<name>/`, returning `SkillNotFoundInSource` if none holds a `SKILL.md`
(`local-skill-repository.ts:284-299 @8fa9761`).

## One source of truth, three call sites

The ordering lives in one place (`SKILL_SOURCE_ROOTS` in `agent-runtime-api`) so
the three discovery paths can never disagree on precedence: the agent-runtime
clone scan, the agent-runtime install resolver (both above), and the api-server's
**public-archive scan** — the credential-free `github.com` tarball path that lets
the catalog UI render without a running agent — which imports the same constant
and `dedupeByName`
(`packages/api-server/src/modules/skills/infrastructure/public-archive-scanner.ts:6 @8fa9761`,
`:119-121 @8fa9761`, `:219 @8fa9761`).

## Locally-installed skills

`listLocal`/`readLocal` don't scan a repo — they read the absolute skill
directories passed in as configured `skillPaths`, resolving `<base>/<name>/SKILL.md`
and skipping dot-entries the same way
(`local-skill-repository.ts:101-104 @8fa9761`).

## History: `.claude/skills/` used to be invisible

Before #1745, scan and install checked only two roots — `skills/*` then a
top-level `*` fallback — and a hidden `.claude/skills/` was never found (the
`startsWith(".")` skip also blocked reaching `.claude`). A common convention
(Claude Code itself, e.g. `dam-agents/spotify-dj`) keeps skills there, so those
imports reported zero skills found. [#1637](https://github.com/dam-agents/dam/issues/1637)
tracked the fix; #1745 added `.claude/skills/` and `.agents/skills/` as recognised
roots. (The clearer "no skills found" messaging remains deferred to #944.)

## See also

- [agent-runtime](../sources/agent-runtime.md) — the pod process whose `skills/` module owns scan/install/publish.
- [api-server](../sources/api-server.md) — hosts the public-archive scan that shares the same source-root ordering.
