# Teleport auth image: stock teleport-distroless with /usr/local/bin/teleport
# replaced by a build of upstream gravitational/teleport v$(UPSTREAM_VERSION)
# plus eng/maestra/patches. tctl, tsh, tbot and the rest stay stock. See
# eng/maestra/README.md.
#
# Build context: the repo root. Dockerfile.dockerignore keeps it to
# UPSTREAM_VERSION and patches/, so a README edit does not bust the cache.
#
# Auth-only binary: no webassets_embed (web assets are loaded only by the
# proxy, lib/service/service.go initProxyEndpoint -> newHTTPFileSystem), no
# bpf / sessionhelper_embed (node), desktop_access_rdp (desktop service, Rust)
# or piv (client YubiKeys). Do not run proxy, node or desktop roles on it.

# Go 1.26.8, not go.mod's 1.25.14 floor: the stock 18.11.1 binary reports
# go1.26.8 (`teleport version`), upstream's release pipeline uses that line.
FROM golang:1.26.8-bookworm@sha256:dc9ad6c05acc7a88e5b71bde60a5fe3bd4b9f0db209011711b464107438a8107 AS src
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
# libpam0g-dev: the pam tag's cgo headers (lib/pam dlopens libpam at runtime,
# so nothing new is linked). gcc and git ship in the golang image; the base
# is digest-pinned, so the apt versions are bookworm's at that digest.
# hadolint ignore=DL3008
RUN apt-get update \
 && apt-get install -y --no-install-recommends libpam0g-dev \
 && rm -rf /var/lib/apt/lists/*
WORKDIR /src
RUN --mount=type=bind,source=eng/maestra/UPSTREAM_VERSION,target=/UPSTREAM_VERSION \
    git -c advice.detachedHead=false clone --depth 1 -b "v$(cat /UPSTREAM_VERSION)" \
      https://github.com/gravitational/teleport .
# All patches in one git apply: atomic, and a patch that no longer applies to
# the new upstream tag fails the build. No patches = a stock rebuild.
# gitref.go is what upstream's `make version` writes (version.mk); the suffix
# marks the binary as ours: v18.11.1-0-g<sha>-maestra.<patches sha8>.
RUN --mount=type=bind,source=eng/maestra,target=/maestra \
    set -eu; \
    set -- /maestra/patches/*.patch; \
    if [ -e "$1" ]; then git apply --verbose "$@"; else echo "no patches: stock rebuild"; fi; \
    PATCHES=$({ [ ! -e "$1" ] || cat "$@"; } | sha256sum | cut -c1-8); \
    printf 'package teleport\n\nfunc init() { Gitref = "%s-maestra.%s" }\n' \
      "$(git describe --long --tags)" "$PATCHES" > gitref.go
ENV CGO_ENABLED=1 \
    GOFLAGS=-buildvcs=false \
    TAGS="pam kustomize_disable_go_plugin_support"
# proxy.golang.org drops HTTP/2 streams under the ~1k-module fan-out
# (INTERNAL_ERROR on 4 of them, 2026-10-21 run 39603291663); the cache mount
# keeps what landed, so a retry only fetches the rest.
RUN --mount=type=cache,target=/go/pkg/mod \
    for i in 1 2 3; do go mod download && break; [ "$i" = 3 ] && exit 1; sleep 10; done

FROM src AS test
# Packages the patches touch; override with --build-arg TEST_PACKAGES=...
ARG TEST_PACKAGES="./lib/events/..."
# hadolint ignore=SC2086
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build \
    go test -tags "$TAGS" $TEST_PACKAGES

FROM src AS build
# Upstream Makefile: TELEPORT_LDFLAGS = GO_LDFLAGS (-w -s + KUBECTL_SETVERSION)
# + teleportBuildType=community; BUILDFLAGS adds -trimpath -buildvcs=false.
# SHELL (pipefail) is inherited from src; hadolint cannot see across stages.
# hadolint ignore=DL4006
RUN --mount=type=cache,target=/go/pkg/mod --mount=type=cache,target=/root/.cache/go-build \
    KUBECTL_VERSION=$(go list -m -f '{{.Version}}' k8s.io/kubectl | sed 's/v0/v1/') \
 && go build -tags "$TAGS" -trimpath \
      -ldflags "-w -s -X k8s.io/component-base/version.gitVersion=$KUBECTL_VERSION -X github.com/gravitational/teleport/lib/modules.teleportBuildType=community" \
      -o /out/teleport ./tool/teleport

# Keep this tag equal to UPSTREAM_VERSION: build-and-push.yml fails otherwise,
# and Renovate bumps both in one PR.
FROM public.ecr.aws/gravitational/teleport-distroless:18.11.1@sha256:382550edf9ad16cb2e34561846b5968f1994daed62cb97ee9dd53c4f081bc3d3
COPY --from=build /out/teleport /usr/local/bin/teleport
