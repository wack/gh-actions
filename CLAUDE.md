# gh-actions — Agent Instructions

## Getting a PR reviewed (Multi-Lens PR Review)

Merging into `trunk` is **gated** on the Multi-Lens PR Review agentic workflow
(`.github/workflows/multi-lens-review.md`). A PR cannot be merged until the review bot
(`@felixbracken`, the sole CODEOWNER) submits an **APPROVE** review. Only a repo admin can
bypass the gate — agents cannot.

### Trigger a review when your PR is ready

Do this **when the PR is actually ready for review**, not on every push. Two ways:

1. **Slash command** — comment `/review` on the pull request.
2. **Dispatch** — run one of:
   ```bash
   gh aw run multi-lens-review -f pr=<PR_NUMBER>
   gh workflow run "Multi-Lens PR Review" -f pr=<PR_NUMBER>
   ```

Rebasing a stacked PR does **not** require re-review — an existing approval survives rebases
(branch protection has "dismiss stale approvals on push" turned off).

### The review loop

The workflow reviews the PR through three lenses in parallel — **Spec** (against the linked
Linear ticket), **QA** (edge cases, test coverage, error handling), and **Architecture**
(coupling, dependency injection, SOLID) — then posts a single review:

- **APPROVE** → the PR satisfies its spec and is safe to merge; the gate is green.
- **REQUEST_CHANGES** → read the inline review comments, address every one, then re-trigger
  the review. Repeat until you get an APPROVE. Do not try to merge before then — you can't.

Inspect outstanding feedback with:
```bash
gh pr view <PR_NUMBER> --json reviewDecision,reviews
gh api repos/wack/gh-actions/pulls/<PR_NUMBER>/comments   # inline review comments
```

### Spec source

The Spec lens loads requirements from the **Linear ticket** encoded in the PR's head branch:
`robbie/multi-<id>-<slug>` → ticket `MULTI-<id>`. If the branch is not a `robbie/multi-…`
branch, no spec is available (the Spec lens says so; the QA and Architecture lenses still run).

## Conventions

- Default branch: `trunk`.
- Linear feature branches: `robbie/multi-<id>-<slug>`.
- After editing any `.github/workflows/*.md` agentic workflow, recompile with
  `gh aw compile <name>` and commit both the `.md` and generated `.lock.yml`.
