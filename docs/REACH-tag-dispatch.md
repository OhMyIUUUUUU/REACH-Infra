# Trigger REACH-Infra from a REACH Release Tag

This guide connects a pushed version tag in the `REACH` repository to the CI/CD workflow in `REACH-Infra`. The dispatcher belongs in `REACH`; the build and deployment remain in `REACH-Infra`.

## How the handoff works

1. A version tag such as `v1.2.3` is pushed to `REACH`.
2. A GitHub Actions workflow in `REACH` sends a `repository_dispatch` event to `REACH-Infra`, including the tag and commit SHA.
3. `REACH-Infra/.github/workflows/reach-release.yml` checks out that exact REACH tag, verifies its SHA, runs the frontend/Go/NLP checks, builds the images, and proceeds to Artifact Registry and Cloud Run when deployment prerequisites are configured.

The source-side dispatcher is not included in this repository yet. This guide shows how to add it to REACH.

## 1. Create a narrowly scoped dispatch token

The default `GITHUB_TOKEN` in REACH cannot dispatch a workflow in another repository. Create a fine-grained personal access token (or GitHub App token) with:

- **Resource owner:** `OhMyIUUUUUU`
- **Repository access:** only `REACH-Infra`
- **Repository permission:** `Contents: Read and write`
- **Expiration:** choose an expiration and plan to rotate it

In the `REACH` repository, open **Settings → Secrets and variables → Actions**, create a repository secret named `INFRA_DISPATCH_TOKEN`, and paste the token value there. Do not put the value in workflow YAML or commit it.

## 2. Add the dispatcher workflow to REACH

Create `REACH/.github/workflows/dispatch-infra-release.yml` with:

```yaml
name: Dispatch REACH release to Infra

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: read

jobs:
  dispatch:
    runs-on: ubuntu-latest
    steps:
      - name: Notify REACH-Infra
        env:
          INFRA_DISPATCH_TOKEN: ${{ secrets.INFRA_DISPATCH_TOKEN }}
          RELEASE_TAG: ${{ github.ref_name }}
          RELEASE_SHA: ${{ github.sha }}
        run: |
          set -euo pipefail
          payload="$(jq -n \
            --arg tag "$RELEASE_TAG" \
            --arg sha "$RELEASE_SHA" \
            '{event_type:"reach_release",client_payload:{tag:$tag,sha:$sha}}')"
          curl --fail-with-body --silent --show-error \
            --request POST \
            --url https://api.github.com/repos/OhMyIUUUUUU/REACH-Infra/dispatches \
            --header "Accept: application/vnd.github+json" \
            --header "Content-Type: application/json" \
            --header "Authorization: Bearer $INFRA_DISPATCH_TOKEN" \
            --header "X-GitHub-Api-Version: 2022-11-28" \
            --data "$payload"
```

Commit and push this workflow to REACH before pushing a release tag. The workflow uses the tag name and commit SHA from GitHub's tag-push event; it does not deploy `main` by accident.

## 3. Release and follow the run

Push a semantic version tag, for example:

```bash
git tag v1.2.3
git push origin v1.2.3
```

The Infra workflow validates tags in `vMAJOR.MINOR.PATCH` format, optionally with a prerelease/build suffix. Follow the dispatcher run in **REACH → Actions**, then the received run in **REACH-Infra → Actions**.

## Deployment safety

In the current Infra workflow, a `repository_dispatch` event of type `reach_release` sets deployment to enabled. If the GCP variables and permissions are configured, pushing a valid REACH tag publishes images to Artifact Registry and deploys to Cloud Run. Do not push a production tag until that is intended.

For a non-deploying check of a REACH revision, start `REACH-Infra`'s `REACH CI/CD` workflow manually, enter `reach_tag`, and leave `deploy` unchecked. This verifies the source tag and runs tests/builds, but does not test the cross-repository token or dispatcher.

Before enabling production deployment, configure the required `REACH-Infra` repository variables and review `CLOUD_RUN_PUBLIC_API=true`; the current Go API has no authentication. The existing Cloud SQL instance is not used by the current REACH services and is not modified by this workflow.

## Troubleshooting

- **No Infra run appears:** confirm the source workflow is on REACH's default branch, the `INFRA_DISPATCH_TOKEN` secret exists in REACH, and the token targets `REACH-Infra` with `Contents: Read and write`.
- **Infra rejects the event:** the event type must be exactly `reach_release`; its payload must include `tag` and `sha`.
- **Infra rejects the tag:** use `vMAJOR.MINOR.PATCH` (for example `v1.2.3`).
- **Build succeeds but deploy stops:** confirm the GCP project/region, Artifact Registry, Workload Identity Federation, deploy service account, IAM permissions, and public API opt-in are configured in REACH-Infra.
- **Token expired:** create a replacement token and update the `INFRA_DISPATCH_TOKEN` repository secret in REACH.

## Related

- Infra workflow: `.github/workflows/reach-release.yml`
- Main architecture: [REACH repository](https://github.com/OhMyIUUUUUU/REACH)
