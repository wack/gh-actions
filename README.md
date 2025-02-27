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
