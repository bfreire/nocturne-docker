# Custom Postgres image with the Nocturne three-role bootstrap script baked in.
#
# Why a custom image instead of a bind mount?
#   When Portainer deploys a "Stack from Git", the repo is cloned inside the
#   Portainer container at /data/compose/<id>/. A relative bind mount like
#   ./pg-init reaches that path — but the Docker daemon on the host doesn't
#   share that filesystem, so it tries to mkdir /data and fails on a
#   read-only host fs. Build context, in contrast, is shipped to the daemon
#   as a tarball at build time, so no host path is involved.
#
# The script itself is an unmodified copy of:
#   https://github.com/nightscout/nocturne/blob/main/docs/postgres/container-init/00-init.sh
FROM docker.io/library/postgres:17.6

COPY 00-init.sh /docker-entrypoint-initdb.d/00-init.sh
RUN chmod +x /docker-entrypoint-initdb.d/00-init.sh
