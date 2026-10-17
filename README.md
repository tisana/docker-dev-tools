# Dev Tools Compose Stacks

Development tools packaged as standard Compose stacks. The files in this
repository use the Compose Specification. Docker Compose is tested; Podman
Compose and other runners require separate runtime validation.

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
or override host ports and set distinct project names for isolated resources:

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

The three-phase upgrade is implemented and verified on Docker Compose. See [the validation record](docs/compose-upgrade-validation.md)
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

## Local configuration and storage

Copy `.env.example` to `.env`; fill the required OpenSearch, Keycloak and mongo-express passwords, generate a stable Kafka cluster ID, and supply
WireMock's upstream and Kafdrop's broker before starting those helpers. Bindings
default to `127.0.0.1`; changing `DEV_BIND_ADDRESS` exposes the selected tools on
that address. Development defaults are not deployment credentials.

Use the project names and port settings below. `-p` isolates named volumes and
networks, but cannot prevent host-port collisions or isolate `./data` bind paths.
For two instances of the same bind-mounted helper, use an override with separate
absolute source directories. Run tests against copies, never original bind data.

```powershell
$env:POSTGRES_PORT = '15432'
docker compose -p dev-postgresql -f postgresql.yml up -d --wait --wait-timeout 120
docker compose -p dev-postgresql -f postgresql.yml down
```

`down` preserves named volumes; `down -v` deletes them. Explicit volumes now
retain MongoDB records, MySQL 8 databases, Elasticsearch documents, monitoring
Grafana settings, Prometheus history, Metabase H2 metadata and DynamoDB tables.
DynamoDB uses a one-shot permission initializer and retains per-credential and
per-region files; use the same access-key ID and region when reconnecting.
Metabase H2 is a development database: stop Metabase before copying its files.
PostgreSQL health checks gate SonarQube, Grafana, Odoo and Keycloak; MongoDB gates
mongo-express, and Kafka gates Schema Registry. Check application readiness
through the UI/API as well; an open port or `up -d` alone is insufficient.

