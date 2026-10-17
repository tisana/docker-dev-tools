# Development Compose Upgrade Implementation Plan

> **For agentic workers:** Use `superpowers:executing-plans` for inline execution, or `superpowers:subagent-driven-development` if the user selects delegated execution. Complete tasks in phase order and track the checkboxes below.

**Goal:** Make the development helpers reliable across container recreation, modernize obsolete software, and verify floating images against their actual current configuration contracts.

**Follow-up (2026-10-17):** The user requested removal of duplicate `mysql8.yml`.
`mysql.yml` is now the sole MySQL entry point, leaving 21 active Compose files.
This supersedes the original instruction to retain both entry points; the
completed checklist below records the original upgrade and its historical tests.

**Follow-up (2026-10-17):** The user requested standalone PostgreSQL18.
`postgresql.yml` now targets18.6 using separate `postgresql-v18-data` storage
mounted at `/var/lib/postgresql`. This supersedes Task2.6's original requirement
to retain major16; the original16 tests below remain historical evidence.

**Architecture:** Preserve the independent Compose stacks and their service-name networking. Repair storage and connectivity first, upgrade related services together second, and audit current floating images last. Each stack gets an isolated runtime check and a documented recovery path.

**Tech stack:** Compose Specification, Docker Compose, Podman Compose, PowerShell, and the databases/development tools in the 22 root Compose files.

**Spec:** The repository assessment and the user's three-phase request in this conversation, dated 2026-10-14. This document captures the execution requirements; it does not authorize execution or data deletion.

## Global constraints

- Execute **Phase 1 -> Phase 2 -> Phase 3**. Complete each phase's exit gate before starting the next.
- Preserve the existing uncommitted changes in `elasticsearch.yml` and `opensearch.yml`; the assessment uses those working versions.
- Keep fully qualified image names, omit the obsolete Compose `version` key, and avoid fixed container names.
- Keep the current host ports as defaults, except remove nonfunctional mappings. Make host bindings and ports configurable; default development bindings to `127.0.0.1`.
- Use a distinct project name per stack. Project names isolate named resources, but not host ports or bind-mounted directories.
- Inspect actual mounts and data before changing them. A new project name alone does not protect `./data/...` bind mounts: runtime tests must override them to isolated copies or temporary directories.
- Never attach a new database major version to an old data directory without its supported migration procedure. Never use `down -v`, volume pruning, or a destructive reset against existing user data without explicit authorization.
- Back up application data, database data, configuration and plugins together where applicable. Verify restoration on a copy before migrating the originals. Rollback means restoring the old image **and** its matching pre-upgrade data, not downgrading an upgraded database in place.
- Treat version numbers from the assessment as a dated starting point. Before each upgrade, verify official release/support documentation, registry tag availability, architecture support and the required upgrade path; record the chosen exact tag and digest.
- During Phase 1, test existing versions/digests where possible. If a floating image must be pulled, record its digest and check only the compatibility required for the Phase 1 repair; the full audit remains in Phase 3.
- For Phase 3, retain the existing floating-image policy by default. Record the resolved version/digest and test date; do not describe a mutable `latest` tag as permanently validated.
- Do not commit credentials, database dumps or generated runtime data. Update `.gitignore` for any new local-only paths.
- Use focused configuration and runtime checks. Do not introduce a unit-test framework for YAML edits. Commit each independently verified task when execution is authorized; do not push automatically.

## File map

