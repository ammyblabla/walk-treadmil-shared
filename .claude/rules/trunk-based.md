# Trunk-based development

This repo uses trunk-based development. The trunk is `main`.

- Commit directly to `main`. Do not create feature branches or pull requests.
- Keep each commit small and self-contained, and leave `main` working: tests pass, the app builds.
- If a change is too big for one commit, split it into small steps. Hide unfinished work behind a flag or keep it unwired. Do not use a long-lived branch for it.
- Before committing, run the repo's tests and lint. Do not commit if they fail.
- Commit only when asked. Push only when asked.
- Use clear commit messages in the form (conventional commit) `type: summary`, e.g. `feat: add plan adjust endpoint`, `fix: …`, `docs: …`, `test: …`, `chore: …`.