| Stack / project | Host port settings (defaults) | Persistent storage |
|---|---|---|
| `dynamodb.yml` / `dev-dynamodb` | `DYNAMODB_PORT=8000` | `dynamodb-v3-data` -> `/home/dynamodblocal/data` |
| `elasticsearch.yml` / `dev-elasticsearch` | `ELASTICSEARCH_PORT=9200`, `ELASTICSEARCH_TRANSPORT_PORT=9300`, `KIBANA_PORT=5601` | `elasticsearch-data` -> `/usr/share/elasticsearch/data` |
| `grafana.yml` / `dev-grafana` | `GRAFANA_PORT=3000`, `GRAFANA_POSTGRES_PORT=5432` | `grafana-db-v17-data` -> `/var/lib/postgresql/data/pgdata`; `grafana-v13-data` -> `/var/lib/grafana` |
| `jenkins.yml` / `dev-jenkins` | `JENKINS_PORT=8080`, `JENKINS_AGENT_PORT=50000` | bind -> `/var/jenkins_home` |
| `kafdrop.yml` / `dev-kafdrop` | `KAFDROP_PORT=9000` | Ephemeral; external dependency |
| `kafka.yml` / `dev-kafka` | `KAFKA_PORT=9092`, `SCHEMA_REGISTRY_PORT=8081` | `kafka-kraft-data` -> `/var/lib/kafka/data` |
| `keycloak.yml` / `dev-keycloak` | `KEYCLOAK_PORT=8080` | bind -> `/var/lib/postgresql/data` |
| `metabase.yml` / `dev-metabase` | `METABASE_PORT=3000` | `metabase-data` -> `/metabase-data` |
| `mongodb.yml` / `dev-mongodb` | `MONGODB_PORT=27017`, `MONGO_EXPRESS_PORT=8081` | `mongodb-v8-data` -> `/data/db` |
| `monitoring.yml` / `dev-monitoring` | `PROMETHEUS_PORT=9090`, `MONITORING_GRAFANA_PORT=3000` | bind -> `/etc/grafana/provisioning`; `monitoring-grafana-v13-data` -> `/var/lib/grafana`; bind -> `/etc/prometheus`; `prometheus-v3-data` -> `/prometheus` |
| `mysql.yml` / `dev-mysql` | `MYSQL_PORT=3306`, `MYSQL_ADMINER_PORT=8080` | bind -> `/var/lib/mysql` |
| `nexus.yml` / `dev-nexus` | `NEXUS_PORT=8081` | bind -> `/nexus-data` |
| `odoo.yml` / `dev-odoo` | `ODOO_PORT=8069` | `odoo-db-v17-data` -> `/var/lib/postgresql/data/pgdata`; `odoo-web-v20-data` -> `/var/lib/odoo`; bind -> `/etc/odoo`; bind -> `/mnt/extra-addons` |
| `opensearch.yml` / `dev-opensearch` | `OPENSEARCH_PORT=9200`, `OPENSEARCH_ANALYZER_PORT=9600`, `OPENSEARCH_DASHBOARDS_PORT=5601` | `opensearch-data1` -> `/usr/share/opensearch/data`; `opensearch-data2` -> `/usr/share/opensearch/data` |
| `phpmyadmin.yml` / `dev-phpmyadmin` | `PHPMYADMIN_PORT=80` | `phpmyadmin_session` -> `/sessions` |
| `postgresql.yml` / `dev-postgresql` | `POSTGRES_PORT=5432` | `postgresql` -> `/var/lib/postgresql/data` |
| `redisinsight.yml` / `dev-redisinsight` | `REDISINSIGHT_PORT=8001` | `redisinsight-v3` -> `/data` |
| `sonarqube.yml` / `dev-sonarqube` | `SONARQUBE_PORT=9001` | `sonarqube_postgresql` -> `/var/lib/postgresql`; `sonarqube_data` -> `/opt/sonarqube/data`; `sonarqube_extensions` -> `/opt/sonarqube/extensions`; `sonarqube_logs` -> `/opt/sonarqube/logs` |
| `swagger.yml` / `dev-swagger` | `SWAGGER_PORT=8001` | bind -> `/usr/share/nginx/html/swagger` |
| `tomcat.yml` / `dev-tomcat` | `TOMCAT_PORT=8080` | bind -> `/usr/local/tomcat/logs` |
| `wiremock.yml` / `dev-wiremock` | `WIREMOCK_PORT=8088` | bind -> `/home/wiremock` |

## Dependency addresses and setup

Host clients use the published host ports; services on the same Compose network
use service names and unchanged container ports. Docker Desktop provides
`host.docker.internal` for a host upstream. Reachability also depends on host
binding and firewall rules. Podman gateway routing is unverified on this machine.

- WireMock recording requires `WIREMOCK_PROXY_TARGET`, for example
  `http://host.docker.internal:8080` for a host app or `http://upstream:8080` for a
  service on the same network. Its own host port is not an upstream. Keep
  `mappings/` and `__files/` together when backing up recorded responses.
- Kafdrop standalone requires `KAFKA_BROKERCONNECT` pointing to a broker reachable
  from its container. For combined use:
  `docker compose -p dev-kafka -f kafka.yml -f kafdrop.yml up -d` with
  `KAFKA_BROKERCONNECT=kafka:29092`. Separate projects do not share DNS.
- Kafka advertises `KAFKA_ADVERTISED_HOST` (default `localhost`) and `KAFKA_PORT`
  to host clients. For remote/container host routing choose a reachable hostname
  and a suitable bind address. The internal broker remains `kafka:29092`.
  Kafka persists in `kafka-kraft-data`; keep its cluster ID with that volume.
- Monitoring Grafana uses `http://prometheus:9090`. Prometheus's
  `host-application` job expects the host application on port 8081 at
  `/management/prometheus`; edit the target for your application. This is not a
  scrape of Prometheus itself. Grafana's initial development login is admin/admin.
- PostgreSQL's development database/user/password are metric/metric/metric.
  SonarQube uses sonar/sonar in PostgreSQL; its initial UI login is admin/admin.
  MongoDB's development root credentials are in `mongodb.yml`; authenticate
  against `admin`. MySQL's root development password is in the relevant stack.
- Metabase requires first-run setup. phpMyAdmin accepts an explicitly reachable
  MySQL hostname; it serves HTTP only, and the unused HTTPS port was removed.
  Adminer server names are `mysql` on their own networks. Redis Insight is a UI
  and requires a separately reachable Redis server. Standalone Grafana uses
  PostgreSQL for durable settings; its Redis cache is intentionally disposable.