| Files | Responsibility and phase |
|---|---|
| `postgresql.yml`, `sonarqube.yml` | PostgreSQL storage and SonarQube JDBC repair in Phase 1; SonarQube floating-image audit in Phase 3 |
| `metabase.yml`, `mongodb.yml`, `mysql8.yml`, `dynamodb.yml`, `elasticsearch.yml`, `monitoring.yml` | Make intended persistence explicit in Phase 1; image upgrades or conformity checks in later phases |
| `wiremock.yml`, `kafdrop.yml`, `kafka.yml`, `phpmyadmin.yml` | Connectivity repair in Phase 1; modernization or floating-image audit later |
| `keycloak.yml`, `odoo.yml`, `jenkins.yml`, `redisinsight.yml`, `mysql.yml`, `grafana.yml`, `tomcat7.yml`, `opensearch.yml` | Existing storage inventory in Phase 1; modernization in Phase 2 |
| `nexus.yml`, `swagger.yml` | Existing mount/connectivity inventory in Phase 1; floating-image audit in Phase 3 |
| `prometheus/prometheus.yml`, `grafana/provisioning/datasources/datasource.yml`, `grafana/provisioning/dashboards/dashboard.yml`, `grafana/provisioning/dashboards/JVM.json` | Monitoring connectivity in Phase 1 and dashboard/provisioning migration in Phase 2 |
| `README.md`, new `.env.example`, `.gitignore` | Per-stack commands, port overrides, credentials, persistence and migration guidance, updated alongside each task |
| New `docs/compose-upgrade-validation.md` | Execution evidence: tested versions/digests, engines, migration results, smoke tests and unresolved limitations |

## Review focus

1. **Previously populated volumes:** records must survive migration and `down`/`up` without volume deletion. Owned by Tasks 1.1-1.2 and every stateful Phase 2 task.
2. **Two stacks using the same default ports or service names:** project and host-port overrides must permit intentional concurrent use without sharing data. Owned by Task 1.4.
3. **Host clients versus container clients:** advertised endpoints must work from both locations; a successful TCP connection alone is insufficient. Owned by Tasks 1.3 and 2.1.
4. **Image upgraded against old application metadata/plugins:** saved configuration must load and plugins must function, or the migration must report a specific incompatibility. Owned by Tasks 2.2-2.5 and 3.1-3.2.
5. **A different image under the same floating tag:** acceptance applies only to the recorded digest, with the same checks rerun after the next pull. Owned by all Phase 3 tasks.

Execution completed2026-10-16 on Docker Compose. Checked items describe the
implementation and the available isolated fixtures. Conditional migrations of
absent original installations and Podman execution remain unverified; see
[the acceptance record](../../compose-upgrade-validation.md) for exact scope.

## Phase 1 — Fix persistence and connectivity

**Outcome:** Data location is predictable, container recreation is safe for the intended persistent state, and each helper can reach its dependencies.

### Task 1.1 — Inventory data and repair PostgreSQL/SonarQube

**Files:** `postgresql.yml`, `sonarqube.yml`, `README.md`, `docs/compose-upgrade-validation.md`.

- [x] Record running containers, installed image IDs/digests, named/anonymous volumes, bind paths, effective `PGDATA`, database versions and whether each stack contains data worth retaining. Use inspection commands without printing credentials into the report.
- [x] Back up and restore a sample of each affected database on isolated storage. Locate PostgreSQL 16's actual anonymous data volume before changing the mount; do not assume the existing named volume contains its records.
- [x] For PostgreSQL 16, mount its dedicated named volume at `/var/lib/postgresql/data`. Retain major 16 in this phase and restore/copy existing data using a compatible procedure.
- [x] Replace `SONARQUBE_JDBC_URL` with `SONAR_JDBC_URL`; set the intended `sonar` database explicitly. Prove SonarQube uses PostgreSQL rather than silently relying on embedded H2. If existing H2 data is found, preserve it and report the supported transfer limitations before switching that installation.
- [x] For SonarQube's PostgreSQL 18, retain the `/var/lib/postgresql` mount and remove the redundant child mount only after inspecting and preserving any data there. Use a SonarQube-specific volume name to avoid accidental overlap with the standalone stack.
- [x] Add PostgreSQL readiness checks and dependency readiness handling supported by the selected Compose runners.
- [x] Validate SQL writes, application connectivity, `down`/`up` persistence, and restoration of the pre-change backup. Record results and commit this task independently.

### Task 1.2 — Define and implement persistence for the remaining stacks

**Files:** `metabase.yml`, `mongodb.yml`, `mysql8.yml`, `dynamodb.yml`, `elasticsearch.yml`, `monitoring.yml`, `README.md`; inspect existing mounts in all other stacks.

