# Cordoba package runtimes

The container images that Cordoba package jobs run in. Each runtime is a
directory here and a tag of `ghcr.io/montopolis/cordoba-package-runtime`:

| Runtime | Image | Contract |
|---|---|---|
| `python-v1` | `ghcr.io/montopolis/cordoba-package-runtime:python-v1` | [python-v1/README.md](python-v1/README.md) |

The images are public, so they pull without credentials. They hold a pinned
base image and pinned, hash-checked libraries - nothing else.

## Publishing

A push to `main` that touches a runtime builds it for `linux/amd64` and
`linux/arm64` and pushes it to GHCR. The workflow summary prints the new
digest.

Jobs always run an image by digest, never by tag. A rebuild therefore changes
nothing until Cordoba is pointed at the new digest, and package versions that
were already published keep the digest they were published with.

A runtime's name is a promise about what it contains: add libraries freely,
but removing or downgrading one means a new runtime (`python-v2`), not a
change to an existing one.