- Keycloak, Odoo and Jenkins use new version-specific storage for major upgrades; Nexus retains its bind path. See the migration notes below.
  Back up Odoo's PostgreSQL database, filestore, configuration and addons as a
  matched set. Back up Jenkins home including jobs/plugins/configuration, and
  Nexus data including artifact metadata. Never change a database major on these
  existing paths without the supported migration and a restoration test.
- OpenSearch requires a strong `OPENSEARCH_INITIAL_ADMIN_PASSWORD`. Search stacks
  require enough memory, suitable `vm.max_map_count` and file descriptor limits.
  Do not assume edited image tags mean old search data has already been migrated.

For recovery, keep the pre-change configuration and exact image digest with the
backup, restore to a dedicated project and copied storage, verify recognizable
records/settings, then migrate the originals. Never restore an older image over
an upgraded database directory. Keep dumps, plugins and runtime data out of Git.

## KRaft development broker

Kafka and Schema Registry now use matching Confluent 8.3.2 images. ZooKeeper and
its port are removed. Generate a cluster ID once and put it in `.env`:

```powershell
docker run --rm --entrypoint kafka-storage docker.io/confluentinc/cp-kafka:8.3.2 random-uuid
# KAFKA_CLUSTER_ID=<generated value>
$env:KAFKA_BROKERCONNECT = 'kafka:29092'
docker compose -p dev-kafka -f kafka.yml -f kafdrop.yml up -d
```

Retain the cluster ID with `kafka-kraft-data`; changing it does not migrate data.
The broker/controller is a single-node development configuration. Schemas live
in Kafka's `_schemas` topic, so broker persistence also protects Schema Registry.
For retained 5.5 data, keep its old image/configuration and storage together.
Use the documented Confluent intermediate upgrade/KRaft bridge or export and
replay at the application level into a fresh cluster. Do not attach 5.5 files to
8.x. Preserve keys, timestamps, headers, schema versions and references in your
export. Re-register referenced schemas first; schema IDs can change and encoded
payloads may require re-encoding. Consumer offsets are not preserved by ordinary
export/replay. Validate dependent clients before switching endpoints. Rollback
restores the old cluster and its matching backup, not an 8.x log downgrade.

## Keycloak and Odoo upgrades

Keycloak 26.8.1 uses PostgreSQL 17.11 and `start-dev`. Bootstrap credentials are
`KEYCLOAK_ADMIN_USER` and required `KEYCLOAK_ADMIN_PASSWORD`. Its default URLs are
`/realms/<realm>` and `/admin`; legacy clients using `/auth` need their issuer,
redirect and discovery configuration updated. The obsolete management port is
removed. New PostgreSQL data defaults to `./data/keycloak_pg17`, leaving the old
`./data/keycloak_data` PostgreSQL 12 installation untouched. On Windows bind
mounts, initial database initialization may take several minutes; use
`--wait --wait-timeout 360` and inspect logs if readiness fails.

For a populated Keycloak 12 installation, retain its exact image, database dump
and realm exports; rehearse the documented intermediate Keycloak/schema upgrades
and a PostgreSQL logical restore on isolated copies. A realm import is not a
substitute for preserving all server metadata. Do not point PostgreSQL 17 at the
old PostgreSQL 12 directory. The tests here cover a fresh modern installation
with an imported realm, client and user; no populated legacy installation was
available to migrate.

Odoo uses 20.0-20260926 with PostgreSQL 17.11, explicit container HTTP binding,
`odoo-web-v20-data` for its filestore and `odoo-db-v17-data` for PostgreSQL. Old
`odoo-web-data` / `odoo-db-data` volumes remain untouched. Configuration and addons
still use bind mounts; `ODOO_CONFIG_DIR` and `ODOO_ADDONS_DIR` allow copied paths.
An existing Odoo 13 database requires an application migration and compatible
addons; a PostgreSQL dump alone cannot upgrade its schema. Migrate and restore
the database, filestore, configuration and addons together, verify business
records and attachments, then switch clients. Unported custom addons block that
installation's upgrade. The fixture used the installed base addon; no original
custom addons or database were present. Odoo 20 attachment clients must use the
new `raw` binary contract instead of the removed `datas` field.