**Default policy:** Preserve user-created development data across normal `down`/`up`. Caches and replaceable runtime files may remain ephemeral. Existing bind mounts remain supported; do not relocate them merely for consistency.

- [x] Add explicit named data volumes for MongoDB (`/data/db`), MySQL 8 (`/var/lib/mysql`), Elasticsearch (`/usr/share/elasticsearch/data`), monitoring Grafana (`/var/lib/grafana`) and Prometheus (`/prometheus`). Recover existing state from actual containers/volumes before replacement.
- [x] Persist DynamoDB Local in a dedicated writable data directory and set `-dbPath`. Preserve the existing per-credential/per-region database behavior; do not introduce `-sharedDb` without a documented data/endpoint transition.
- [x] For Metabase, persist the H2 application database with an explicit `MB_DB_FILE` and volume. Copy existing H2 files while Metabase is stopped and verify saved questions survive. Keep an external-database migration outside this initial storage repair.
- [x] Check permissions and actual storage locations for Jenkins, Nexus, Keycloak, Odoo, Redis Insight and WireMock. Preserve Odoo's database and filestore as a matched pair. Confirm Grafana's Redis cache is intentionally disposable.
- [x] Inventory Kafka topics/messages and Schema Registry schemas before Phase 2. Archive required data now; the new persistent Kafka layout is implemented with the KRaft conversion in Task 2.1.
- [x] Write one recognizable record or saved setting per affected service, run `down` without `-v`, start again, and verify that record. Confirm the named mount actually holds the files. Record the Kafka conversion exception explicitly in the Phase 1 evidence.

### Task 1.3 — Repair dependency addresses and proxy behavior

**Files:** `wiremock.yml`, `kafdrop.yml`, `kafka.yml`, `monitoring.yml`, `prometheus/prometheus.yml`, `grafana/provisioning/datasources/datasource.yml`, `phpmyadmin.yml`, `.env.example`, `README.md`.

- [x] Make WireMock's upstream a required `WIREMOCK_PROXY_TARGET` setting when recording/proxy mode is used. Replace the current container-local `localhost:8088` target. Document a host upstream using `host.docker.internal` and a container upstream using its service name.
- [x] Replace Kafdrop's fixed public IP with `KAFKA_BROKERCONNECT`. Document both combined-stack use (`kafka:29092` on the same project/network) and standalone use through a reachable published broker endpoint. Do not assume separate projects share DNS.
- [x] Verify Kafka's advertised addresses from both a host client and a container client. Parameterize the external advertised hostname/port consistently with the published port; retain the current broker generation until Task 2.1.
- [x] Verify Schema Registry reaches the broker using the settings supported by the existing Confluent version. Correct startup dependencies so it does not merely wait for ZooKeeper creation.
- [x] Keep Grafana's datasource URL at `http://prometheus:9090`. Correct the misleading Prometheus job name/comment: it currently scrapes a host application's `/management/prometheus`, not Prometheus itself. Document the required host application endpoint and test against a known fixture.
- [x] Check host-gateway resolution on Docker Desktop and the intended Podman runner. Add only the runner-specific gateway mapping that testing shows is needed; document any unsupported host/engine combination.
- [x] Remove phpMyAdmin's unused `443:443` mapping. Verify a login to a test MySQL service through a reachable hostname; publishing a port alone does not enable TLS.
- [x] Exercise WireMock proxy/record/replay against a local fixture, Kafdrop topic browsing, Schema Registry schema registration, and a Prometheus scrape visible in Grafana. Record results and commit.

### Task 1.4 — Standardize local startup and port overrides

**Files:** all 22 Compose files as needed, `README.md`, `.env.example`, `.gitignore`.

