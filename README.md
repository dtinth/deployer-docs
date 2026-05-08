# deployer-docs

Receives built documentation artifacts from authorized source repositories and deploys them as subdirectories of the `dtinth/docs` gh-pages branch. Authentication is via GitHub Actions OIDC, federated by [octo-sts](https://github.com/octo-sts/app). No deploy credentials are stored in source repositories.

## Usage from a source repo

```yaml
permissions:
  contents: read
  id-token: write   # required for octo-sts

steps:
  - run: <build docs>
  - uses: actions/upload-artifact@v4
    with:
      name: docs
      path: <build-output-dir>
  - uses: dtinth/deployer-docs@main
    with:
      artifact-name: docs
```

The action exits once the deploy is dispatched. Deploy status appears in this repository's [Actions tab](https://github.com/dtinth/deployer-docs/actions).

## Adding a new source repo

1. Extend the `subject_pattern` regex in [`.github/chainguard/dispatcher.sts.yaml`](./.github/chainguard/dispatcher.sts.yaml) to include the new repo.
2. Add a `case` in [`.github/workflows/deploy.yml`](./.github/workflows/deploy.yml) mapping the source repo to its `target_repo`, `branch`, and `subdir`.
3. If the source repo is **private**, add `.github/chainguard/deployer-docs.sts.yaml` there:
   ```yaml
   issuer: https://token.actions.githubusercontent.com
   subject_pattern: "repo:dtinth/deployer-docs:ref:refs/heads/main"
   permissions:
     actions: read
   ```
   Public repos do not need this file.

## How it works

1. The source repo builds its docs and uploads them as a GitHub Actions artifact.
2. It calls `dtinth/deployer-docs@main`, which uses octo-sts to mint a short-lived `actions: write` token for this repo, then calls `workflow_dispatch` on `deploy.yml`.
3. `deploy.yml` determines the deploy target from the source repo name, downloads the artifact, mints a `contents: write` token for `dtinth/docs` via octo-sts, and pushes the artifact contents into the target subdirectory of the gh-pages branch.
