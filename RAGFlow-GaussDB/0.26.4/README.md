[RAGFlow-GaussDB](https://github.com/huaweicloud-samples/database-ragflow/tree/gaussdb-adaptation-combined)

# Configure GaussDB as the metadata database and DocEngine

This guide explains how to use an existing GaussDB instance for both of the
following RAGFlow storage roles:

- **Metadata database**: Stores users, tenants, datasets, documents, tasks,
  model settings, and other relational application data.
- **DocEngine**: Stores document chunks, document metadata, and retrieval
  indexes. When GaussDB is the DocEngine, the Memory message store uses the
  same GaussDB connection and schema.

RAGFlow does not deploy or administer GaussDB. Prepare the database, users,
schemas, network access, and VectorDB capability before starting RAGFlow.

## Understand the two configuration paths

The metadata database and DocEngine are selected and configured independently:

| Storage role | Selector | Connection settings | Connection pool |
| --- | --- | --- | --- |
| Metadata database | `DB_TYPE=gaussdb` | `GAUSSDB_METADATA_*` environment variables | Peewee metadata pool |
| DocEngine and Memory message store | `DOC_ENGINE=gaussdb` | `GAUSSDB_*`, rendered as the flat `gaussdb:` block in `service_conf.yaml` | Shared GaussDB DocEngine pool |

Setting only `DB_TYPE=gaussdb` does not change the DocEngine. Setting only
`DOC_ENGINE=gaussdb` does not change the metadata database.

The two roles can use different GaussDB instances or databases. If they share
one database, use different schemas and preferably different users, for
example:

- Metadata database: `ragflow.ragflow_meta`, owned by `ragflow_meta`.
- DocEngine: `ragflow.ragflow_doc`, owned by `ragflow_doc`.

:::warning
Changing either selector does not migrate existing data. Do not point an
existing deployment at an empty GaussDB database unless you have planned and
validated the data migration.
:::

## Prerequisites

### GaussDB compatibility mode

Use an Oracle-compatible database:

- **Centralized GaussDB**: `SHOW sql_compatibility` must return `A`.
- **Distributed GaussDB**: `SHOW sql_compatibility` must return `ORA`.

The compatibility mode is chosen when the database is created and cannot be
changed for an existing database.

Use UTF-8 for a new database. RAGFlow forces `client_encoding=UTF8` for its
GaussDB connections. A server using `SQL_ASCII` can be connected, but it does
not validate or convert stored bytes and is not recommended for a new
deployment.

### VectorDB capability

The DocEngine and Memory message store create `floatvector` columns and
`gsdiskann` vector indexes. A database administrator must set
`enable_vectordb=on` for the instance before RAGFlow creates vector indexes.

For a distributed cluster, enable it on every required CN and DN according to
your GaussDB version and deployment method. Reconnect after changing the
parameter, then verify:

```sql
SHOW enable_vectordb;
```

The result must be `on`. A session-only setting is not a substitute for the
node-level configuration required by a distributed cluster.

### Network and driver protocol

- Every RAGFlow API, worker, and data-sync process must be able to reach the
  configured GaussDB host and port.
- Use the GaussDB PostgreSQL-compatible client endpoint.
- Do not use `127.0.0.1` as the host from a Docker container unless GaussDB is
  running in that same container. Use a routable address such as the host IP,
  a private DNS name, or `host.docker.internal` where supported.

## Prepare the database

The following example uses one database and two isolated schemas. Run the
administrative statements as a GaussDB administrator and replace all example
names and passwords.

### 1. Create application users

```sql
CREATE USER ragflow_meta IDENTIFIED BY 'replace-with-a-strong-password';
CREATE USER ragflow_doc IDENTIFIED BY 'replace-with-a-different-password';
```

### 2. Create an Oracle-compatible database

Run only the statement that matches your deployment type:

```sql
-- Centralized GaussDB
CREATE DATABASE ragflow
  WITH DBCOMPATIBILITY = 'A'
       ENCODING = 'UTF8';

-- Distributed GaussDB
CREATE DATABASE ragflow
  WITH DBCOMPATIBILITY = 'ORA'
       ENCODING = 'UTF8';
```

Connect to the new `ragflow` database before continuing.

### 3. Create and assign schemas

```sql
GRANT CONNECT ON DATABASE ragflow TO ragflow_meta;
GRANT CONNECT ON DATABASE ragflow TO ragflow_doc;

CREATE SCHEMA ragflow_meta AUTHORIZATION ragflow_meta;
CREATE SCHEMA ragflow_doc AUTHORIZATION ragflow_doc;

GRANT USAGE, CREATE ON SCHEMA ragflow_meta TO ragflow_meta;
GRANT USAGE, CREATE ON SCHEMA ragflow_doc TO ragflow_doc;
```

RAGFlow creates and migrates its own tables and indexes. The application users
must own their schemas, or otherwise have enough privileges to create, alter,
read, write, and delete the objects in those schemas. Do not pre-create
RAGFlow application tables.

### 4. Verify each application connection

Connect once as `ragflow_meta` and once as `ragflow_doc`, then run the
appropriate schema check:

```sql
SHOW sql_compatibility;
SHOW server_encoding;
SHOW client_encoding;

SELECT has_schema_privilege(
  current_user,
  'ragflow_meta',
  'USAGE'
);
SELECT has_schema_privilege(
  current_user,
  'ragflow_meta',
  'CREATE'
);
```

Use `ragflow_doc` in the two privilege checks when connected as the DocEngine
user. Also verify `SHOW enable_vectordb` as the DocEngine user.

## Configure Docker Compose

Edit `docker/.env`. The following example uses the same GaussDB database with
separate schemas and users:

```dotenv
# Select GaussDB for both storage roles.
DB_TYPE=gaussdb
DOC_ENGINE=gaussdb

# Do not start the built-in MySQL metadata service.
METADATA_DB_PROFILE=gaussdb
COMPOSE_PROFILES=${DOC_ENGINE},${DEVICE},metadata-${METADATA_DB_PROFILE}

# Metadata database: used only when DB_TYPE=gaussdb.
GAUSSDB_METADATA_HOST=gaussdb.example.com
GAUSSDB_METADATA_PORT=8000
GAUSSDB_METADATA_DBNAME=ragflow
GAUSSDB_METADATA_USER=ragflow_meta
GAUSSDB_METADATA_PASSWORD=replace-with-a-strong-password
GAUSSDB_METADATA_SCHEMA=ragflow_meta
GAUSSDB_METADATA_MAX_CONNECTIONS=100
GAUSSDB_METADATA_STALE_TIMEOUT=30

# DocEngine and Memory message store: used only when DOC_ENGINE=gaussdb.
GAUSSDB_HOST=gaussdb.example.com
GAUSSDB_PORT=8000
GAUSSDB_DATABASE=ragflow
GAUSSDB_USER=ragflow_doc
GAUSSDB_PASSWORD=replace-with-a-different-password
GAUSSDB_SCHEMA=ragflow_doc
```

`GAUSSDB_METADATA_SCHEMA` and `GAUSSDB_SCHEMA` are schema names, not database
names.

The container entrypoint renders the DocEngine settings as this flat mapping
in `conf/service_conf.yaml`:

```yaml
gaussdb:
  host: 'gaussdb.example.com'
  port: 8000
  database: 'ragflow'
  user: 'ragflow_doc'
  password: 'replace-with-a-different-password'
  schema: 'ragflow_doc'
```

Do not nest these fields below a `config:` key.

Review the resolved Compose configuration without storing or sharing its
secret-bearing output:

```bash
cd docker
docker compose -f docker-compose.yml config
```

Start RAGFlow:

```bash
docker compose -f docker-compose.yml up -d
docker compose -f docker-compose.yml ps
```

## Configure a source-code deployment

For a deployment started directly from the source tree, set `DB_TYPE`,
`DOC_ENGINE`, and all `GAUSSDB_METADATA_*` values in `docker/.env` as shown in
the Docker Compose example. `docker/launch_backend_service.sh` loads that file
when it starts.

The source launcher does not render `docker/service_conf.yaml.template`.
Create `conf/local.service_conf.yaml` for the DocEngine:

```yaml
gaussdb:
  host: 'gaussdb.example.com'
  port: 8000
  database: 'ragflow'
  user: 'ragflow_doc'
  password: 'replace-with-a-different-password'
  schema: 'ragflow_doc'
```

Then start the backend using the normal source deployment command:

```bash
source .venv/bin/activate
export PYTHONPATH="$(pwd)"
bash docker/launch_backend_service.sh
```

`conf/local.service_conf.yaml` overrides the matching top-level block in
`conf/service_conf.yaml` and should not be committed with real credentials.

## Configure Helm

The Helm chart does not deploy GaussDB. Disable the chart's MySQL instance and
provide both GaussDB connection groups in an override file:

```yaml
# values.gaussdb.yaml
mysql:
  enabled: false

env:
  DB_TYPE: gaussdb
  DOC_ENGINE: gaussdb

  GAUSSDB_METADATA_HOST: gaussdb.example.com
  GAUSSDB_METADATA_PORT: "8000"
  GAUSSDB_METADATA_DBNAME: ragflow
  GAUSSDB_METADATA_USER: ragflow_meta
  GAUSSDB_METADATA_PASSWORD: replace-with-a-strong-password
  GAUSSDB_METADATA_SCHEMA: ragflow_meta
  GAUSSDB_METADATA_MAX_CONNECTIONS: "100"
  GAUSSDB_METADATA_STALE_TIMEOUT: "30"

  GAUSSDB_HOST: gaussdb.example.com
  GAUSSDB_PORT: "8000"
  GAUSSDB_DATABASE: ragflow
  GAUSSDB_USER: ragflow_doc
  GAUSSDB_PASSWORD: replace-with-a-different-password
  GAUSSDB_SCHEMA: ragflow_doc
```

Validate and apply:

```bash
helm lint ./helm
helm template ragflow ./helm -f values.gaussdb.yaml >/tmp/ragflow-rendered.yaml
helm upgrade --install ragflow ./helm \
  --namespace ragflow \
  --create-namespace \
  -f values.gaussdb.yaml
```

The rendered manifest contains Kubernetes Secret data. Protect and delete any
rendered file according to your security policy.

## Validate the running deployment

### Check startup logs

For Docker Compose:

```bash
cd docker
docker compose -f docker-compose.yml logs --tail=300 ragflow-cpu
# Use ragflow-gpu instead when DEVICE=gpu.
```

Confirm that:

- Metadata initialization completed without migration or permission errors.
- The log reports a centralized or distributed GaussDB metadata deployment.
- The GaussDB DocEngine connection initialized with the intended database and
  schema. Passwords must not appear in logs.

### Check the authenticated health endpoints

After signing in, call the system endpoints with a valid RAGFlow access token:

```bash
curl -sS \
  -H 'Authorization: Bearer <access-token>' \
  http://127.0.0.1:9380/api/v1/system/status

curl -sS \
  -H 'Authorization: Bearer <access-token>' \
  http://127.0.0.1:9380/api/v1/system/gaussdb/status
```

The general status must report a healthy metadata database and DocEngine. The
GaussDB-specific status must report:

- `status: alive`
- `sql_compatibility: A` or `ORA`
- `client_encoding: UTF8`
- The intended database and schema in the masked connection URI

### Check database objects

RAGFlow creates metadata tables during startup. DocEngine tables and indexes
are created lazily when datasets and vector indexes are created.

```sql
SELECT table_schema, count(*)
FROM information_schema.tables
WHERE table_schema IN ('ragflow_meta', 'ragflow_doc')
GROUP BY table_schema
ORDER BY table_schema;
```

Create a small dataset, upload and parse one document, and run a retrieval
test. Then confirm that DocEngine tables and `gsdiskann` indexes exist in
`ragflow_doc`.

## Optional metadata connection settings

The metadata pool supports:

| Variable | Default | Meaning |
| --- | --- | --- |
| `GAUSSDB_METADATA_MAX_CONNECTIONS` | `100` | Maximum Peewee pool connections |
| `GAUSSDB_METADATA_STALE_TIMEOUT` | `30` | Stale connection timeout in seconds |
| `GAUSSDB_METADATA_OPTIONS` | Generated automatically | Complete libpq options string |

Normally, leave `GAUSSDB_METADATA_OPTIONS` unset. RAGFlow then generates:

```text
-c search_path=<GAUSSDB_METADATA_SCHEMA>
-c client_encoding=UTF8
-c default_transaction_read_only=off
```

If `GAUSSDB_METADATA_OPTIONS` is set, it replaces the complete generated value;
it is not appended. Include all required options, especially `search_path`,
`client_encoding=UTF8`, and `default_transaction_read_only=off`.

## Troubleshooting

### RAGFlow still connects to MySQL

- Confirm that the application container has `DB_TYPE=gaussdb`.
- For Docker Compose, set `METADATA_DB_PROFILE=gaussdb` so the built-in MySQL
  service is not started.
- Recreate the RAGFlow containers after changing `docker/.env`.

### Metadata startup reports that `search_path` has no existing schema

- Create `GAUSSDB_METADATA_SCHEMA` before starting RAGFlow.
- Confirm that the metadata user has `USAGE` and `CREATE` on that schema.
- Use an unquoted schema identifier containing only letters, digits, and
  underscores, starting with a letter or underscore.

### DocEngine reports missing GaussDB fields

The DocEngine does not read `GAUSSDB_METADATA_*`. Set all required
`GAUSSDB_HOST`, `GAUSSDB_PORT`, `GAUSSDB_DATABASE`, `GAUSSDB_USER`, and
`GAUSSDB_PASSWORD` values. In a source deployment, also add the flat
`gaussdb:` block to `conf/local.service_conf.yaml`.

### Health reports unsupported compatibility

The configured database is not A/ORA-compatible. Create a new compatible
database and migrate or recreate the data; the compatibility mode cannot be
changed in place.

### Vector index creation fails with an unsupported-feature error

Verify `SHOW enable_vectordb` on a new DocEngine connection. In a distributed
deployment, confirm that the DBA enabled the parameter on all required CN and
DN nodes. RAGFlow does not silently fall back to an unindexed vector scan.

### Permission errors occur during table or index creation

Make the application user the schema owner, or grant the equivalent privileges
for all RAGFlow-managed objects. The DocEngine performs a startup check for
`USAGE` and `CREATE`, but later migrations also require ownership or suitable
`ALTER` and `DROP` privileges.