- [x] Add a shared `DEV_BIND_ADDRESS` defaulting to `127.0.0.1` and tool-specific host-port variables, preserving existing default port numbers. Use distinct names such as `POSTGRES_PORT`, `GRAFANA_PORT`, `METABASE_PORT`, `KAFKA_PORT` and `SCHEMA_REGISTRY_PORT`; list the complete names/defaults in `.env.example`.
- [x] Keep container ports unchanged unless an image contract changes. Align Kafka advertised external ports with overrides. Allow an explicit broader bind address for container-to-host or remote access where needed, and test that route.
- [x] Document a unique project name per stack, the distinction between host and container addresses, startup prerequisites, login/setup steps and data locations. Explain that `-p` does not resolve host-port collisions.
- [x] Add readiness checks for dependency-bearing services using tools actually present in their images. Use application/API probes for the rest; do not assume every image contains `curl` or a shell.
- [x] Start two otherwise colliding stacks using distinct projects and overridden ports. Confirm both are reachable and their data remains separate after independent recreation.

**Phase 1 exit gate:** All Compose files parse; corrected services pass functional connectivity checks; the persistence checks pass for each newly persistent service; existing data has a verified recovery path; Kafka's deferred storage conversion is documented. No unrelated major-version upgrades are mixed into these repairs.

## Phase 2 — Modernize obsolete stacks, then refresh newer pinned stacks

**Outcome:** Default helpers use supported software and configuration, with migrations tested on copied data. Within this phase, do Tasks 2.1-2.5 before Task 2.6.

### Task 2.1 — Replace legacy Kafka/ZooKeeper with KRaft

**Files:** `kafka.yml`, `kafdrop.yml` connection examples, `.env.example`, `README.md`.

- [x] Select a supported Confluent 8.x release (assessment baseline: 8.3.x); pin matching broker and Schema Registry releases.
- [x] Replace ZooKeeper with a single-node development KRaft broker/controller. Configure a stable cluster ID, node/controller settings, internal/external/controller listeners, single-node replication settings and a persistent broker data volume using the selected image's documented path.
- [x] Replace Schema Registry's ZooKeeper connection with Kafka bootstrap servers. Keep schemas persisted in Kafka's internal topic.
- [x] For a fresh helper, initialize an isolated cluster. For retained old data, use a documented supported bridge/migration or application-level export/import; never mount 5.5 broker data directly into 8.x. Document any consumer-offset/schema-ID limitations before migration.
- [x] Create a topic, register a schema, produce and consume from both host and container clients, inspect through Kafdrop, and repeat after recreation. Verify schema references and required messages survive.

### Task 2.2 — Modernize Keycloak and Odoo with their databases

**Files:** `keycloak.yml`, `odoo.yml`, `.env.example`, `README.md`.

- [x] Move Keycloak to a verified current 26.x release (assessment: 26.8.1), with `start-dev`, `KC_DB*` and `KC_BOOTSTRAP_ADMIN_*` settings. Remove obsolete WildFly settings and port 9990; document the changed default URL path if applications expect `/auth`.
- [x] Select a PostgreSQL major supported by that Keycloak release. Migrate PostgreSQL 12 through logical backup/restore or the documented upgrade procedure; follow Keycloak's supported schema-upgrade sequence on a copy.
- [x] Test an imported realm, an existing user, client login/token issuance, discovery URL and recreation. Confirm realm/client redirect settings remain valid.
- [x] Move Odoo to a supported major with compatible addons (assessment target: 20.0). Select a PostgreSQL version supported by that release and replace PostgreSQL 12 with a tested migration.
- [x] Treat existing Odoo databases as application migrations: migrate the database, addons and filestore together. If required custom addons cannot migrate, report that stack as blocked on addon work rather than claiming a successful upgrade from a blank database.
- [x] Test database creation, login, an installed addon, a saved business record and an attachment before/after recreation. Document recovery and commit each application separately.

### Task 2.3 — Upgrade database helpers and Redis Insight

**Files:** `dynamodb.yml`, `mongodb.yml`, `mysql.yml`, `mysql8.yml`, `redisinsight.yml`, `.env.example`, `README.md`.

