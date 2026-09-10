# GHCR Build and Push

Composite GitHub Action that builds a multi-arch Docker image, pushes it to GitHub Container Registry with `sha-7` / `latest` / branch tags, and optionally triggers a Dokploy redeploy (on `staging`) or a Porter tag update (on `master`).

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
          porter_app_name: my-app
          porter_cluster: '2752'
          porter_project: '7505'
          porter_token: ${{ secrets.PORTER_STACK_7505_2752 }}
```

## Inputs

| Input                  | Description                                                                | Required | Default                          |
| ---------------------- | -------------------------------------------------------------------------- | -------- | -------------------------------- |
| `app_name`             | App name; exported as `APP_NAME` env var for downstream steps              | No       | `''`                             |
| `dockerfile_path`      | Path to the Dockerfile relative to the build context                       | No       | `./Dockerfile`                   |
| `context`              | Docker build context                                                       | No       | `.`                              |
| `platforms`            | Comma-separated target platforms                                           | No       | `linux/amd64,linux/arm64/v8`     |
| `checkout_token`       | Token for `actions/checkout` (needed for private submodules)               | No       | `''`                             |
| `github_token`         | Token used to log in to GHCR; falls back for checkout when `checkout_token` is empty | No | `''`                            |
| `publish_package_token`| Build arg forwarded as `PUBLISH_PACKAGE_TOKEN` for private npm installs    | No       | `''`                             |
| `extra_build_args`     | Additional multiline build args forwarded to `docker build`                | No       | `''`                             |
| `dokploy_url`          | If set, GET this URL on `staging` builds to trigger a Dokploy redeploy    | No       | `''`                             |
| `porter_app_name`      | If set, runs `porter app update-tag <name>` on `master` builds             | No       | `''`                             |
| `porter_host`          | Porter host URL                                                            | No       | `https://dashboard.porter.run`   |
| `porter_cluster`       | Porter cluster ID                                                           | No       | `''`                             |
| `porter_project`       | Porter project ID                                                           | No       | `''`                             |
| `porter_token`         | Porter auth token                                                           | No       | `''`                             |

## Image tags

For every build three tags are produced:

- `ghcr.io/<owner>/<repo>:<sha-7>` — short SHA
- `ghcr.io/<owner>/<repo>:latest`
- `ghcr.io/<owner>/<repo>:<branch>` (lowercased)

## License

MIT