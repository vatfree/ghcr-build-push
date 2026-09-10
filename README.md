# GHCR Build and Push

Composite GitHub Action that builds a multi-arch Docker image, pushes it to GitHub Container Registry with `sha-7` / `tree-12` / `latest` / branch tags, and optionally triggers a Dokploy redeploy (on `staging`) or a Porter tag update (on `master`).

When the `tree-12` tag already exists in GHCR for the current tree hash, the build is skipped and the existing image is re-tagged with the current `sha-7`, `latest`, and branch tags — so unchanged source produces no rebuild, but downstream deploys still get a fresh sha.

## Usage

```yaml
name: Deploy to Registry
on:
  push:
    branches:
      - staging
      - master
  workflow_dispatch:
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  docker-build:
    runs-on: ubuntu-latest
    steps:
      - uses: vatfree/ghcr-build-push@v1
        with:
          app_name: my-app
          checkout_token: ${{ secrets.GH_TOKEN_JS }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          dokploy_url: https://nl.vatfree.com/api/deploy/XXXXXXXX
          porter_cluster: '2752'
          porter_project: '7505'
          porter_token: ${{ secrets.PORTER_STACK_7505_2752 }}
```

For monorepos with multiple images, set `tree_subdir` per image so unrelated changes don't invalidate the cache:

```yaml
      - uses: vatfree/ghcr-build-push@v1
        with:
          app_name: api-gateway
          tree_subdir: .       # or 'admin' / 'worker' / etc.
          dockerfile_path: ./Dockerfile
          ...
```

## Inputs

| Input                  | Description                                                                | Required | Default                          |
| ---------------------- | -------------------------------------------------------------------------- | -------- | -------------------------------- |
| `app_name`             | App name used as the image label and as the Porter app to update on master. Required only when `porter_token` is set. | No | `''`                             |
| `dockerfile_path`      | Path to the Dockerfile relative to the build context                       | No       | `./Dockerfile`                   |
| `context`              | Docker build context                                                       | No       | `.`                              |
| `platforms`            | Comma-separated target platforms                                           | No       | `linux/amd64,linux/arm64/v8`     |
| `checkout_token`       | Token for `actions/checkout` (needed for private submodules)               | No       | `''`                             |
| `github_token`         | Token used to log in to GHCR; falls back for checkout when `checkout_token` is empty | No | `''`                            |
| `publish_package_token`| Build arg forwarded as `PUBLISH_PACKAGE_TOKEN` for private npm installs    | No       | `''`                             |
| `extra_build_args`     | Additional multiline build args forwarded to `docker build`                | No       | `''`                             |
| `tree_subdir`          | Subdirectory whose git tree hash is used to skip rebuilds (default: full repo). Use this in monorepos to scope the cache to the relevant subdirectory. | No | `.`                              |
| `skip_if_unchanged`    | If `"true"`, skip the build when the tree-hash tag already exists in GHCR and re-tag the existing image with the current sha/branch tags instead. If `"false"`, always build. | No | `true`                            |
| `force_build`          | If `"true"`, always build and push, ignoring the tree-hash cache check      | No       | `false`                          |
| `dokploy_url`          | If set, GET this URL on `staging` builds to trigger a Dokploy redeploy    | No       | `''`                             |
| `porter_host`          | Porter host URL                                                            | No       | `https://dashboard.porter.run`   |
| `porter_cluster`       | Porter cluster ID                                                           | No       | `''`                             |
| `porter_project`       | Porter project ID                                                           | No       | `''`                             |
| `porter_token`         | Porter auth token. When set, the Porter deploy runs on `master`; otherwise it's skipped | No | `''`                             |

## Image tags

For every build four tags are produced:

- `ghcr.io/<owner>/<repo>:<sha-7>` — short commit SHA
- `ghcr.io/<owner>/<repo>:<tree-12>` — first 12 chars of the git tree hash of `tree_subdir` (used as the cache key)
- `ghcr.io/<owner>/<repo>:latest`
- `ghcr.io/<owner>/<repo>:<branch>` (lowercased)

When the build is skipped because `<tree-12>` already exists, the existing image is re-tagged with the current `<sha-7>`, `:latest`, and `<branch>` tags so downstream consumers (Dokploy, Porter) always see the new commit SHA pointing at the same image bytes.

## License

MIT