- [x] Upgrade DynamoDB Local to a verified pinned 3.x release (assessment: 3.3.1). Test table creation, put/get, the application's relevant API features and retained local data. Keep port 8000.
- [x] Upgrade MongoDB 6 to a supported 8.0 release. For existing files, follow the required intermediate versions and feature-compatibility changes, or a documented compatible logical migration. Test authenticated CRUD and retained indexes; do not directly reuse 6.0 files under 8.0.
- [x] Replace MySQL 5.7 with a supported 8.4 LTS release; make `mysql8.yml` explicit about 8.4 as well. Preserve both entry points for now and document their storage/configuration differences. Do not silently delete or rename either file.
- [x] Review MySQL command flags, character set and case sensitivity. Use `utf8mb4` for new databases; preserve existing `lower_case_table_names` requirements during initialization/migration. Test authentication and mixed-case table access as applicable.
- [x] Upgrade Adminer in both MySQL stacks to a verified current supported release (assessment: 6.x). Test login, a query and compatible authentication against the selected MySQL image.
- [x] Replace Redis Insight's old repository with `docker.io/redis/redisinsight`, select an explicit stable tag, mount `/data`, and map the existing host port 8001 to container port 5540. Use a fresh volume unless a supported migration is documented; preserve/export old connection settings first.
- [x] Test Redis Insight against a temporary Redis fixture, inspect a key, and verify connection settings after recreation. The UI does not itself provide a Redis server.
- [x] For each upgraded database, compare fixture records and required indexes before/after migration and after `down`/`up`. Verify rollback by restoring the pre-upgrade backup into its matching old image on isolated storage.

### Task 2.4 — Upgrade Grafana/Prometheus and migrate dashboards

**Files:** `grafana.yml`, `monitoring.yml`, all files under `grafana/provisioning/`, `prometheus/prometheus.yml`, `README.md`.

- [x] Select supported Grafana 13.x and Prometheus 3.x releases (assessment: Prometheus 3.15.0). Use matching Grafana releases in both stacks.
- [x] Upgrade standalone Grafana's PostgreSQL 14 to a supported compatible major, with verified backup/restore. Check Redis 7's support and cache compatibility; select a supported Redis version without treating cache data as durable application state.
- [x] Remove the legacy pie-chart plugin installation and use the built-in panel. Migrate `singlestat` and legacy `graph` panels in `JVM.json` to supported panels, preserving queries, units, thresholds and variables.
- [x] Remove obsolete/irrelevant datasource fields and placeholder certificate values. Keep the standalone Grafana stack independent; only the monitoring stack should require its bundled Prometheus datasource.
- [x] Validate Prometheus configuration with the selected image's `promtool`, check required metric names/labels against a fixture, and account for Prometheus 3 scrape/format changes.
- [x] Open the provisioned dashboard, verify representative panels and variables with actual fixture metrics, inspect plugin/provisioning errors, and prove saved dashboards and historical samples survive recreation.

### Task 2.5 — Replace obsolete Jenkins, Tomcat and WireMock baselines

**Files:** `jenkins.yml`, `tomcat7.yml` -> new `tomcat.yml`, `wiremock.yml`, `README.md`.

- [x] Select a current Jenkins LTS image with JDK 21 or 25 and an explicit release tag. Test on a copy of `JENKINS_HOME`; follow required intermediate upgrades and update incompatible plugins.
- [x] Verify existing jobs load, an agent connects if used, a sample build succeeds and settings survive recreation. Keep build-tool JDK requirements distinct from Jenkins' own runtime.
- [x] Replace the default Tomcat 7 helper with a pinned supported Tomcat 11/JDK 21 image in `tomcat.yml`; remove the old default file and update its documented command. Record that legacy `javax.servlet` applications need migration to Jakarta APIs; do not claim they will run unchanged.
- [x] Deploy a small compatible test WAR and verify its endpoint and logs. If the user still needs Tomcat 7 reproduction, preserve it only as an explicitly requested legacy helper, not the modern default.
- [x] Move WireMock to a pinned stable official `docker.io/wiremock/wiremock` 3.x image. Validate the CLI flags, mappings, file ownership and any extensions against that release.
- [x] Repeat stub matching and Phase 1 proxy/record/replay checks, including replay after recreation with the upstream stopped.

### Task 2.6 — Refresh the newer pinned stacks

**Files:** `elasticsearch.yml`, `opensearch.yml`, `postgresql.yml`, `README.md`.

