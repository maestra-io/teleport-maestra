<!-- cspell:words maestra distroless kustomize -->
# teleport-maestra

The Teleport **auth** image for `teleport.maestra.io` on us-iota: stock `public.ecr.aws/gravitational/teleport-distroless:<version>` with `/usr/local/bin/teleport` rebuilt from upstream `gravitational/teleport` `v<version>` plus the patches in `patches/` (Cloudflare R2 session recordings, maestra-io/issues-maestra#1632). `tctl`, `tsh`, `tbot`, `fdpass-teleport`, dumb-init and the entrypoint stay stock. Pushed to `public.ecr.aws/g5f9s8a4/teleport-maestra`.

All maestra changes live here; upstream is cloned at build time, so it flows in without merges (the maestra-io/dotnet-monitor `eng/maestra` layout):

- `UPSTREAM_VERSION` is the upstream tag built from source (without the `v`).
- `patches/*.patch` are applied in one `git apply` onto the fresh checkout. A patch that no longer applies fails the build. No patches = a stock rebuild.
- `Dockerfile` clones, patches, runs the package tests of the patched code (`--target test`, packages in `ARG TEST_PACKAGES`), builds `./tool/teleport` and copies it over the stock binary. The binary is auth-only: tags `pam kustomize_disable_go_plugin_support`, no web assets (only the proxy loads them), no BPF, RDP, PIV or embedded session helper. Do not run proxy, node or desktop roles on this image. `teleport version` prints `git:v<version>-0-g<sha>-maestra.<patches sha8>`.
- `.github/workflows/build-and-push.yml` runs on changes here (Markdown excluded), on pull requests (build and test, no push) and on dispatch. On `main` it pushes `<version>-maestra.<run_number>` and the moving major tag (`18`), and writes the digest to the step summary. ECR Public has no tag immutability: pin the digest.
- `renovate.json` bumps `UPSTREAM_VERSION` (`github-tags` of `gravitational/teleport`) and the `teleport-distroless` `FROM` (tag and digest) in one grouped PR. The workflow fails when the `FROM` tag differs from `UPSTREAM_VERSION`. Majors stay manual.

## Change a patch

```sh
git clone --depth 1 -b "v$(cat eng/maestra/UPSTREAM_VERSION)" https://github.com/gravitational/teleport /tmp/tp
cd /tmp/tp && git apply /path/to/teleport-maestra/eng/maestra/patches/*.patch
# edit, then regenerate (one patch per concern, keep the header)
git diff > /path/to/teleport-maestra/eng/maestra/patches/0001-<name>.patch
```

Check locally from the repo root:

```sh
docker build -f eng/maestra/Dockerfile --target test .
docker build -f eng/maestra/Dockerfile -t teleport-maestra .
docker run --rm --entrypoint teleport teleport-maestra version
```

## How the image reaches iota

maestra-io/fluxcd `iota/us-iota-lw-kube-infra/teleport-maestra/teleport.yaml` (HelmRelease `teleport-maestra`, chart `teleport-cluster`) gets a postRenderer patch on Deployment `teleport-maestra-auth`: an `op: test` on the chart's stock image (`public.ecr.aws/gravitational/teleport-distroless:<chart version>`), then an `op: replace` with this image by digest. A chart bump changes the stock image, the `op: test` fails the render, and the auth image has to be rebuilt for the new version first: auth never silently falls back to stock or drifts from the proxy.

The registry is public on purpose. Teleport on iota follows the cold-start rule (public images only, no tbot): a private ECR pull there goes through a Teleport-issued token, which does not exist while Teleport is down. See maestra-io/kubernetes-clusters `k0s/us-iota-lw-kube-infra/teleport-cold-start.md`.
