# Handoff: issue #100 — prompt templates vs skills

Status: ready to implement. This is a working artifact for the issue_100 branch, not
reviewed architecture content. Delete it once #100 is closed.

Issue: https://github.com/docs-as-code-toolkit/architecture-knowledge-toolkit/issues/100
Branch: `issue_100` (per `skills/implement-issue-workflow/SKILL.md`)
Commit prefix: `issue_100:`

## Context

The repo currently keeps two parallel layers:

- `skills/**/SKILL.md` — authoritative, engine-agnostic workflow knowledge.
- `templates/prompts/*.md` — 22 hand-authored prompt files that mostly paraphrase a
  single skill, each carrying a copy-pasted toolkit lookup block.

The prompt layer is a fork, not a separation: it restates skill rules, drifts its
lookup block, and forces users to learn that two layers exist. The harness story
backs this up — pi and OpenCode still support prompt templates, but Claude Code,
Cursor, GitHub Copilot, and Codex all steer users to skills (Codex custom prompts
are deprecated and not even repo-shareable). Prompt templates should survive only
as thin, optional invocation cards that carry a real task slot.

## Decisions (locked by the owner)

1. **Nested skills**: flatten to top-level. Do not add recursion to the adapters or
   the installer.
2. **Optional prompt install**: add a `--prompts` flag to `install-skills.sh`,
   default off.
3. **Card format**: one portable `.md` card for every harness; document harness
   limitations instead of emitting per-harness copies.
4. **Surviving cards**: keep only cards with a real input slot or mode (list below).
5. **Lookup partial**: `templates/prompt-parts/lookup.md`.
6. **Acceptance evidence**: checked-in `features/prompt-wiring.feature`, bridged to
   `test/prompt-wiring.test.mjs`.

### Survivor list (6 cards)

Keep and regenerate as thin cards:

- `apply-contracts` (task slot)
- `bootstrap-greenfield`, `bootstrap-existing-artifacts` (idea/artifact slot + modes)
- `migrate-local-contract`, `migrate-local-skill`, `migrate-local-skill-set`
  (local path slot)

Delete all 1:1 fronts: `adr`, `risk`, `quality-scenario`, `traceability-review`,
`pr-review`, `architecture-impact`, `bdd-specification`, `implement-issue-workflow`,
`commit-message`, `domain-modeling`, `slice-issues`, `post-merge-sync`,
`bootstrap-project` (generic), `grilling`, `grill-me`, `grill-with-docs`.

Litmus: keep a card only if it carries a task slot or mode the skill's own trigger
does not. No card may restate a rule owned by its skill.

## Harness prompt-template support (research result)

| Harness | Mechanism | Status |
|---|---|---|
| pi | `prompts/*.md` → `/name`, `description`/`argument-hint`/args | Current, first-class |
| OpenCode | `.opencode/commands/*.md` → `/name`, frontmatter | Current |
| Cursor | slash commands; `/migrate-to-skills` (2.4+) | Supported, migrating to skills |
| Claude Code | `.claude/commands/*.md` → `/name` | Merged into skills (same mechanism) |
| GitHub Copilot (VS Code) | `.github/prompts/*.prompt.md` → `/name` | Deprecated for Agent Host; migrate to skills |
| Codex | `~/.codex/prompts/*.md` → `/prompts:name` | Deprecated; home-dir only, not shareable via repo |
| Vibe | no documented prompt-template mechanism found; skills supported | None found |

Consequence: prompt install is opt-in and secondary; skills are the primary path.

## New finding that shapes the plan

`skills/grilling/me/SKILL.md` and `skills/grilling/with-docs/SKILL.md` already exist
as nested sub-skills, and the `grill-me` / `grill-with-docs` prompts duplicate them.
But `scripts/build-agent-adapters.js` (`listSkillFiles`) and
`adapters/shared/install-skills.sh` (the `skills/*/SKILL.md` glob) only read
top-level skills, so the nested entry points are currently unreachable natively.
The nested dir name `me` also mismatches the skill name `grill-me`.

Flattening is the simpler fix: move them to top-level so existing discovery works,
with no recursion and no dir-name/front-matter mismatch. `skills/grilling/` stays
the hidden helper. No other nested skills exist (`skills/references/` holds guides,
not skills).

## Target end state

- `skills/**/SKILL.md` (all top-level) owns all workflow and is discovered by
  description.
- `skills/grill-me/` and `skills/grill-with-docs/` are exposed entry points;
  `skills/grilling/` stays `adapter_expose: false`.
- `templates/prompts/` holds only the 6 thin generated cards:
  `description` + `argument-hint` + one-line delegation + shared lookup.
- Canonical lookup at `templates/prompt-parts/lookup.md`, outside the installable dir
  (so it is never discovered as a command).
