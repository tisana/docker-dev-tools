# Compose upgrade validation

Execution date: 2026-10-14 (Asia/Bangkok). Branch: `chore/compose-upgrade`.

## Environment and safety inventory

Docker Desktop 4.95.0, Engine 29.8.2, Compose 5.6.0, linux/amd64.
Podman is not installed; no Podman runtime compatibility claim is made.
No containers or named volumes for these 22 helpers were present. The original
checkout's `data/` contained only `.gitkeep`. The running MongoDB belongs to
another repository and was not changed. Three otherwise unattached anonymous
volumes were inspected read-only: one empty, two containing PostgreSQL 18
clusters (63.1 MB and 46.8 MB). Ownership could not be established; all were
preserved. No existing data was migrated, deleted or opened by a new major.
All test projects use `upgrade-review-*`; bind-mounted fixtures must use copies
under ignored `.runtime/`, never the original `data/` directories.

## Task 1.1: PostgreSQL and SonarQube

PASS: baseline rendered config lacked both the PostgreSQL 16 data mount and
`SONAR_JDBC_URL`. After repair both checks passed. PostgreSQL 16's data directory
is `/var/lib/postgresql/data`; PostgreSQL 18's is `/var/lib/postgresql/18/docker`.
No old PostgreSQL 16 anonymous data or SonarQube H2 installation was found.

| Image | Resolved version | Digest | Platform |
|---|---|---|---|
| docker.io/library/postgres:16 | 16.15 | sha256:ca0bd484cb98bf4b24eb1010e73fb3fcbd6714d240fbc1a10eea5b7dbecb641d | linux/amd64 |
| docker.io/library/postgres:18 | 18.6 | sha256:74935e72241653ca55e0414067e6d8763aceb8a810eb51b452253ec3dcfc4336 | linux/amd64 |
| docker.io/library/sonarqube:latest | Version audit deferred to Phase 3 | sha256:2e378f7852b67d43507d3d383ee8fb5a522b8b0f920be9803f74f72952fce8f1 | linux/amd64 |

Executed with `POSTGRES_PORT=15432`:

```powershell
docker compose -p upgrade-review-pg16 -f postgresql.yml up -d --wait --wait-timeout 120
docker compose -p upgrade-review-pg16 -f postgresql.yml exec -T db psql -U metric -d metric -v ON_ERROR_STOP=1 -c "CREATE TABLE upgrade_fixture (id integer PRIMARY KEY, note text); INSERT INTO upgrade_fixture VALUES (1, 'phase1-persistence');"
docker compose -p upgrade-review-pg16 -f postgresql.yml exec -T db sh -c 'pg_dump -U metric -d metric -Fc -f /tmp/upgrade.dump'
docker compose -p upgrade-review-pg16 -f postgresql.yml exec -T db sh -c 'createdb -U metric restored; pg_restore -U metric -d restored /tmp/upgrade.dump'
docker compose -p upgrade-review-pg16 -f postgresql.yml down
docker compose -p upgrade-review-pg16 -f postgresql.yml up -d --wait --wait-timeout 120
# SELECT note FROM upgrade_fixture in metric and restored: phase1-persistence
```

SonarQube: polled `/api/system/status` until `UP` (bounded 180 seconds), created
`upgrade-fixture` using `/api/projects/create`, and queried its name from the
PostgreSQL `projects` table. `pg_dump -Fc` / `pg_restore` into separate `restored`
database passed. After `down` / `up`, both databases returned `Upgrade fixture`.
Compose dependency health check passed. Startup-only `--wait` is insufficient
for SonarQube; the explicit API poll was necessary. Stacks were stopped without
`-v`; isolated test volumes remain for diagnosis.

References: [PostgreSQL storage contract](https://github.com/docker-library/docs/blob/master/postgres/README.md),
[SonarQube database requirements](https://docs.sonarsource.com/sonarqube-community-build/server-installation/installing-the-database).

## Acceptance status

All 22 root files passed `docker compose -p review-<stack> -f <file> config --quiet`
with `OPENSEARCH_INITIAL_ADMIN_PASSWORD`, `WIREMOCK_PROXY_TARGET` and
`KAFKA_BROKERCONNECT` supplied. This is structural validation only.
Phase 1 remains in progress; Phases 2 and 3 have not started.
Kafka storage conversion remains deferred to Task 2.1.
