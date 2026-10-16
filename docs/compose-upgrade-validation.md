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

## Task 2.1: KRaft modernization (2026-10-15)

PASS on Docker linux/amd64: Confluent Kafka/Schema Registry 8.3.2. Registry
manifests also advertise linux/arm64; that platform was not runtime-tested.
Broker digest `sha256:5e8f3ab5b4977c9a8fd6137d26af2caad878aca316f24c55f08206217e3cec48`;
Schema Registry digest `sha256:482d3048b3f6029bae5794b5c3b5e2eea8d61ca064dacd32241315e818fd8d7c`.

Commands: `docker compose -p upgrade-review-kraft -f kafka.yml -f kafdrop.yml up
-d --wait --wait-timeout 180`, with `KAFKA_CLUSTER_ID` retained, Kafka port 19092,
Schema Registry 18082, Kafdrop 19000 and internal bootstrap `kafka:29092`.
`kafka-topics`, console producer/consumer and a Windows JDK21 Kafka client proved
internal and advertised host connectivity. `phase2-message` and
`host-client-message` survived `down`/`up`. HTTP schema registration persisted
schema ID 1 plus `fixture-base` / `fixture-wrapper` and their versioned reference.
Kafdrop's topic page still displayed the retained topic after recreation.

The modern image lacks `tar`; the initial host-test JAR export failed. Copying
individual JARs with `docker cp -L` worked without modifying the image. The host
client emitted an optional missing SLF4J binding warning but produced/consumed
successfully. No retained original 5.5 data existed; this was a fresh cluster
with fixture records, not an assertion of an in-place 5.5 migration.