## Database helpers and Redis Insight

MongoDB now uses 8.0.34 and a separate `mongodb-v8-data` volume. For retained
6.0 files, back up while stopped, work on a copy, upgrade to 7.0, set FCV 7.0
with confirmation, stop cleanly, then upgrade to 8.0 and set FCV 8.0. Compare
records and indexes at every step. Preserve the pre-upgrade volume for rollback;
never run 8.0 directly on 6.0 files or downgrade upgraded files in place.

`mysql.yml` is the single MySQL entry point, using 8.4.11 LTS and Adminer 6.1.2.
It keeps its
case-insensitive initialization policy (`lower_case_table_names=1`) and uses
`utf8mb4` in the new `./data/mysql84` bind directory. The duplicate `mysql8.yml`
has been removed; use `MYSQL_PORT` and `MYSQL_ADMINER_PORT` for port overrides.
Existing project volumes ending in `_mysql8-data` are not deleted or
automatically imported. Export their databases using the old configuration from
Git history, then restore logically into the retained helper after checking
table-name collisions under its case-insensitive policy. Do not copy the old
data directory into `MYSQL_DATA_DIR` to change case policy. That setting
cannot be changed on an initialized database by editing the startup flag.
For 5.7 data, rehearse 5.7 -> 8.0 -> 8.4 using the supported upgrade checks or
logical user-database exports/restores. Export routines/events/triggers too;
review users/grants and authentication separately. Current clients must support
8.4 authentication. The obsolete `--ssl` flag was removed; no TLS host mapping
is implied by that flag.

DynamoDB Local 3.3.1 uses `dynamodb-v3-data`; copy stopped SQLite files to the new
volume only after backup and test on that copy. Keep the access-key ID/region
consistent. Redis Insight 3.8.0 uses `docker.io/redis/redisinsight`, `/data` in
`redisinsight-v3`, and host port 8001 -> container 5540. It requires a Redis server.
Export old 1.x connections before retiring its `/db` volume; the new helper uses
fresh storage rather than claiming automatic metadata conversion.

## Grafana / Prometheus migration

Both Grafana helpers use 13.2.4; monitoring uses Prometheus 3.15.0. Standalone
Grafana uses PostgreSQL 17.11 and disposable Redis 8.2.10 cache. It does not mount
the monitoring datasource. New durable volumes are version-specific; old volumes
are retained for recovery. Back up Grafana's database, data/plugins and config
before migration. Rehearse intermediate Grafana upgrades and PostgreSQL logical
backup/restore on copies before changing a populated installation.

Monitoring's Prometheus storage path is explicitly `/prometheus/data`, matching
the old helper's on-disk layout inside its volume. Copy stopped storage into
`prometheus-v3-data` and verify historical timestamps before switching. Preserve
the old image and old storage together for rollback. Scrape fixtures/exporters
must emit valid Prometheus text/OpenMetrics with correct Content-Type and LF
line termination. The bundled JVM dashboard uses supported stat/time-series
panels; queries, units, thresholds, active value mappings and variables were
preserved. The unused legacy pie-chart plugin is no longer installed.

## Jenkins, Tomcat and WireMock

Jenkins now uses 2.580.1 LTS/JDK21 with a separate default home at
`./data/jenkins_lts21` (`JENKINS_HOME_DIR`). Complete its normal setup wizard on
first launch. For an existing installation, back up the complete old home,
rehearse the required intermediate core/plugin upgrades on a copy, and verify
jobs and agents before switching. Jenkins' runtime JDK is separate from the
JDK/toolchains used by builds. Restore the old core and matching home together
for rollback; no old home is silently opened by the new default.

`tomcat.yml` replaces `tomcat7.yml` and uses Tomcat11.0.27/JDK21. Start it with
`docker compose -p dev-tomcat -f tomcat.yml up -d`; deploy a compatible WAR to
`/usr/local/tomcat/webapps` or supply a deployment bind override. Legacy
`javax.servlet` applications require migration to Jakarta APIs. The helper's
root URL may return404 until an application is deployed.

