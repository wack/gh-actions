# GitHub Actions

This repo contains a collection of reusable GitHub Actions for frequently used workflows.
In particular, we use a lot of Rust at Wack, so there are reusable workflows for
linting, testing, and building Rust code.

# Available Workflows

Right now, the only reusable workflows is the one we use most often: validate.
We use it in every Rust project we have, which tends to be a lot of them.

You can add it to an existing workflow like this:

```yml
jobs:
  build-and-validate:
    uses: wack/gh-actions/.github/workflows/validate.yml@trunk
```

To enable Covecov (not yet fully implemented. We need to pass in a `secret` to the Workflow
so it can read the token.)
  
```yml
jobs:
  build-and-validate:
    uses: wack/gh-actions/.github/workflows/validate.yml@trunk
    with:
      codecov-enabled: true
    secrets:
      codecov_access_token: ${{ secrets.token }}
```

You can also upload artifacts generated as part of your workflow.

# Multi-Lens PR Review (agentic workflow)

`multi-lens-review` is a [GitHub Agentic Workflow](https://github.com/github/gh-aw) (gh-aw)
that reviews a pull request through three lenses in parallel — **Spec** (against the linked
Linear ticket), **QA** (edge cases, test coverage, error handling), and **Architecture**
(coupling, dependency injection, SOLID) — then an orchestrator synthesizes the findings and
posts a single review, as a bot CODEOWNER, to gate merging: `APPROVE` when the PR is clean,
`REQUEST_CHANGES` with inline comments otherwise.

It runs on demand — comment `/review` on a PR, or `gh aw run multi-lens-review -f pr=<N>` —
so it never fires on the intermediate rebase pushes of a stacked-PR workflow. See
[`CLAUDE.md`](./CLAUDE.md) for the agent-facing review loop.

Unlike `validate.yml`, an agentic workflow is **not** a reusable `workflow_call` workflow, so
you can't reference it with `uses:`. Instead you install a copy into the target repo and
compile it there.

## Adding it to another repository (as a template)

1. **Initialize the target repo for gh-aw** (once):
   ```bash
   gh extension install github/gh-aw   # if not already installed
   gh aw init
   ```
2. **Add this workflow** from here (installs into `.github/workflows/`):
   ```bash
   gh aw add wack/gh-actions/.github/workflows/multi-lens-review.md@trunk
   # or, for guided setup that also helps wire secrets and open a PR:
   gh aw add-wizard
   ```
   Both the `.md` extension and the `@trunk` ref are required: without `.md` the spec is
   rejected, and without `@trunk` gh-aw defaults to a `main` ref that this repo doesn't have.
   (The full blob URL — `https://github.com/wack/gh-actions/blob/trunk/.github/workflows/multi-lens-review.md`
   — also works, but mind the long path if you copy it.)
3. **Adapt the repo-specific bits** in the copied `multi-lens-review.md`:
   - the Linear team key / branch convention (`robbie/multi-…` → `MULTI-…`) if the repo maps
     to a different Linear team, and
   - the review-bot account used for the merge gate.
4. **Recompile** so the `.lock.yml` matches:
   ```bash
   gh aw compile multi-lens-review
   ```
5. **Add the secrets** (`gh secret set <NAME>`, or Settings → Secrets → Actions):
   - `ANTHROPIC_API_KEY` — Claude engine (the workflow pins `claude-sonnet-5`)
   - `LINEAR_API_KEY` — Linear personal API key (the Spec lens's source)
   - `REVIEW_BOT_TOKEN` — the review bot's fine-grained PAT (repo-scoped, Pull requests: Read & write)
6. **Set up the merge gate** so only a clean review unblocks merging:
   - add `.github/CODEOWNERS` with `*  @<review-bot>` — the bot must have **write** access and
     must **not** be the account that authors the PRs it reviews;
   - protect the default branch: require a PR + **Require review from Code Owners**, with
     **Dismiss stale approvals on push OFF** (so approvals survive rebases) and admin bypass on.
7. **Commit both** `multi-lens-review.md` and the generated `multi-lens-review.lock.yml`.

To pull in later fixes from this repo, use `gh aw update`.
