# ECR Public Image Cache

Keep GitHub Actions jobs off the `public.ecr.aws` rate limits.

GitHub-hosted runners share egress IPs with other tenants, and anonymous ECR Public pulls are throttled per source IP. Jobs fail with `toomanyrequests: Rate exceeded` or `toomanyrequests: Data limit exceeded` because of someone else's usage. This repository has two actions that work as a pair, like `actions/cache/restore` and `actions/cache/save`:

- **`restore`** signs in to ECR Public with the job's AWS credentials (falling back to anonymous pulls with a notice when it has none), restores an image tarball from the Actions cache, loads it, and optionally pulls the listed images that are still missing, with retries.
- **`save`**, at the end of the job, packs an explicit image list and/or every image matching a pattern among the job's containers (`docker ps -a`), and writes it to the Actions cache.

Neither action ever fails the job: a cache or sign-in problem costs pulls, not the run.

## Usage

```yaml
jobs:
  test:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      id-token: write # OIDC into the AWS role that allows the ECR Public sign-in
    steps:
      # Optional. Without credentials, `restore` pulls anonymously.
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@<sha> # vX
        with:
          role-to-assume: arn:aws:iam::<account>:role/<Repo>-Deployment
          aws-region: us-east-1

      - name: Restore container images
        id: images
        uses: CVector-Energy/ecr-public-image-cache/restore@<sha> # v1.0.0
        with:
          key: my-images-${{ runner.os }}-${{ env.TOOL_VERSION }}
          restore-keys: |
            my-images-${{ runner.os }}-
          images: public.ecr.aws/docker/library/node:24-bookworm # optional

      # ... steps that run containers ...

      - name: Save container images
        if: ${{ !cancelled() && steps.images.outputs.cache-hit != 'true' }}
        uses: CVector-Energy/ecr-public-image-cache/save@<sha> # v1.0.0
        with:
          key: ${{ steps.images.outputs.cache-primary-key }}
          images: public.ecr.aws/docker/library/node:24-bookworm
          # and/or
          pattern: '^public\.ecr\.aws/supabase/[^:]*:[^:]*$'
```

The AWS role needs `ecr-public:GetAuthorizationToken` and `sts:GetServiceBearerToken`. CVector deployment roles get them from the shared `CvecEcrDeployment` managed policy in `cvec-management-account`, or from a role-specific `EcrPublicPull` inline policy.

Use one restore/save pair per job: both use a fixed tarball path under `runner.temp`.

## `restore` inputs

| Input | Description | Default |
|-------|-------------|---------|
| `key` | Cache key. Include whatever determines the image tags. | required |
| `restore-keys` | Newline-separated prefixes to fall back to. `cache-hit` stays `false` on a prefix match, so `save` refreshes the entry. | `''` |
| `images` | Whitespace-separated images to pull, with retries, if the cache did not supply them. | `''` |
| `sign-in` | Sign in to `public.ecr.aws` when the job has AWS credentials. | `'true'` |

Outputs: `cache-hit`, `cache-matched-key`, `cache-primary-key`.

## `save` inputs

| Input | Description | Default |
|-------|-------------|---------|
| `key` | Key to write under; pass `restore`'s `cache-primary-key`. | required |
| `images` | Whitespace-separated images to pack. Missing ones are skipped with a notice. | `''` |
| `pattern` | Extended regex matched against the images of the job's containers (`docker ps -a`). Anchor it to `name:tag`. Containers run with `--rm` are gone by the end of the job; list their images in `images`. | `''` |

Matching against containers rather than all local images keeps a prefix-match restore from carrying every previous generation of images forward into each new entry.
