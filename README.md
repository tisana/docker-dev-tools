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

## Local configuration and storage

Copy `.env.example` to `.env`, fill the required OpenSearch password, and supply
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
| `dynamodb.yml` / `dev-dynamodb` | `DYNAMODB_PORT=8000` | `dynamodb-data` -> `/home/dynamodblocal/data` |
| `elasticsearch.yml` / `dev-elasticsearch` | `ELASTICSEARCH_PORT=9200`, `ELASTICSEARCH_TRANSPORT_PORT=9300`, `KIBANA_PORT=5601` | `elasticsearch-data` -> `/usr/share/elasticsearch/data` |
| `grafana.yml` / `dev-grafana` | `GRAFANA_PORT=3000`, `GRAFANA_POSTGRES_PORT=5432` | `grafana-db-data` -> `/var/lib/postgresql/data/pgdata`; `grafana-data` -> `/var/lib/grafana` |
| `jenkins.yml` / `dev-jenkins` | `JENKINS_PORT=8080`, `JENKINS_AGENT_PORT=50000` | bind -> `/var/jenkins_home` |
| `kafdrop.yml` / `dev-kafdrop` | `KAFDROP_PORT=9000` |  |
| `kafka.yml` / `dev-kafka` | `ZOOKEEPER_PORT=2181`, `KAFKA_PORT=9092`, `KAFKA_PORT=9092`, `SCHEMA_REGISTRY_PORT=8081` |  |
| `keycloak.yml` / `dev-keycloak` | `KEYCLOAK_PORT=8080`, `KEYCLOAK_MANAGEMENT_PORT=9990` | bind -> `/var/lib/postgresql/data` |
| `metabase.yml` / `dev-metabase` | `METABASE_PORT=3000` | `metabase-data` -> `/metabase-data` |
| `mongodb.yml` / `dev-mongodb` | `MONGODB_PORT=27017`, `MONGO_EXPRESS_PORT=8081` | `mongodb-data` -> `/data/db` |
| `monitoring.yml` / `dev-monitoring` | `PROMETHEUS_PORT=9090`, `MONITORING_GRAFANA_PORT=3000` | bind -> `/etc/grafana/provisioning`; `monitoring-grafana-data` -> `/var/lib/grafana`; bind -> `/etc/prometheus`; `prometheus-data` -> `/prometheus` |
| `mysql.yml` / `dev-mysql` | `MYSQL_PORT=3306`, `MYSQL_ADMINER_PORT=8080` | bind -> `/var/lib/mysql` |
| `mysql8.yml` / `dev-mysql8` |  | `mysql8-data` -> `/var/lib/mysql` |
| `nexus.yml` / `dev-nexus` | `NEXUS_PORT=8081` | bind -> `/nexus-data` |
| `odoo.yml` / `dev-odoo` | `ODOO_PORT=8069` | `odoo-db-data` -> `/var/lib/postgresql/data/pgdata`; `odoo-web-data` -> `/var/lib/odoo`; bind -> `/etc/odoo`; bind -> `/mnt/extra-addons` |
| `opensearch.yml` / `dev-opensearch` | `OPENSEARCH_PORT=9200`, `OPENSEARCH_ANALYZER_PORT=9600`, `OPENSEARCH_DASHBOARDS_PORT=5601` | `opensearch-data1` -> `/usr/share/opensearch/data`; `opensearch-data2` -> `/usr/share/opensearch/data` |
| `phpmyadmin.yml` / `dev-phpmyadmin` | `PHPMYADMIN_PORT=80` | `phpmyadmin_session` -> `/sessions` |
| `postgresql.yml` / `dev-postgresql` | `POSTGRES_PORT=5432` | `postgresql` -> `/var/lib/postgresql/data` |
| `redisinsight.yml` / `dev-redisinsight` | `REDISINSIGHT_PORT=8001` | `redisinsight` -> `/db` |
| `sonarqube.yml` / `dev-sonarqube` | `SONARQUBE_PORT=9001`, `SONARQUBE_LEGACY_PORT=9092` | `sonarqube_postgresql` -> `/var/lib/postgresql`; `sonarqube_conf` -> `/opt/sonarqube/conf`; `sonarqube_data` -> `/opt/sonarqube/data`; `sonarqube_extensions` -> `/opt/sonarqube/extensions`; `sonarqube_logs` -> `/opt/sonarqube/logs`; `sonarqube_bundled-plugins` -> `/opt/sonarqube/lib/bundled-plugins` |
| `swagger.yml` / `dev-swagger` | `SWAGGER_PORT=8001` | bind -> `/usr/share/nginx/html/swagger` |
| `tomcat7.yml` / `dev-tomcat7` | `TOMCAT_PORT=8080` | bind -> `/usr/local/tomcat/logs` |
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
  Legacy Kafka storage remains ephemeral pending the planned KRaft migration;
  export needed topics/messages and schemas before stopping its project.
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
- Keycloak, Odoo, Jenkins and Nexus retain their existing bind/volume layouts.
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
