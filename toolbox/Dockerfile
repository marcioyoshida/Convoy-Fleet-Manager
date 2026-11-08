# Convoy toolbox image — skeleton (milestone M1, see ROADMAP.md).
# Goals: real POSIX utilities, a real init so signals reach the process (ADR 0008),
# artifact helpers that hash and meter (ADR 0001, cost story), and the annotation database.
# Nothing here is final; convoy-* helpers do not exist yet.

FROM debian:bookworm-slim

ARG TOOLBOX_VERSION=0.1.0
LABEL org.opencontainers.image.title="convoy-toolbox" \
      org.opencontainers.image.version="${TOOLBOX_VERSION}" \
      org.opencontainers.image.source="https://github.com/marcioyoshida/Convoy-Fleet-Manager"

# coreutils, findutils, diffutils, gawk/mawk, sed, grep, bsdextrautils, tini as init
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      coreutils findutils diffutils grep sed gawk bsdmainutils \
      tini ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# Collation and locale are pinned: sort on node A must merge with sort on node B.
ENV LC_ALL=C LANG=C

# The annotation database ships with the image and is versioned with it.
COPY annotations.yaml /etc/convoy/annotations.yaml

# Artifact helpers (to be implemented in M1):
#   convoy-get <key>            fetch an artifact, verify sha256, honour inlined small outputs
#   convoy-put <key>            write stdout to an artifact, record size/hash/request count
#   convoy-get-all <prefix>     fetch a partition set as files
#   convoy-merge-sum, convoy-merge-sum-by-key, convoy-merge-renumber
# COPY bin/convoy-* /usr/local/bin/

# A real init: tini reaps and forwards signals to the child, so TERM/HUP/USR1 reach the process.
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["sh"]
