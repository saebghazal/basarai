# basarai

# basarai

hi

## Development workflow

Features are built phase by phase from the Spec Kit task lists in `specs/<feature>/tasks.md`.
OpenCode implements; Claude Code reviews.

### Per-phase loop

1. Start from a clean tree: `git status` shows no changes.
2. Implement the phase with OpenCode (`/speckit.implement`).
3. Commit the phase on its own, e.g. `feat(003): phase 1 setup (T001–T006)`.
4. Review that commit with Claude Code (`git diff HEAD~1` or `git diff main...HEAD`).
5. Commit review fixes separately, e.g. `fix(003): phase 1 review fixes`.
6. Move to the next phase.

### Rules

- **One phase per commit.** Every changed file must belong to a task in that phase. Check with
  `git status --short` before committing.
- **Tooling changes stand alone.** Run `specify init`, Spec Kit upgrades, and integration changes
  only on a clean tree, between phases, and commit them immediately as a separate `chore:` commit.
  Never mix `.specify/`, `.opencode/`, or `.claude/` changes with `apps/` or `packages/` changes.
- **Record what was touched.** Execution notes in `tasks.md` list the files changed per task and
  any task left unchecked, with the reason.
- **Unverified tasks stay unchecked.** A task is marked `[X]` only after its stated verification
  passes.
