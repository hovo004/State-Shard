# Contributing to state-shard

Thanks for considering a contribution! This is a small, focused bash utility, so contributions are easiest to review when they stay narrow in scope.

## Ways to contribute

- **Bug reports** — open an issue with your OS, bash version (`bash --version`), terraform version, backend type, and the exact command/flags you ran.
- **Feature requests** — open an issue describing the use case before submitting a large PR, so we can agree on the approach first.
- **Pull requests** — bug fixes, documentation improvements, and small enhancements are all welcome.

## Before submitting a PR

1. **Test against a real (non-critical) Terraform state.** Since this script mutates state, manually verify:
   - `terraform plan` shows **No changes** in both source and target after a migration.
   - Backups (`.tfstate.bak`) are created correctly.
   - The script still pauses for confirmation at every major step.
2. **Keep bash compatibility.** The script targets bash 4+ (uses `mapfile`). Avoid bash 5-only features unless necessary, and note it in the PR if you do.
3. **Update the README** if you change flags, behavior, or requirements.
4. **Keep the diff focused.** Unrelated formatting/style changes make PRs harder to review — please open a separate PR for those.

## Reporting security issues

If you find an issue that could cause state corruption or data loss under normal usage, please open an issue with the `bug` label and mark it clearly — these are treated as high priority.

## Code style

- Use `set -euo pipefail` behavior where practical.
- Prefer explicit, readable bash over clever one-liners — this script edits infrastructure state, and clarity matters more than brevity.
- Quote variables consistently to avoid word-splitting issues with resource addresses.

Thanks again for helping improve state-shard!