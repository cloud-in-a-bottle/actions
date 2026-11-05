# actions

Shared GitHub Actions for cloud-in-a-bottle repos.

## registry-login

Logs docker in to the app image registry, `registry.cloudinabottle.org`, so the workflow can push its repo's images. It swaps the workflow's GitHub OIDC token for a push credential, which lasts 6 hours, and runs `docker login` with it. How the registry checks this is in [imbue-hosted-spaces' `services/app-image-registry`](https://github.com/cloud-in-a-bottle/imbue-hosted-spaces/tree/main/services/app-image-registry).

The registry only issues push credentials to workflows in `cloud-in-a-bottle` repos, running on `main` from `push`, `workflow_dispatch`, or `schedule`, on GitHub-hosted runners. A repo may push `<repo>` and `<repo>/*`, lowercased; the `image` output is `registry.cloudinabottle.org/<repo>`.

The job needs `id-token: write`. Building, pushing, and signing an image:

```yaml
name: Publish image
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  id-token: write  # the registry login, and keyless signing

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - id: login
        uses: cloud-in-a-bottle/actions/registry-login@v1
      - uses: docker/setup-buildx-action@v3
      - id: build
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ steps.login.outputs.image }}:${{ github.sha }}
      - uses: sigstore/cosign-installer@v4.1.2
      - name: Sign
        run: cosign sign --yes "${{ steps.login.outputs.image }}@${{ steps.build.outputs.digest }}"
```

`test-registry-login.yml` runs the same steps on a tiny image, pushed to `registry.cloudinabottle.org/actions/registry-login-test`, then pulls it back and verifies the signature. It only runs when started by hand (`gh workflow run test-registry-login.yml`), since every run leaves an image in the registry.

## Releasing

Callers use the `v1` tag. After a compatible change merges, move it: `git tag -f v1 && git push -f origin v1`. Changes that would break callers get a `v2`.
