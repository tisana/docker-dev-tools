# Compose upgrade validation

Execution dates: 2026-10-14 through 2026-10-16 (Asia/Bangkok).
Branch: `chore/compose-upgrade`.

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
| docker.io/library/sonarqube:latest | Community Build26.10 (audited in Task3.1) | sha256:2e378f7852b67d43507d3d383ee8fb5a522b8b0f920be9803f74f72952fce8f1 | linux/amd64 |

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

All three phase gates passed for the available isolated fixtures. The final
repository acceptance matrix lists each stack, image digest, functional check
and persistence result. Unavailable populated legacy installations and Podman
are explicitly qualified rather than treated as migration/runtime passes.

## Phase 1 historical checks

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
created. At this Phase1 gate, legacy storage was deferred; Task2.1 subsequently completed the KRaft conversion.

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

## Task 2.5: Jenkins / Tomcat / WireMock

PASS on Docker linux/amd64 (2026-10-14 through16): Jenkins2.580.1-jdk21 created
and ran a sample shell job successfully; job/build/settings survived recreation.
The fixture initialization disabled setup/security ONLY in its ignored isolated
home to exercise the build API; these settings are not part of the Compose file.
No external agents or original installed plugins/jobs existed to migrate.

Tomcat11.0.27-jdk21-temurin-noble served an actual WAR containing a JSP importing
`jakarta.servlet.http.HttpServletRequest`; response was `phase2-war GET`.
No assertion is made that old javax applications run unchanged. The old helper
was replaced by `tomcat.yml`.

Official WireMock3.13.2 loaded the copied2.x mapping and successfully proxied a
new unique path `/fixture-v3-new`, saved its mapping, recreated, and replayed it
with the upstream stopped. Both old/new mapping files remained in isolated
`.runtime/wiremock`; original bind data was never used.

Digests: Jenkins `sha256:a660310e39ade10631f774bacd5219de767dfd08947ea5c32a739b5e5bb382c1`;
Tomcat `sha256:eaad3aaf03a79837917d1c09e4bf5baedfbc3c6392f89a1c32a054cebb7b9d75`;
WireMock `sha256:0d4ecb3e4dc8213fd7a4d37d6a78f6e6b553a6d2e15bd51b0999781282ac61b3`.

