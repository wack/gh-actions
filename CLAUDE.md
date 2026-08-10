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

### What the lenses actually see (`.github/review-exclude`)

The lenses do not read the whole PR. The diff is split per file, any path matching
`.github/review-exclude` **in the repository being reviewed** is dropped, and the remainder
fills a 20,000-line budget (override per repo with the `REVIEW_DIFF_MAX_LINES` variable) a
whole file at a time. Whatever did not fit is reported in `diff-coverage.json` as
`omitted_files` with `truncated: true`, and the orchestrator is forbidden from approving a
truncated review — it must REQUEST_CHANGES and name what went unreviewed.

This exists because `gh pr diff` emits files in alphabetical order, so before it a single
early-sorting generated file could consume the entire budget and silently evict every source
file after it. A review that reads no source code must never be able to approve.

**This file is load-bearing: anything it matches is invisible to all three lenses.** Two
rules follow.

- **Add generated and vendored artifacts that get committed** — lockfiles, committed build
  output, generated clients, snapshots. Consuming repos maintain their own list; the shape
  differs per ecosystem (`Cargo.lock` and `target/` for Rust, `pnpm-lock.yaml` and `.next/`
  for a Next.js app, `.devenv/` for Nix), which is exactly why this workflow ships no
  built-in list. A repo with no such file excludes nothing.
- **Never add hand-written code.** Exclusions are silent, so a source path listed here
  narrows the review with nothing in the output to reveal it.

Format: one gitignore-flavoured glob per line (`*`/`?` stop at a `/`, `**` spans them, a
trailing `/` means everything beneath), `#` comments, or `re:` for a raw regex.

The list is read from `.github/`, which gh-aw restores from the **base** branch before
pre-agent steps run, so a pull request cannot add entries to hide its own changes. Keep it
under `.github/` for that reason — moving it elsewhere would silently make the exclusion
list attacker-controlled by any PR author.

This repo is reviewed by the same workflow, so it can carry its own `.github/review-exclude`
if it ever commits generated files. It currently has none, so the file is absent.

### Spec source

The Spec lens loads requirements from the **Linear ticket** encoded in the PR's head branch:
`robbie/multi-<id>-<slug>` → ticket `MULTI-<id>`. If the branch is not a `robbie/multi-…`
branch, no spec is available (the Spec lens says so; the QA and Architecture lenses still run).

## Conventions

- Default branch: `trunk`.
- Linear feature branches: `robbie/multi-<id>-<slug>`.
- After editing any `.github/workflows/*.md` agentic workflow, recompile with
  `gh aw compile <name>` and commit both the `.md` and generated `.lock.yml`.
