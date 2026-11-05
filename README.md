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
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262  # v4.4.0
      - id: login
        uses: cloud-in-a-bottle/actions/registry-login@d8a8869a427242308e6fd22754961d1e6e095674  # v1
      - uses: docker/setup-buildx-action@8d2750c68a42422c14e847fe6c8ac0403b4cbd6f  # v3.12.0
      - id: build
        uses: docker/build-push-action@10e90e3645eae34f1e60eeb005ba3a3d33f178e8  # v6.19.2
        with:
          push: true
          tags: ${{ steps.login.outputs.image }}:${{ github.sha }}
      - uses: sigstore/cosign-installer@756fe2d284c7292c4e9b62c8a28f1e5629c59532  # v4.1.3
      - name: Sign
        run: cosign sign --yes "${{ steps.login.outputs.image }}@${{ steps.build.outputs.digest }}"
```

`test-registry-login.yml` runs the same steps on a tiny image, pushed to `registry.cloudinabottle.org/actions/registry-login-test`, then pulls it back and verifies the signature. It only runs when started by hand (`gh workflow run test-registry-login.yml`), since every run leaves an image in the registry.

## Releasing

Callers pin a commit, with its tag in a comment (`@<sha>  # v1`), so a moved tag can't change what they run. After a compatible change merges, move `v1` to it (`git tag -f v1 && git push -f origin v1`) and update the pin in the example above; callers pick it up when they bump their pin. Changes that would break callers get a `v2`.