- [x] Refresh Elasticsearch/Kibana together to a compatible patched release (assessment current: 9.5.5). Refresh all OpenSearch/Dashboards images together (assessment current: 3.9.0). Preserve and review the existing working-tree changes.
- [x] Inspect the version of any existing search data. In particular, do not assume volumes were already migrated from Elasticsearch 7 or OpenSearch 2 just because the Compose tags were edited. Follow supported intermediate upgrades or snapshot/reindex procedures.
- [x] Test index creation, search, dashboard connection/login as applicable, and retained documents after recreation. Verify the required OpenSearch admin password and host kernel/resource prerequisites are documented.
- [x] Keep standalone PostgreSQL 16 unless application parity calls for another supported major; refresh its patch release/digest and repeat the Phase 1 persistence check. A newer major is not required merely because it exists.

**Phase 2 exit gate:** Every targeted stack runs the selected supported release, application smoke tests pass, restored data is verified, and migrations/rollback instructions are recorded. A clean first boot alone is not sufficient evidence for a populated installation. Clearly list any application/addon compatibility blocker.

## Phase 3 — Verify configuration against current floating images

**Outcome:** Every explicit or implicit `latest` image has a dated, reproducible conformity check. Fix settings only where documentation or runtime evidence shows a mismatch.

### Task 3.1 — Audit floating application images

**Files:** `metabase.yml`, `nexus.yml`, `sonarqube.yml`, `README.md`, validation record.

- [x] Resolve/pull each current image in the isolated test environment; record application version, image digest, architecture and official configuration reference. Recheck this immediately before runtime tests so a changed tag cannot invalidate the evidence.
- [x] **Metabase:** verify `MB_DB_FILE`, startup flags, filesystem permissions and the persisted application database contract. Create a question/dashboard, recreate the service and verify both. Test an upgrade using a copied database from the pre-pull image.
- [x] **Nexus:** verify current Java/memory requirements, `/nexus-data` ownership, database format and required intermediate migrations. Remove irrelevant Elasticsearch comments/settings only where unsupported or unnecessary. Upload/download a test artifact and verify it after recreation.
- [x] **SonarQube:** verify the edition behind `latest`, PostgreSQL 18 compatibility, `SONAR_JDBC_*`, memory/kernel limits and supported mount paths. Review the `conf` and `lib/bundled-plugins` mounts so stale image-owned files do not mask the current image's contents; preserve local configuration and supported extensions before changing mounts. Remove port 9092 if the selected release has no required listener there.
- [x] Run an actual SonarQube analysis, verify its project/result in PostgreSQL-backed storage, recreate the stack and confirm results remain. Check logs for ignored settings and plugin failures.

### Task 3.2 — Audit floating UI/helper images

**Files:** `kafdrop.yml`, `mongodb.yml` (mongo-express), `phpmyadmin.yml`, `swagger.yml`, `README.md`, validation record.

- [x] Record the current version/digest and documented environment variables for each image. Check any repository/name changes before replacing a reference.
- [x] **Kafdrop:** verify JVM options against its bundled JDK, replacing invalid options or inadequate memory settings based on startup/runtime evidence. Browse the Phase 2 broker's topics and messages; test its documented standalone connection mode too.
- [x] **mongo-express:** verify MongoDB URL/authentication and web-UI authentication variables against the current image. Test authenticated browsing and CRUD against the upgraded MongoDB, including a wrong-password failure.
- [x] **phpMyAdmin:** verify the current repository, `PMA_ARBITRARY`, `/sessions`, HTTP port and supported database authentication. Log in, execute a query and verify configured session behavior; confirm no unused HTTPS mapping remains.
- [x] **Swagger UI:** verify the supported specification URL variable, document-root path and serving port. Load `swagger/sample.yml` in the browser, check the network request returns that document, and confirm no fallback to the default example specification. Make the specification mount read-only if the image supports it.
- [x] Retain each floating tag unless a concrete compatibility constraint requires an explicit tag. If pinned as an exception, record the reason and the condition for returning to the floating policy.

### Task 3.3 — Complete the repository-wide acceptance record

