# Dev Tools Compose Stacks

Development tools packaged as standard Compose stacks. The files in this
repository are engine agnostic and can be run with Docker Compose, Podman
Compose, or another Compose Specification compatible runner.

## Usage

Start a tool with your preferred container engine:

```sh
docker compose -f postgresql.yml up -d
podman compose -f postgresql.yml up -d
```

Stop it the same way:

```sh
docker compose -f postgresql.yml down
podman compose -f postgresql.yml down
```

Most stacks expose common local development ports, so run one stack at a time
or set a project name when you want isolated resources:

```sh
docker compose -p dev-postgres -f postgresql.yml up -d
podman compose -p dev-postgres -f postgresql.yml up -d
```

## Notes

- Compose files omit the obsolete top-level `version` key and follow the
  Compose Specification.
- Docker Hub images are fully qualified with `docker.io/...` so Podman does not
  need short-name resolution prompts.
- Services avoid fixed `container_name` values so multiple projects can run
  without name collisions.
- Monitoring uses the default Compose network. Grafana reaches Prometheus at
  `http://prometheus:9090`, and Prometheus uses `host.docker.internal` when
  scraping an app running on the host.
- Search tools such as Elasticsearch, OpenSearch, and SonarQube may still need
  host kernel or file descriptor settings depending on your operating system
  and container engine.

## Upgrade execution

Work is staged in three phases. See [the validation record](docs/compose-upgrade-validation.md)
for actual checks; a parsed configuration is not evidence of application readiness.

Standalone PostgreSQL 16 now stores data in its named volume at
`/var/lib/postgresql/data`. Before using an existing installation, inspect its
container mounts: the previous parent mount may have left real records in an
anonymous child volume. Stop writers, take `pg_dump -Fc`, restore into isolated
PostgreSQL 16 storage, compare records, then restore the retained data into the
corrected volume. Do not delete the source volume until recovery is verified.
SonarQube uses the `sonar` database through `SONAR_JDBC_URL`, waits for PostgreSQL
readiness, and stores PostgreSQL 18 data in `sonarqube_postgresql` at
`/var/lib/postgresql`. Existing H2 databases must be preserved; switching JDBC
does not migrate their results. Rollback restores the old image and matching
backup together. Never downgrade an upgraded data directory in place.