References: [Jenkins Java policy](https://www.jenkins.io/doc/book/platform-information/support-policy-java/),
[Tomcat supported versions](https://tomcat.apache.org/whichversion.html),
[WireMock Docker](https://wiremock.org/docs/standalone/docker/).

## Task 2.6: search stacks / PostgreSQL patch

PASS on 2026-10-16, Docker linux/amd64. Elasticsearch9.4.1 fixture data
was cold-copied read-only into isolated source storage, then backed up using
the snapshot API. A fresh9.5.5 cluster restored that snapshot, found the
recognizable document by query, connected Kibana9.5.5 (`available`), and
retained the document through recreation. The source and snapshot remain
available for recovery; no original data volume was upgraded in place.
This does not establish a direct Elasticsearch7 ->9 migration.

OpenSearch3.9.0 formed a two-node green cluster, accepted authenticated index
writes/search, connected matching Dashboards3.9.0, and retained the document
through recreation. Browser admin login reached `/app/home`. The status API
also requires authentication. No populated legacy OpenSearch installation
existed; an OpenSearch2 data migration remains installation-specific.
`vm.max_map_count` was1048576 on the tested Docker Linux VM; no host settings
were changed. Allocate sufficient VM memory for both nodes and Dashboards.
Demo HTTPS uses a self-signed certificate; use the required strong initial
admin password and limit this demo configuration to development.

PostgreSQL16.15 read `phase1-persistence` before and after recreation under
`upgrade-review-pg16` on15432; major16 storage and prior restore remain valid.

Resolved digests (all amd64):

| Image | SHA256 |
| --- | --- |
| Elasticsearch9.5.5 | `13b4a40bb095774f5fda635e45fd5677cc6a26565e42b24997b0c72b88e75984` |
| Kibana9.5.5 | `13b72ce7222c36e4213229004ae62a87aeb32fd372ebfa6fb38f7a3285fdbb78` |
| OpenSearch3.9.0 | `adfa61f85025d06b4aeb562e7e74fde7e31c437039c93c3862c17e9acebd6c7c` |
| OpenSearch Dashboards3.9.0 | `4bdb8ded547ce7f4c379a10261f1b4aad58726900c215be570a7d1fe86f2a7d8` |
| PostgreSQL16.15 | `ca0bd484cb98bf4b24eb1010e73fb3fcbd6714d240fbc1a10eea5b7dbecb641d` |

References: [Elastic releases](https://www.elastic.co/docs/release-notes),
[Elastic upgrade guidance](https://www.elastic.co/docs/deploy-manage/upgrade/deployment-or-cluster),
[OpenSearch releases](https://opensearch.org/releases/),
[OpenSearch version history](https://docs.opensearch.org/latest/version-history/).

## Task 3.1: current floating applications

Pulled immediately before checks on2026-10-16, linux/amd64:

| Image | Resolved application | SHA256 |
| --- | --- | --- |
| metabase/metabase:latest | v0.64.2.7 (2026-10-15) | `f69044cd03f1a9968cdd427f23e3f50b893666f0092e6cac1ef7e2e30e100a96` |
| sonatype/nexus3:latest | 3.97.0-13 Community Edition, Java21.0.12 | `772ad5badde07e15f2c25c750437fcbb01ebdb84a8c43a3ed38eda60bb6448f1` |
| sonarqube:latest | Community Build26.10.0.133078 | `2e378f7852b67d43507d3d383ee8fb5a522b8b0f920be9803f74f72952fce8f1` |
| postgres:18 | 18.6 (Sonar dependency) | `74935e72241653ca55e0414067e6d8763aceb8a810eb51b452253ec3dcfc4336` |

Metabase PASS: copied the stopped Phase1 H2 volume read-only into a new project,
then started the newly pulled digest. Question40 and dashboard2 retained their
names and loaded before/after recreation. `MB_DB_FILE` remains valid; the actual
image stores its H2 file beneath `/metabase-data/metabase.db/`. Preserve the
whole mounted directory while stopped. No external database migration was made.

Nexus PASS: UID200 owns `/nexus-data`; the current image uses Java21 and H2
`db/nexus.mv.db`. Its startup launcher sizes heap/direct memory; no custom JVM
flags were needed. Removed an irrelevant Elasticsearch comment. The user
explicitly reviewed/accepted the CE EULA for this isolated instance before the
artifact test. Created a raw hosted repository, uploaded/downloaded a fixture,
recreated, and downloaded identical content. No original Nexus database existed;
this does not verify an OrientDB migration. Older installations must follow the
vendor's H2/Java intermediate upgrade sequence before latest; retain a complete
stopped `/nexus-data` backup and its matching image for recovery.

Sonar PASS: copied stopped Phase1 PostgreSQL and extensions into a new project.
PostgreSQL18 is supported starting with Community Build26.8. Removed unused9092
and default `conf`/`lib/bundled-plugins` mounts; retained old volumes untouched.
Only data/extensions/logs are default application mounts, so bundled files come
from the current image. The real scanner analyzed one Python file; CE task
succeeded, PostgreSQL `snapshots` contained results, and two analyses remained
after recreation. Server logs showed no plugin/startup failure; image emits a
locale COMPAT notice. Scanner8.1.0.6389 digest
`a3f4215076706c95a17a68c19322ee916e40a3acd081a8c1a1e839e0194afa57` was a test
dependency only. The scanner now writes to `/tmp/.scannerwork` by default;
the test set `SCANNER_WORKDIR_PATH` to retain its task report. No scanner token
is committed. Before changing an existing installation, export local conf and
inventory extensions, apply supported environment settings and compatible
plugins; never discard the old configuration volumes as part of `down`.

References checked2026-10-16:
[Metabase Docker](https://www.metabase.com/docs/latest/installation-and-operation/running-metabase-on-docker),
[Nexus image](https://github.com/sonatype/docker-nexus3),
[Nexus onboarding/migration](https://help.sonatype.com/en/ce-onboarding.html),
[Nexus EULA API](https://help.sonatype.com/en/eula-rest-api.html),
[Sonar database support](https://docs.sonarsource.com/sonarqube-community-build/server-installation/installing-the-database),
[Sonar Docker mounts/settings](https://docs.sonarsource.com/sonarqube-community-build/server-installation/from-docker-image/set-up-and-start-container).

## Task 3.2: current floating UI helpers

Pulled immediately before checks on2026-10-16, linux/amd64:

| Image | Resolved application | SHA256 |
| --- | --- | --- |
| obsidiandynamics/kafdrop:latest | 4.3.0, Java25.0.4 | `4486f52da3b037d62665f5effca34ec096675183742f4c596237b368501546e5` |
| mongo-express:latest | 1.0.2, Node18.20.3 | `1b23d7976f0210dbec74045c209e52fbb26d29b2e873d6c6fa3d3f0ae32c2a64` |
| phpmyadmin/phpmyadmin:latest | 5.2.3, PHP8.3.26 | `42a200db07b4e70fbf32c594ad4521cf16399b8e54bbb5adceae98e7566dfbeb` |
| swaggerapi/swagger-ui:latest | 5.33.3 (bundle PACKAGE_VERSION) | `b26661cd754a7efbb2e1aab419ffb5603320d8f35f0bd086d1d7506e23934d23` |

Kafdrop: replaced the48MB heap/180KB stack and deprecated `-noverify` with
64MB initial/256MB maximum heap and512KB stack. Current Java25 starts cleanly
with those options. Combined-stack browsing was tested in Phase2; the final
standalone project uses `host.docker.internal:19092`, with the broker advertising
that hostname and binding explicitly to0.0.0.0. It sees the retained topic and
messages through the host route. Separate projects never rely on shared DNS.

mongo-express PASS: the actual image's `config.default.js` reads
`ME_CONFIG_BASICAUTH` (the upstream main branch documents the newer `_ENABLED`
name). Both Mongo URL/admin auth and explicit UI username/password work.
Browser created/read/updated/deleted a temporary document. Mongo retained the
Phase1 record/index after recreation, while the UI-deleted document remained
absent. Wrong web password returned401; wrong database credentials failed auth.
`MONGO_EXPRESS_PASSWORD` is required. Retain floating policy and recheck the
authentication variable against the next pulled digest.

phpMyAdmin PASS: HTTP80 only, `PMA_ARBITRARY=1`, authenticated to MySQL8.4.11
on an explicitly shared isolated network, executed a SELECT returning the
fixture value and server version. `/sessions` is writable and retained its
session file after recreation. The image regenerates its cookie encryption
secret, so recreation required login again; persistent session files do not
guarantee persistent authentication. No HTTPS listener is implied.

Swagger PASS: browser requested the local `/swagger/sample.yml` (HTTP200),
rendered that Swagger2 document and reported `swagger/sample.yml` as its
configured URL. The repository sample itself is titled Swagger Petstore; the
network request confirms this was the local file, not the remote default.
`API_URL` is handled by the image's configurator. The specification bind is
read-only and the running container reports readwrite=false for that mount.

References: [Kafdrop](https://github.com/obsidiandynamics/kafdrop),
[mongo-express image](https://github.com/docker-library/docs/blob/master/mongo-express/README.md),
[mongo-express configuration](https://github.com/mongo-express/mongo-express),
[phpMyAdmin image/session contract](https://github.com/phpmyadmin/docker),
[Swagger installation](https://github.com/swagger-api/swagger-ui/blob/main/docs/usage/installation.md).

## Follow-up: standalone PostgreSQL18 (2026-10-17)

At the user's request, `postgresql.yml` now pins `postgres:18.6` and mounts
`postgresql-v18-data` at `/var/lib/postgresql`. The old `postgresql` volume is
not reused or deleted. The previous major16 requirement in the plan is superseded.

PASS on Docker linux/amd64: the pre-change image assertion failed on16.15;
the updated configuration resolves18.6, separate storage and the parent mount.
Created an isolated16.15 fixture with a recognizable row, primary key and note
index; exported using `pg_dump -Fc`, copied the binary archive with `docker cp`,
and stopped the source. Restored into a fresh18.6 project using
`pg_restore --no-owner --no-acl --exit-on-error`, wrote a second row, then ran
`down`/`up --wait`. Both rows and indexes remained. SQL reported180006 and
`/var/lib/postgresql/18/docker`. The project used host25433 and was stopped
afterward; no existing user database was opened or migrated by this check.
All21 configurations, six required-setting negative checks, port overrides and
`git diff --check` passed after the change.

Pulled18.6 digest `sha256:74935e72241653ca55e0414067e6d8763aceb8a810eb51b452253ec3dcfc4336`
(amd64); source16.15 digest
`sha256:ca0bd484cb98bf4b24eb1010e73fb3fcbd6714d240fbc1a10eea5b7dbecb641d`.
The October16 inventory below is historical; the current standalone reference
is `docker.io/library/postgres:18.6` instead of16.15. Existing users must perform
their own logical migration, including required roles, grants and extensions.
Rollback requires the old image and its matching pre-upgrade storage.
Reference checked2026-10-17:
[official PostgreSQL image storage rules](https://github.com/docker-library/docs/blob/master/postgres/README.md#pgdata).

## Follow-up: remove duplicate MySQL entry point (2026-10-17)

At the user's request, removed `mysql8.yml` and its `MYSQL8_*` port settings.
The repository now contains 21 active Compose files. `mysql.yml` remains the
MySQL8.4.11/Adminer6.1.2 entry point with its existing bind-storage and
case-insensitive initialization policy. README examples now reference that file.
Old `_mysql8-data` volumes are preserved; logical migration is required before
their data appears under the retained configuration. The original22-stack
results below are historical evidence, including the retired helper's tests.
All 21 remaining files passed Compose parsing, six required-variable negative
checks, PostgreSQL/MySQL port overrides, and `git diff --check` on2026-10-17.

## Initial repository acceptance (2026-10-16)

Fresh whole-branch review covered all22 configurations, provisioning/dashboard,
environment example, migration guide and acceptance record. No Critical or
Important issue was found. Two presentation issues remain deferred: the File
Descriptors time-series panel uses a linear axis instead of the old log10 axis,
and Thread States/Log Events use default colors instead of explicit alias colors.
The data and queries remain intact; these affect visual readability and severity
color cues, not collection or persistence. Check tooltips/legends when reading
those panels. The review independently confirmed target/variable preservation.

Implementation decisions: used a PowerShell ledger and focused runtime checks
instead of Bash helper scripts/a YAML test framework; the cost is that rerunning
the full fixture suite requires PowerShell and the recorded environment. Used
new version-specific storage where a major upgrade must not open old files;
the cost is that existing users must explicitly migrate their retained data
before the modern helpers show it. Old storage was preserved.

All22 final stacks parse on Compose5.6.0;29 active image references resolved to
local amd64 digests below. Implicit latest references were included. Negative
checks unset each of six required variables (Kafka cluster ID, Keycloak,
mongo-express and OpenSearch passwords, standalone Kafdrop broker, WireMock
upstream), and each configuration failed with its setting named. PostgreSQL
25432/0.0.0.0 and MySQL23306/Adminer28080 overrides rendered correctly. Phase1
also ran two independent PostgreSQL projects with separate data/ports and a
broader-bind host route. `promtool check config` under3.15.0 and `git diff
--check` passed. Podman executable/runner is unavailable: unverified.

| Original stack -> final entry | Functional and persistence evidence | Task |
| --- | --- | --- |
| dynamodb.yml | Copied SQLite upgrade, CRUD/PartiQL, recreation and restored old-image recovery | 1.2,2.3 |
| elasticsearch.yml | Snapshot restore9.4->9.5, query, Kibana available, retained document | 2.6 |
| grafana.yml | Saved dashboard, PostgreSQL logical migration, cache, recreation and old recovery | 2.4 |
| jenkins.yml | Job/build/settings on isolated home, recreation; no external agent/plugins present | 2.5 |
| kafdrop.yml | Combined and standalone broker routes, browser topics/messages; no durable local state | 2.1,3.2 |
| kafka.yml | Host/container produce/consume, schemas/references and messages retained | 2.1 |
| keycloak.yml | Imported realm/client/user, token/discovery, PostgreSQL recreation; legacy installation absent | 2.2 |
| metabase.yml | Pre-pull H2 copied upgrade; question/dashboard retained | 1.2,3.1 |
| mongodb.yml | 6->7->8/FCV/indexes, browser CRUD, wrong auth, recreation, restored old recovery | 2.3,3.2 |
| monitoring.yml | Fixture scrape/JVM panels, Grafana copied migration, Prom history/recreation and old recovery | 2.4 |
| mysql.yml | 5.7->8.0->8.4 logical migration, mixedcase/indexes/settings, Adminer query/recreation | 2.3 |
| mysql8.yml | Authenticated SQL/indexes, Adminer query/recreation | 1.2,2.3 |
| nexus.yml | Raw artifact upload/download/recreation after user-authorized EULA; old installation absent | 3.1 |
| odoo.yml | Base addon/business record/attachment/login retained; original custom addons absent | 2.2 |
| opensearch.yml | Two-node green cluster, authenticated index/search, Dashboards login, recreation | 2.6 |
| phpmyadmin.yml | Authenticated MySQL8.4 SQL; session files retained, login required after recreation | 3.2 |
| postgresql.yml | SQL, dump/restore and recreation at16.15 | 1.1,2.6 |
| redisinsight.yml | Redis key browsing and saved profile recreation; new storage, no automatic1.x migration | 2.3 |
| sonarqube.yml | Real scanner/CE result, PostgreSQL snapshots and recreation | 1.1,3.1 |
| swagger.yml | Local specification HTTP200/rendering, read-only bind; no application state | 3.2 |
| tomcat7.yml -> tomcat.yml | Jakarta-compatible WAR/JSP endpoint; javax migration required | 2.5 |
| wiremock.yml | Copied mapping, fresh proxy/record and recreated offline replay | 1.3,2.5 |

Final image inventory follows. Pinned versions identify their runtime baseline;
floating application versions are recorded in Tasks3.1/3.2. The same digest may
appear in several stacks. These are audit records, not digest locks in Compose.

| Image reference | Resolved digest (SHA256), linux/amd64 |
| --- | --- |
| `docker.elastic.co/elasticsearch/elasticsearch:9.5.5` | `13b4a40bb095774f5fda635e45fd5677cc6a26565e42b24997b0c72b88e75984` |
| `docker.elastic.co/kibana/kibana:9.5.5` | `13b72ce7222c36e4213229004ae62a87aeb32fd372ebfa6fb38f7a3285fdbb78` |
| `docker.io/amazon/dynamodb-local:3.3.1` | `ff89bd48ff32cd8d9be5fee8873b65b8854dc408f1afe881be6eb00247bc0dab` |
| `docker.io/confluentinc/cp-kafka:8.3.2` | `5e8f3ab5b4977c9a8fd6137d26af2caad878aca316f24c55f08206217e3cec48` |
| `docker.io/confluentinc/cp-schema-registry:8.3.2` | `482d3048b3f6029bae5794b5c3b5e2eea8d61ca064dacd32241315e818fd8d7c` |
| `docker.io/grafana/grafana:13.2.4` | `dbbc38ea21103cefb50396340cd88afec779c7e070b0a7bf5a75b4ce46651d3f` |
| `docker.io/jenkins/jenkins:2.580.1-jdk21` | `a660310e39ade10631f774bacd5219de767dfd08947ea5c32a739b5e5bb382c1` |
| `docker.io/library/adminer:6.1.2` | `c4e439badea7c611be25d4b1e83f82333b5f10cc2fe8c899725c665541e33ff1` |
| `docker.io/library/mongo:8.0.34` | `6657bf2413fefc5be649390318f5b4cd7328e982722f8ce2ac6dfa6f0a42cb03` |
| `docker.io/library/mongo-express:latest` | `1b23d7976f0210dbec74045c209e52fbb26d29b2e873d6c6fa3d3f0ae32c2a64` |
| `docker.io/library/mysql:8.4.11` | `24989179f58856db763f418010c1169fecc944e3ec1ea7e921572d83eb070f70` |
| `docker.io/library/odoo:20.0-20260926` | `cdd83e8359b3e8c357895d476396c05021fed9975bf420f353bab25fcaed1533` |
| `docker.io/library/postgres:16.15` | `ca0bd484cb98bf4b24eb1010e73fb3fcbd6714d240fbc1a10eea5b7dbecb641d` |
| `docker.io/library/postgres:17.11` | `2d2b8998d31037bf721cfdf764d76ba74171b4fab3431b7f72c27c56ddbdf9e3` |
| `docker.io/library/postgres:18` | `74935e72241653ca55e0414067e6d8763aceb8a810eb51b452253ec3dcfc4336` |
| `docker.io/library/redis:8.2.10` | `47670742d7924adbcdb404d1288b6327815b23141969c0939b8e5d61f78c2634` |
| `docker.io/library/sonarqube:latest` | `2e378f7852b67d43507d3d383ee8fb5a522b8b0f920be9803f74f72952fce8f1` |
| `docker.io/library/tomcat:11.0.27-jdk21-temurin-noble` | `eaad3aaf03a79837917d1c09e4bf5baedfbc3c6392f89a1c32a054cebb7b9d75` |
| `docker.io/metabase/metabase` | `f69044cd03f1a9968cdd427f23e3f50b893666f0092e6cac1ef7e2e30e100a96` |
| `docker.io/obsidiandynamics/kafdrop:latest` | `4486f52da3b037d62665f5effca34ec096675183742f4c596237b368501546e5` |
| `docker.io/opensearchproject/opensearch:3.9.0` | `adfa61f85025d06b4aeb562e7e74fde7e31c437039c93c3862c17e9acebd6c7c` |
| `docker.io/opensearchproject/opensearch-dashboards:3.9.0` | `4bdb8ded547ce7f4c379a10261f1b4aad58726900c215be570a7d1fe86f2a7d8` |
| `docker.io/phpmyadmin/phpmyadmin:latest` | `42a200db07b4e70fbf32c594ad4521cf16399b8e54bbb5adceae98e7566dfbeb` |
| `docker.io/prom/prometheus:v3.15.0` | `efd719c99d83b060d9daefdcf00360461adf279f45ef5391f8d111892118753e` |
| `docker.io/redis/redisinsight:3.8.0` | `b5e19ee240abef6edb435871b90ff8a210995422e8e018ab61c0339d318a1f84` |
| `docker.io/sonatype/nexus3` | `772ad5badde07e15f2c25c750437fcbb01ebdb84a8c43a3ed38eda60bb6448f1` |
| `docker.io/swaggerapi/swagger-ui` | `b26661cd754a7efbb2e1aab419ffb5603320d8f35f0bd086d1d7506e23934d23` |
| `docker.io/wiremock/wiremock:3.13.2` | `0d4ecb3e4dc8213fd7a4d37d6a78f6e6b553a6d2e15bd51b0999781282ac61b3` |
| `quay.io/keycloak/keycloak:26.8.1` | `79c623a22abacf3fe6c00a5fb30f5b7843d717b234033b80995891c06116d244` |