WireMock uses official 3.13.2 and retains `/home/wiremock` mappings/files. Back up
extensions and recordings before upgrading, and validate custom extensions
against3.x. A fresh proxy request, recording and offline replay were tested with
the required upstream setting. No original custom extensions were present.

## Search helpers

Elasticsearch/Kibana use9.5.5; OpenSearch/Dashboards use3.9.0. Standalone
PostgreSQL remains on16.15. Inspect the actual search data version before
upgrading a populated installation; follow supported intermediate upgrades or
snapshot/reindex procedures on copies. New tags do not migrate old files.
OpenSearch requires a strong `OPENSEARCH_INITIAL_ADMIN_PASSWORD`, adequate VM
memory for two nodes plus Dashboards, and `vm.max_map_count >=262144`. Check the
Linux VM's value before startup; change host settings only through your engine's
documented procedure. Demo TLS is self-signed. Dashboards login is `admin` with
the supplied password. Elasticsearch's bundled helper has security disabled for
local development. See the validation record for snapshot/recreation evidence.

## Floating application and UI images

The `latest` policy is retained for Metabase, Nexus, SonarQube, Kafdrop,
mongo-express, phpMyAdmin and Swagger UI; Sonar's PostgreSQL18 major tag also
floats. Acceptance applies to the digests resolved on2026-10-16 in the
[validation record](docs/compose-upgrade-validation.md). Repeat those checks
after pulling different digests, using copied application data first.

SonarQube latest resolves to Community Build26.10 and supports PostgreSQL18.
Only port9000 is required (host9001). Before retiring old `conf` and
`lib/bundled-plugins` mounts, export local settings and inventory plugins;
apply settings through supported `SONAR_*` environment variables and install
compatible extensions. Old volumes are not deleted. Data, extensions and logs
remain mounted. The real scanner analysis and results survived recreation.

Metabase's `MB_DB_FILE` persists H2 beneath `/metabase-data`; back up the whole
volume with Metabase stopped, including its `metabase.db` subdirectory. First-run
setup creates the administrator. H2 is intended for this development helper.

Nexus latest resolves to Community Edition3.97/Java21. Complete onboarding and
review/accept its [CE EULA](https://links.sonatype.com/products/nxrm/ce-eula)
before artifact transfers; startup alone does not enable uploads/downloads.
Read the generated initial admin password from `/nexus-data/admin.password`,
then change it during setup. The retained bind must be writable by UID200 on
Linux; the current image sizes JVM memory in its launcher. Allocate adequate VM
memory and follow [vendor upgrade steps](https://help.sonatype.com/en/ce-onboarding.html)
for old OrientDB/H2 installations. No original Nexus database existed here.

mongo-express requires `MONGO_EXPRESS_PASSWORD`; username defaults to `admin`.
This web password is distinct from the development MongoDB credentials. The
tested1.0.2 image reads `ME_CONFIG_BASICAUTH`; recheck that name after a new pull,
since upstream development has changed it. The UI is for private development.

Kafdrop uses a256MB maximum heap and512KB stack compatible with its current
Java25 runtime. For standalone use, the broker must advertise an endpoint the
Kafdrop container can reach. The tested Docker Desktop host route used
`KAFKA_ADVERTISED_HOST=host.docker.internal`, `KAFKA_PORT=19092`, and an explicit
`DEV_BIND_ADDRESS=0.0.0.0` on the broker; the separate Kafdrop project used
`KAFKA_BROKERCONNECT=host.docker.internal:19092`. Limit broader bindings to the
required development network. Combined use retains `kafka:29092`.

phpMyAdmin accepts a reachable server on its login screen; combine it with
`mysql.yml` under the same project to use server `mysql`, overriding host ports
if needed. Its session files persist in `/sessions`, but the image regenerates
its cookie secret when recreated, so expect to log in again. HTTP only is
published. Swagger UI serves the read-only local `swagger/sample.yml` using
`API_URL`; the supplied sample's title is Swagger Petstore, but its document is
fetched locally, not from the remote default.

Docker Desktop linux/amd64 is the tested environment. Podman is unavailable;
its dependency conditions, bind ownership and host gateway routing remain
unverified. No populated original Keycloak12/Odoo13/Jenkins/plugin installations
were available. Their migration instructions are prerequisites, not a claim
that arbitrary existing application data or custom addons were converted.
