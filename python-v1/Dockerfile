# Cordoba package runtime: python-v1
#
# A fixed, versioned image that Cordoba package jobs run in. Nothing is
# installed at run time: jobs run with no network and a read-only root
# filesystem, so every library a package may import has to be baked in here.
#
# Changing anything in this directory changes what `python-v1` means. Published
# package versions are pinned to the image digest they were published with, so
# a rebuild never alters an existing version - but do not remove or downgrade a
# library without introducing `python-v2`.
#
# Base image pinned by digest (python:3.13-slim-bookworm, multi-arch index).
FROM python:3.13-slim-bookworm@sha256:2325bb286ec344af3e5898cc224b5844e2707ac6e26b1632516fd3edc84a5e26

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1 \
    HOME=/tmp \
    PACKAGE_IN=/in \
    PACKAGE_OUT=/out

COPY requirements.txt /tmp/requirements.txt

# --require-hashes refuses any distribution whose hash is not listed, including
# transitive ones, so the image cannot pick up an unpinned dependency.
RUN pip install --require-hashes --no-deps -r /tmp/requirements.txt \
    && rm /tmp/requirements.txt

# Matches the --user the runner passes. /in and /out are bind mounts at run
# time; creating them here only keeps the image usable without the runner.
RUN groupadd --gid 10001 package \
    && useradd --uid 10001 --gid 10001 --no-create-home --home-dir /tmp --shell /usr/sbin/nologin package \
    && mkdir /in /out \
    && chown 10001:10001 /out

USER 10001:10001
WORKDIR /in

# The runner appends the package's entrypoint, e.g. `/in/build.py`.
ENTRYPOINT ["python"]