**Files:** `README.md`, `.env.example`, `docs/compose-upgrade-validation.md`; affected Compose files only if checks expose an issue.

- [x] Scan every active image reference. Include implicit `latest` and any floating references introduced during earlier phases; ensure none escapes the conformity audit.
- [x] Parse every final Compose file using the documented environment setup. Test default ports and representative overrides, required-variable errors and dependency readiness.
- [x] Confirm all 22 original stacks have a completed entry, with `tomcat7.yml` explicitly mapped to its replacement. Include each final image tag/digest, engine/version, functional check, persistence result and source reference.
- [x] Run the documented workflows on Docker Compose and the available Podman Compose runner. Record unavailable engines as unverified; qualify README portability claims rather than claiming runtime compatibility from syntax alone.
- [x] Review the final diff for unintended changes, credentials, generated data, stale image references and obsolete comments. Record remaining limitations and commit the documentation with the relevant verified changes.

**Phase 3 exit gate:** Every floating-image stack has passed configuration and functional checks against its recorded digest, all final Compose files parse, and README commands match the tested behavior. Report any unavailable-engine or external-dependency checks separately from passes.

## Common verification procedure

Use an isolated copy of bind-mounted data, a dedicated project and unused host ports for each check. Supply the documented environment variables before these commands; do not paste secret-bearing rendered configuration into the evidence file.

```powershell
docker compose -p review-postgres -f postgresql.yml config --quiet
docker compose -p review-postgres -f postgresql.yml up -d
docker compose -p review-postgres -f postgresql.yml ps
# Run the task's SQL/API/UI assertions and save a recognizable fixture record.
docker compose -p review-postgres -f postgresql.yml down
docker compose -p review-postgres -f postgresql.yml up -d
# Repeat the assertions and read the same record.
```

For each task, record the exact commands used, expected result, actual result and any prerequisite. Use bounded readiness polling and inspect service logs on failure. `config --quiet` proves only structural validity; `up -d` proves only that startup was requested. Readiness, actual service use and persistence must be checked separately. Remove only isolated test resources after recording results.

## Reference baseline

The assessment checked these sources on 2026-10-14. Recheck the relevant source when executing its task.

- [PostgreSQL image storage rules](https://github.com/docker-library/docs/blob/master/postgres/README.md) and [support policy](https://www.postgresql.org/support/versioning/).
- [SonarQube JDBC environment variables](https://docs.sonarsource.com/sonarqube-server/2025.2/setup-and-update/environment-variables) and [Community Build database compatibility](https://docs.sonarsource.com/sonarqube-community-build/server-installation/installing-the-database).
- [Confluent versions and KRaft requirements](https://docs.confluent.io/platform/current/installation/versions-interoperability.html).
- [Keycloak container configuration](https://www.keycloak.org/server/containers), [Odoo image/migration guidance](https://hub.docker.com/_/odoo), [Jenkins Java requirements](https://www.jenkins.io/doc/book/platform-information/support-policy-java/).
- [MongoDB lifecycle](https://www.mongodb.com/legal/support-policy/lifecycles), [MySQL images](https://hub.docker.com/_/mysql/), [DynamoDB Local release history](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocalHistory.html).
- [Redis Insight Docker configuration](https://redis.io/docs/latest/operate/redisinsight/install/install-on-docker/), [WireMock Docker configuration](https://wiremock.org/docs/standalone/docker/).
- [Grafana release notes](https://grafana.com/docs/grafana/latest/whatsnew/), [Prometheus releases](https://prometheus.io/download/), [Elastic releases](https://www.elastic.co/docs/release-notes), [OpenSearch releases](https://opensearch.org/releases/).
- [Metabase Docker guide](https://www.metabase.com/docs/latest/installation-and-operation/running-metabase-on-docker), [Nexus image documentation](https://github.com/sonatype/docker-nexus3), [Kafdrop](https://github.com/obsidiandynamics/kafdrop), [phpMyAdmin image](https://hub.docker.com/_/phpmyadmin), [Swagger UI installation](https://github.com/swagger-api/swagger-ui/blob/main/docs/usage/installation.md).