Reference: [Confluent Docker configuration and KRaft](https://docs.confluent.io/platform/current/installation/docker/config-reference.html).

## Task 2.2: Keycloak / Odoo (verified through 2026-10-16)

Keycloak 26.8.1 (digest `sha256:79c623a22abacf3fe6c00a5fb30f5b7843d717b234033b80995891c06116d244`)
and PostgreSQL 17.11 (`sha256:2d2b8998d31037bf721cfdf764d76ba74171b4fab3431b7f72c27c56ddbdf9e3`):
imported `upgrade-fixture` realm/client/user; password token issuance and discovery
issuer passed before/after recreation and on resume. PostgreSQL contained the
realm record. Project `upgrade-review-keycloak`, port 18084, copied bind path
`.runtime/keycloak_pg17`. First bind initialization exceeded 180s; readiness now
allows a 120s start period plus bounded retries. Subsequent starts passed.

Odoo 20.0-20260926 (`sha256:cdd83e8359b3e8c357895d476396c05021fed9975bf420f353bab25fcaed1533`):
database creation, login, installed base addon, business record and actual 17-byte
attachment survived recreation. Project `upgrade-review-odoo`, port 18069,
separate config/addon directories and version-specific named volumes. Runtime
showed default HTTP interface was loopback; explicit `--http-interface=0.0.0.0`
fixed published access. The initial attachment test used removed `datas`; inspecting
the selected image's ORM showed `raw.content` is the correct encoded read value.
The corrected full test passed (`.superpowers/logs/odoo-retained-final.log`).

No original PostgreSQL12/Keycloak12/Odoo13 state or custom addons existed locally.
These results establish modern helper functionality/persistence, not compatibility
of arbitrary legacy realms, Odoo metadata or addons. Those installations require
separate copied migration rehearsals as documented in README.

References: [Keycloak containers](https://www.keycloak.org/server/containers),
[Keycloak database support](https://www.keycloak.org/server/db),
[Odoo official image](https://hub.docker.com/_/odoo).

## Task 2.3: database helpers / Redis Insight

PASS (Docker linux/amd64, 2026-10-14 through 2026-10-16):

- MongoDB: copied stopped Mongo6 fixture to separate v8 volume, booted 7.0 and
  set FCV7, then booted 8.0.34 and set FCV8. Authenticated record and unique
  `note_1` index survived migration/recreation. A separate restored pre-upgrade
  copy booted under Mongo6 and returned the same record.
- MySQL: 5.7.44 logical export -> 8.0.46 restore/export -> 8.4.11 restore.
  `review.MixedCase` record `phase2`, primary/unique indexes, utf8mb4 and case
  policy survived. `mysql8.yml` also retained its Phase1 fixture. Both Adminer
  6.1.2 UIs authenticated and executed SELECT queries through the browser.
  The 5.7 SQL backup restored successfully into a separate 5.7 instance.
- DynamoDB: copied 1.13.5 SQLite database into v3 storage; 3.3.1 retained the
  record, supported PartiQL SELECT, and retained a new record after recreation.
  Readiness polling replaced an insufficient fixed 5s delay in the test.
  A separate stopped-file restoration worked under 1.13.5.
- Redis Insight: browser opened the temporary Redis8.2.10 fixture, displayed
  `upgrade-fixture=phase2-value`, and retained the same connection ID after
  full stack recreation. The temporary Redis server was intentionally ephemeral.

Old Mongo6/Mongo7 and MySQL5.7/MySQL8.0 were migration/recovery fixtures only;
none remains a default helper. Database rollback results are in the ignored
`.superpowers/logs/database-rollback.log`; no dumps or fixture data are tracked.

| Selected image | Digest |
|---|---|
| mongo:8.0.34 | sha256:6657bf2413fefc5be649390318f5b4cd7328e982722f8ce2ac6dfa6f0a42cb03 |
| mysql:8.4.11 | sha256:24989179f58856db763f418010c1169fecc944e3ec1ea7e921572d83eb070f70 |
| adminer:6.1.2 | sha256:c4e439badea7c611be25d4b1e83f82333b5f10cc2fe8c899725c665541e33ff1 |
| amazon/dynamodb-local:3.3.1 | sha256:ff89bd48ff32cd8d9be5fee8873b65b8854dc408f1afe881be6eb00247bc0dab |
| redis/redisinsight:3.8.0 | sha256:b5e19ee240abef6edb435871b90ff8a210995422e8e018ab61c0339d318a1f84 |

Intermediate digests: Mongo7 `sha256:9650758f83af117fc96699f030a44f9300bed839734e67012f56df05b2fd6290`,
MySQL5.7 `sha256:4bc6bc963e6d8443453676cae56536f4b8156d78bae03c0145cbe47c2aad73bb`,
MySQL8.0 `sha256:7dcddc01f13bab2f15cde676d44d01f61fc9f99fe7785e86196dfc07d358ae2b`.

References: [Mongo7 upgrade](https://www.mongodb.com/docs/manual/release-notes/7.0-upgrade-standalone/),
[Mongo8 upgrade](https://www.mongodb.com/docs/manual/release-notes/8.0-upgrade-standalone/),
[MySQL paths](https://dev.mysql.com/doc/refman/8.4/en/upgrade-paths.html),
[DynamoDB history](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocalHistory.html),
[Redis Insight Docker](https://redis.io/docs/latest/operate/redisinsight/install/install-on-docker/).

## Task 2.4: Grafana / Prometheus

PASS on linux/amd64 through 2026-10-16:

- Monitoring copied the Grafana7.1.5 SQLite metadata, retired the copied legacy
  pie-chart plugin outside the loaded plugin directory, upgraded through 12.3.5
  to 13.2.4, and retained the saved dashboard after recreation.
- Standalone Grafana9.1.7/PostgreSQL14 saved a dashboard, dumped its database,
  restored into PostgreSQL17.11, upgraded through Grafana12.3.5 to13.2.4 and
  retained the dashboard after recreation. The pre-change 9.1.7/14 fixture still
  loaded its dashboard as a recovery check. Redis8.2.10 cache worked throughout.
- Prometheus3.15.0 passed `promtool check config`; a restored COPY of v2.21 data
  returned fixture value42 at timestamp1791996960, and v3 returned the same
  historical timestamp/value. New samples also survived recreation.
- Browser verified the provisioned JVM dashboard against host fixture metrics:
  uptime1h, heap50%, nonheap25%, CPU25%/15%, threads12/6/15, and populated memory
  graphs. Application/instance/pool variables resolved. Metrics absent from the
  bounded fixture (such as GC/load) correctly showed no data.
- JSON migration preserved all target expressions, variable queries, units,
  thresholds and active range/value mappings. Four singlestat panels became
  stat;27 legacy graph panels became timeseries. No external panel plugin is
  needed. Screenshot is local-only `.runtime/jvm-dashboard.png`.

The first copied-database provisioning attempt forced a different datasource UID
and failed. Keeping the existing identity by name fixed it. The modern API probe
uses the UID proxy route; the old numeric-ID route returned404. A Windows fixture
with a final CRLF failed text parsing; canonical LF output passed.

Automatic approval review rejected opening the existing history volume with an
old image. The recovery check instead cloned that source READ-ONLY into a new
isolated restoration volume, then ran the old image only on the copy; it passed.
No original history was opened or changed by that rejected action.

Digests: Grafana13.2.4 `sha256:dbbc38ea21103cefb50396340cd88afec779c7e070b0a7bf5a75b4ce46651d3f`;
Grafana12.3.5 bridge `sha256:a604f143dbb145a16cd58d22b1451826ee59fcf9de561dc6d698b1de222272d0`;
Grafana9.1.7 `sha256:8b81f1225bc450e56cdcaffbfa5a051b891c8332789960e7362c38edba73a123`;
Prometheus3.15.0 `sha256:efd719c99d83b060d9daefdcf00360461adf279f45ef5391f8d111892118753e`;
Redis8.2.10 `sha256:47670742d7924adbcdb404d1288b6327815b23141969c0939b8e5d61f78c2634`.

References: [Grafana13 upgrade](https://grafana.com/docs/grafana/latest/upgrade-guide/upgrade-v13.0/),
[Prometheus3 migration](https://prometheus.io/docs/prometheus/latest/migration/),
[Redis releases](https://github.com/redis/redis/releases).