- Prompt install is `--prompts`, opt-in, documented as secondary/legacy per harness.
- A first-time user can start from one instruction and never learn that two layers
  exist.

## Steps

### Phase 0 — Record the decision

1. Add a proposed ADR: *"Skills own workflow; prompt templates are thin, optional
   invocation cards, kept only where a task slot exists."*
   - Include the harness table and the opt-in rule.
   - Include the acceptance criterion: one instruction starts work; the user never
     learns two layers exist.
   - Mark it proposed unless the owner accepts it in the repository.

### Phase 1 — Canonical lookup + generator

1. Create `templates/prompt-parts/lookup.md` with the single canonical wording
   (replaces the three current variants).
2. Create `scripts/build-prompts.js`, modeled on `scripts/build-agent-adapters.js`:
   - reads the 6 survivor definitions plus skill front-matter;
   - emits each card with `description` and `argument-hint`;
   - excludes partials;
   - supports `--check`.

### Phase 2 — Delete the 1:1 prompt tree

1. Delete the 16 dropped prompt files listed above.
2. Regenerate the 6 survivors as thin cards.
3. Confirm no card restates a skill-owned rule.

### Phase 3 — Flatten grill skills and reconcile references

1. `git mv skills/grilling/me skills/grill-me`
2. `git mv skills/grilling/with-docs skills/grill-with-docs`
3. Keep `skills/grilling/SKILL.md` with `adapter_expose: false`.
4. Update `README.md` lines ~84–85 and ~412–414 to the new paths.
5. Fold bootstrap variants into `skills/bootstrap-project/SKILL.md` as explicit
   modes; update its "Example Prompt Workflow" section.
6. `migrate-*` / `apply-contracts`: keep as thin prompt cards only (no skill
   promotion).

### Phase 4 — Optional prompt installer

1. Add `--prompts` to `adapters/shared/install-skills.sh` (default off). Install the
   6 cards into harness prompt roots where they exist (pi, OpenCode).
2. Document Cursor/Claude/Copilot/Codex/Vibe status from the harness table.
3. Keep cards portable; keep partials out of every installed root.

### Phase 5 — Docs and references

1. Update `README.md` (~line 357) and rewrite `templates/prompts/README.md`:
   thin-card contract, generator, `--prompts`, harness status.
2. Update `adapters/shared/README.md` with `--prompts` and the legacy status.
3. Align `example/README.md` inlined prompts for the surviving flows.

### Phase 6 — Guards

1. Add `features/prompt-wiring.feature`, bridged to
   `test/prompt-wiring.test.mjs`:
   - every surviving card's delegation target resolves to a skill;
   - every card's lookup block byte-matches `templates/prompt-parts/lookup.md`;
   - no card restates a skill-owned rule;
   - every card has `description` and `argument-hint`;
   - no dropped 1:1 card exists.
2. Add `test/build-prompts.test.mjs` for the generator and `--check`.
3. Extend `test/skill-wiring.test.mjs` with a guard that every `SKILL.md` is
   top-level and discovered (catches a future nest regression).
4. Add `prompts` / `check-prompts` tasks to `build.sh`; wire `check-prompts` into
   `all`.

### Phase 7 — Validate and evidence

1. Run `./build.sh all`.
2. Regenerate adapters (`node scripts/build-agent-adapters.js`).
3. Exercise the `features/prompt-wiring.feature` scenario.
4. Walk the acceptance criterion on a scratch consumer project (skill alone plus one
   sentence of task input).
5. Record the result as evidence on #100.

## Commit slices

1. `issue_100: Record the prompt and skill boundary` (ADR)
2. `issue_100: Add canonical lookup partial and prompt generator`
3. `issue_100: Reduce prompts to cards with a task slot`
4. `issue_100: Flatten grill entry-point skills`
5. `issue_100: Install prompts optionally via --prompts`
6. `issue_100: Guard prompt wiring and top-level skill discovery`
7. `issue_100: Update README, prompt README, and example`

## Conventions to respect

- Reference, don't copy: cards reference skills through the lookup order; never
  duplicate skill rules, `features/`, or contract text.
- Preserve stable IDs; mark AI-created content draft/proposed until human acceptance
  is recorded.
- Keep changes small and reviewable; one commit slice per step.
- Prefer explicit AsciiDoc anchors for generated links; do not derive xrefs from
  artifact-id-prefixed file names.
- Run `./build.sh all` before declaring the work done.

## Open questions

None. All six decisions are locked.

## Issue comment draft (see response / post after implementation starts)

A brief summary comment is drafted in the session that produced this handoff; post it
to #100 when work starts so the owner and the workshop participants see the chosen
direction.
