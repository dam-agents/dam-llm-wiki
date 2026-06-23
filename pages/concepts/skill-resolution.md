---
source: dam-agents/dam
commit: cd63253da5a7b70cae650a68428cec963f29dec8
files: [packages/agent-runtime/src/modules/skills/infrastructure/local-skill-repository.ts, packages/agent-runtime/src/modules/skills/services/scan.ts]
updated: 2026-06-23
---

# Skill resolution — which folders DAM scans for skills

When you point DAM at a git source to import skills (`scan`), or install a named
skill from one (`install`), the [agent-runtime](../sources/agent-runtime.md)
`skills/` module decides which directories count as skills. A directory is a
skill iff it contains a `SKILL.md`
(`packages/agent-runtime/src/modules/skills/infrastructure/local-skill-repository.ts:266 @cd63253`).

## Scan (enumerate a source) — two candidate roots, first hit wins

`findSkillDirsInClone` checks exactly two roots, in order, and returns the
**first** that yields any skill:

1. `<repo>/skills/*` — a top-level `skills/` folder
2. `<repo>/*` — skill dirs loose at the repo root

(`local-skill-repository.ts:254 @cd63253`; declared "Search order: `skills/*`
first, fall back to top-level `*`" at `local-skill-repository.ts:57 @cd63253`).
Because the fallback root only runs when `skills/` produced nothing
(`local-skill-repository.ts:272 @cd63253`), a repo that has *both* a populated
`skills/` and root-level skill dirs will only surface the `skills/` ones.

**Dot-directories are skipped at both roots.** The scan loop does
`if (!ent.isDirectory() || ent.name.startsWith(".")) continue`
(`local-skill-repository.ts:263 @cd63253`), so a hidden folder is never entered.

## Install (resolve one named skill) — same two locations

`resolveSkillDirInClone` tries `<repo>/skills/<name>/` then `<repo>/<name>/`,
returning `SkillNotFoundInSource` if neither holds a `SKILL.md`
(`local-skill-repository.ts:280 @cd63253`).

## Locally-installed skills

`listLocal`/`readLocal` don't scan a repo — they read the absolute skill
directories passed in as configured `skillPaths`, resolving `<base>/<name>/SKILL.md`
and skipping dot-entries the same way
(`local-skill-repository.ts:96 @cd63253`, `:102 @cd63253`).

## Consequence: `.claude/skills/` is not discovered

A common convention (Claude Code itself, e.g. `dam-agents/spotify-dj`) keeps
skills under a hidden `.claude/skills/` folder. As of `@cd63253` DAM does **not**
find these, for two compounding reasons: `.claude/skills` is not one of the two
candidate roots, and even reaching `.claude` is blocked by the
`startsWith(".")` skip. The import reports zero skills found with no hint why.
[dam-agents/dam#1637](https://github.com/dam-agents/dam/issues/1637) tracks
adding `.claude/skills/` as a recognised location (with the clearer
"no skills found" messaging deferred to #944).

## See also

- [agent-runtime](../sources/agent-runtime.md) — the pod process whose `skills/` module owns scan/install/publish.
