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

## Phase 1 checks completed so far

| Stack | Actual functional/persistence result | Isolated project / host port |
|---|---|---|
| MongoDB 6 | Authenticated insert/read; same record after recreation | upgrade-review-mongo / 17017 |
| MySQL 8 (resolved 8.4.11) | Authenticated SQL create/insert/select; retained after recreation | upgrade-review-mysql8 / 13306 |
| DynamoDB Local 1.13.5 | Create table, put/get, retained after recreation; `review_us-east-1.db` exists | upgrade-review-dynamodb / 18000 |
| Metabase latest | Setup, saved dashboard ID 2 and question ID 40; both retained after recreation | upgrade-review-metabase / 13100 |
| Elasticsearch 9.4.1 | Index/document write, retained document after recreation | upgrade-review-elasticsearch / 19200 |
| Monitoring | Host metric `upgrade_fixture=42`, Grafana datasource proxy; dashboard and historical sample retained after recreation; promtool passed | upgrade-review-monitoring / 13000, 19090 |
| WireMock 2.27.1 | Proxy, persisted mapping, replay after recreation with upstream stopped | upgrade-review-wiremock / 18088 |
| Kafka 5.5 | Topic, internal produce/consume and schema registration ID 1 passed | upgrade-review-kafka / 19092, 18082 |
| Concurrent PostgreSQL | Two projects on 15432/15433; `phase1-persistence` and `separate-project` remained separate after recreation | upgrade-review-pg16 and upgrade-review-pg16b |

The DynamoDB volume initially failed `test -w` as UID 1000 (root-owned 0755).
A one-shot same-image root initializer changes only that directory's ownership,
then the server runs as its original UID 1000. No `-sharedDb` was added.

Host routing: Docker Desktop resolves `host.docker.internal` to 192.168.65.254.
No extra gateway mapping was needed. The initial Windows HttpListener fixture
failed because HTTP.sys denied URL registration; a plain TCP HTTP fixture worked.
WireMock and Prometheus successfully reached that host fixture from containers.

Required-variable negative checks passed for WireMock target, Kafdrop broker and
OpenSearch password. `git diff --check` passed after normalizing mixed line endings.
No PostgreSQL 16/SonarQube source state was found to recover; sample backup/restore
checks were performed instead. Original bind directories for Jenkins, Nexus,
Keycloak, Odoo and WireMock were absent. Their existing mount paths were retained;
permissions of populated installations remain deployment-specific. Odoo database
and filestore must be recovered together. Grafana Redis remains ephemeral cache.
No retained original Kafka topics/schemas were found; only isolated fixtures were
created. Legacy broker storage is deliberately deferred to the KRaft conversion.

Task 1.3 final results: Windows JDK 21 with the broker's dereferenced client JARs
produced and consumed `host-client-message` at `localhost:19092`. The container
client produced and consumed at `kafka:29092`. Kafdrop 4.3.0's topic page returned
HTTP 200 containing `upgrade-fixture`. phpMyAdmin authenticated to the test MySQL
service at `mysql` and returned its logged-in page (logout control present).
Docker Desktop's host route passed; Podman is unavailable and unverified.
No gateway mapping was added. No existing service was stopped or data removed.

Phase 1 image digests (linux/amd64; mutable tags are dated observations):

| Image | Digest |
|---|---|
| mongo:6 | sha256:8b6d8f5bbedb25cb73517b65cf99f13aeb75ad5b157a56c479287a840bbad3ac |
| mysql:8 | sha256:24989179f58856db763f418010c1169fecc944e3ec1ea7e921572d83eb070f70 |
| metabase/metabase:latest | sha256:1607e8808c3c628e7dadafe0825ec13a98090493ba047ab4e5d48b0fe322531b |
| amazon/dynamodb-local:1.13.5 | sha256:d03b77d09ac9fe26f98eed083bfec7b189ebbae2df213ee780b1632de17e9d3d |
| prom/prometheus:v2.21.0 | sha256:d43417c260e516508eed1f1d59c10c49d96bbea93eafb4955b0df3aea5908971 |
| grafana/grafana:7.1.5 | sha256:579044d31fad95f015c78dff8db25c85e2e0f5fdf37f414ce850eb045dd47265 |
| elasticsearch:9.4.1 | sha256:74c82c22b899cdd184598c96456d0f2d254c17ab204210fb470c1901e0b646f4 |
| rodolpheche/wiremock:2.27.1 | sha256:e2fc94f195a230fc483c29d4f6201653032786af7466dec00273cf654be94d00 |
| confluentinc/cp-kafka:5.5.0 | sha256:0aa096fcc79432f0b1e8414f2796d0e36e7caeb8148b27b4e53da8a2523d72ce |
| confluentinc/cp-zookeeper:5.5.0 | sha256:32cbe5706332f8bf208e5f6a3cbf5e491dcacd68f8515b149ec5f96edb09277e |
| confluentinc/cp-schema-registry:5.5.0 | sha256:b38661525957847a19172b52695615b4a2ce45a4ef31a9d4855c9a0994624add |
| obsidiandynamics/kafdrop:latest | sha256:4486f52da3b037d62665f5effca34ec096675183742f4c596237b368501546e5 |
| phpmyadmin/phpmyadmin:latest | sha256:42a200db07b4e70fbf32c594ad4521cf16399b8e54bbb5adceae98e7566dfbeb |
