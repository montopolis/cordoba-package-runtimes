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
| `/in/captures/…` | read-only | Only when `package.yaml` has a `captures:` entry: the captured agency printouts (PDFs) it selected, plus `index.json` (below) |
| `/in/<entrypoint>` | read-only | The package's code (`build.py` by default), alongside the rest of the package tree |
| `/out/…` | read-write | The deliverable. Everything written here is collected |
| `/out/summary.json` | optional | A JSON object of flags Claude reads back |
| `/out/error.json` | optional | `{"code", "message", "hint"}` - marks the job failed with that message |

`/in` is the published package tree with `args.json` and `data/` added (and
`captures/`, for a package with a `captures:` entry), so a package may not ship
a top-level `args.json`, `data/` or `captures/`.

### Captured agency printouts (`/in/captures/`)

A package can use official agency printouts (an RRC Specific Lease Query, a
Comptroller Lease Drop, ...) that Cordoba has captured beforehand, without any
network access of its own. Its `package.yaml` lists one or more queries that
return capture ids:

```yaml
captures:
  - file: sql/06_captures.sql   # SELECT-only, :arg binds like `sql` entries;
                                # must return a `capture_id` column
```

Before the container starts, Cordoba runs them, copies each selected capture's
PDF into `/in/captures/` and writes `/in/captures/index.json`:

```json
{ "captures": [
  { "capture_id": 12, "file": "12_rrc.specific_lease_query_279375.pdf",
    "type": "rrc.specific_lease_query", "source": "rrc",
    "params": { "lease_number": "279375", "district": "08", "kind": "G",
                "from": "2024-01", "to": "2024-03" },
    "status": "ok", "row_count": 3, "lease_name": "DAVIS, A. L.",
    "metadata": {}, "sha256": "…", "capture_method": "automated",
    "captured_at": "2026-11-05T15:04:05Z" } ] }
```

- `file` is relative to `/in/captures/`. Match captures to what the package
  needs by `type` + `params` (the Fetcher document type and its parameters),
  never by file name. If several match, use the newest `captured_at`.
- `status` is `ok` or `no_data`. A `no_data` capture is still the agency's
  record (an empty result, printed) and belongs in a filing.
- `capture_method` is `automated` (captured by the Fetcher) or `manual` (saved
  by a person from Cordoba's manual queue).
- With a `captures:` entry and nothing selected, `/in/captures/` still exists
  and `index.json` is `{"captures": []}`. Without a `captures:` entry there is
  no `/in/captures/` at all.

The container still runs with `--network none`: everything in
`/in/captures/` was fetched by Cordoba before the job started.

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
