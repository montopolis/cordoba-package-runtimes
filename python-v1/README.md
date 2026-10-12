# python-v1 package runtime

The image Cordoba package jobs run in. Published to public GHCR as
`ghcr.io/montopolis/cordoba-package-runtime:python-v1` by
`.github/workflows/build.yml`; Cordoba runs it by the digest that workflow
prints.

## Contract

| Path | Access | Contents |
|---|---|---|
| `/in/args.json` | read-only | The validated run arguments |
| `/in/data/*.csv` | read-only | One CSV per `sql` entry in `package.yaml`, header row first |
| `/in/files/…` | read-only | The package's `files/` directory |
| `/in/<entrypoint>` | read-only | The package's code (`build.py` by default), alongside the rest of the package tree |
| `/out/…` | read-write | The deliverable. Everything written here is collected |
| `/out/summary.json` | optional | A JSON object of flags Claude reads back |
| `/out/error.json` | optional | `{"code", "message", "hint"}` - marks the job failed with that message |

`/in` is the published package tree with `args.json` and `data/` added, so a
package may not ship a top-level `args.json` or `data/`.

The container has no network, a read-only root filesystem, a 256 MB `/tmp`,
and runs as uid 10001. stdout and stderr become the job's event log.
`PACKAGE_DEBUG=1` is set for debug runs.

## Running a package locally

```bash
docker run --rm --network none --read-only --tmpfs /tmp:size=256m \
  --cap-drop ALL --security-opt no-new-privileges --user 10001:10001 \
  --pids-limit 256 --memory 1g --memory-swap 1g --cpus 1 \
  -v "$PWD/in":/in:ro -v "$PWD/out":/out \
  ghcr.io/montopolis/cordoba-package-runtime@sha256:<digest> /in/build.py
```

Use the digest recorded on the package version (or on the failed job) to
reproduce a run on exactly the image it used.

## Changing the libraries

Edit `requirements.in`, then regenerate the pinned, hashed `requirements.txt`
on the same Python the image uses:

```bash
docker run --rm -v "$PWD":/work -w /work python:3.13-slim-bookworm sh -c \
  "pip install -q pip-tools && pip-compile --generate-hashes --allow-unsafe --strip-extras --output-file=requirements.txt requirements.in"
```

Merge to `main` to publish, then point Cordoba at the new digest.
