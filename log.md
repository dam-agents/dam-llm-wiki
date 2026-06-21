# Wiki log

Chronological, append-only log of what happened and when. Newest entry last.
Entry format:

`## [YYYY-MM-DD] <onboard|ingest|lint|query> | <subject>`

## [2026-06-19] onboard | dam-agents/dam @662ebe4
## [2026-06-19] ingest | dam-agents/dam @662ebe4 — eager first pass: 8 source pages, 7 concept pages, 7 entity pages; watermark set to 662ebe4.
## [2026-06-20] ingest | dam-agents/dam @4a48ae2 — delta from 662ebe4 (1 commit: unify provider definitions). Refreshed connections-and-contributions (new provider-preset registry note), re-pinned api-server/cli/ui source pages to 4a48ae2; watermark bumped.
## [2026-06-20] lint | re-pinned dam.md overview to @4a48ae2 (packages/ glob covered the provider-definitions refactor; module map unchanged). No contradictions, orphans, or broken links. All other pages left at their pinned commits.
## [2026-06-21] ingest | dam-agents/dam @0158357 — delta from 4a48ae2 (1 commit: unbreak from-scratch cluster install / keycloak theme type-check). Re-pinned dam.md overview to @0158357 (package.json keycloakify packageExtension); enriched entities/keycloak.md Theme section with keycloakify build (vite build && keycloakify build) and the pnpm peer-dep extension. Watermark bumped.
## [2026-06-21] lint | clean sweep at @0158357 — verified staleness across all 22 pages (no drift beyond ingest's dam.md/keycloak.md refresh), 0 orphans, 0 broken links, no contradictions. No coverage gaps surfaced.
