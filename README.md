# platform-ci

Reusable GitHub Actions workflows for apps deployed on the Bitovi platform.
Maintained by the platform team; the platform's own gitops, charts, and
Terraform live separately in `bitovi/bitovi-platform-services`.

This repository is **public** so that both public and private app repos can
call these workflows. GitHub does not allow a public repository to call a
reusable workflow stored in a private one — the private-repo Actions access
setting grants access *only from private repositories*, and no setting widens
it. Keeping the shared CI here removes that constraint instead of routing
around it.

Nothing here is a secret. Credentials reach these workflows at call time: AWS
access through the caller's OIDC token, and the ECR repository, push role, and
write-back App key as Actions secrets.

## Workflows

| Workflow | Purpose |
| --- | --- |
| `container-build.yml` | Build an image, push it to the app's ECR repository, and write the tag into a values file so Argo CD deploys it. |
| `container-promote.yml` | Promote the image an environment already validated by re-tagging that exact manifest — no rebuild — then write the tag into the target values file. |
| `static-site-deploy.yml` | Build a static site, sync it to its S3 bucket, and invalidate CloudFront. The sync is the deploy; Argo CD is not in the path. |

Each file's header documents its full contract: every input, what the caller
must grant, and the failure modes worth knowing about.

## Using them

Pin a tag. `main` is not a stable ref — a change here would reach every app
tracking it with no review in that app's repo.

```yaml
jobs:
  staging:
    uses: bitovi/platform-ci/.github/workflows/container-build.yml@0.0.1
    # A called workflow can only narrow permissions, so the caller grants all
    # three: OIDC to AWS, the write-back commit, and cancelling the run when
    # there is nothing to deploy.
    permissions:
      id-token: write
      contents: write
      actions: write
    with:
      values-file: deploy/staging/values.yaml
      # An app interface file keeps the tag on the workload. The default is the
      # legacy root `.image.tag`, and the write-back refuses rather than
      # inventing a key, so a wrong path fails the release instead of quietly
      # deploying nothing.
      image-tag-path: .workloads.main.image.tag
      # Pushes the write-back as a GitHub App. Needed wherever the deploy
      # branch has a ruleset requiring pull requests, since GITHUB_TOKEN
      # cannot push past one.
      writeback-app-id: ${{ vars.CI_WRITEBACK_APP_ID }}
      environment-label: staging
    secrets: inherit
```

`secrets: inherit` is the intended way to pass credentials — it forwards
`ECR_PUSH_ROLE_ARN`, `ECR_REPOSITORY`, `CI_WRITEBACK_APP_PRIVATE_KEY`, and
`BITOVI_PLATFORM_AWS_ACCOUNT_ID` with no per-caller mapping. It forwards only
what the calling repo can actually see, so an org-level secret scoped to
private repositories will not reach a public one.

An explicit `secrets:` block **replaces** inheritance rather than adding to it,
so a caller that maps any secret by hand — as the static-site staging job does
for its `STAGING_`-prefixed values — must map every secret it needs, including
`BITOVI_PLATFORM_AWS_ACCOUNT_ID`.

Apps with no staging environment call `container-build.yml` on the release tag
directly and skip promotion.

## Versioning

Tags are the release surface, starting at `0.0.1`. A fix here reaches an app by
bumping its pin — no PR in this repository is needed for an app to adopt one,
and none is possible for an app to avoid one.

While the series is `0.0.x` every tag may carry a breaking change, so read the
release notes before bumping a pin rather than assuming the increment is safe.
Changes worth that caution are a new required input, a removed input, and a
changed default that moves where the image tag is written.

Because a tag here publishes code to every app that pins it, tagging is treated
as a release action rather than a bookmark:

- **Tags are immutable.** A ruleset blocks updating and deleting them, so an
  existing pin can never be repointed at different code. Cleanup of a bad tag
  is an org-admin bypass, deliberately.
- **A tag's commit must be on `main`** — `verify-release-tag.yml` fails any tag
  pointing at something that was never merged. It runs after the tag exists, so
  it catches mistakes rather than preventing them; immutability is what protects
  apps already pinned.

Write access to this repo is not the same thing as permission to deploy the
fleet, which is what those two rules are for. Pinning a commit SHA instead of a
tag sidesteps tag handling entirely, and is the tightest option available to a
caller.
