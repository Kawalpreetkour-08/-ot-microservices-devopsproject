# OT-Microservices DevOps

Hands-on DevOps practice on the [OpsTree OT-Microservices](https://github.com/OT-MICROSERVICES) application.
The application code belongs to OpsTree. This repo documents **my** setup, steps and fixes.

# 01 - Employee API: Local Setup

Running the [OT-Microservices `employee-api`](https://github.com/OT-MICROSERVICES/employee-api) on my laptop, with ScyllaDB running in Docker.


## Architecture

```
curl / Swagger UI
      |
      v
employee-api  (Go binary on the laptop, port 8080)
      |  config.yaml -> 127.0.0.1:9042
      v
laptop port 9042  <-- docker -p 9042:9042 -->  container port 9042
                                                    |
                                                    v
                                         ScyllaDB (container: employee-scylla)
                                         keyspace: employee_db
                                         table:    employee_info
```

## Tools used

| Tool | Purpose |
|---|---|
| Docker | Runs ScyllaDB as a container |
| ScyllaDB | Database used by employee-api |
| Go 1.27 | Builds the application |
| golang-migrate 4.17.1 | Creates the table from `migration/*.sql` |
| jq | Reads the DB connection string from `migration.json` |

## Setup steps

### 1. Start ScyllaDB in Docker
The container is limited to 1 CPU and 1 GB RAM so it runs on a laptop.
```bash
docker run -d --name employee-scylla -p 9042:9042 scylladb/scylla:latest \
  --smp 1 --memory 1G --overprovisioned 1 --developer-mode 1
docker exec -it employee-scylla nodetool status
```
Expected: a line starting with `UN` (Up / Normal).

### 2. Create the keyspace
The migration only contains `CREATE TABLE`, so the keyspace has to exist first.
```bash
docker exec -it employee-scylla cqlsh -e "CREATE KEYSPACE IF NOT EXISTS employee_db \
  WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};"
```
`SimpleStrategy` and replication factor 1 are fine for a single local node. Production would use `NetworkTopologyStrategy` with RF 3.

### 3. Point the config to localhost
`config.yaml` and `migration.json` contained a hardcoded container IP (`172.17.0.3`). Container IPs change, so I used the mapped port instead.
```bash
sed -i 's/172.17.0.3/127.0.0.1/' config.yaml migration.json
```

### 4. Run the migration
This is the same command `make run-migrations` runs.
```bash
migrate -source file://migration -database "$(jq -r '.database' migration.json)" up
docker exec -it employee-scylla cqlsh -e "DESCRIBE TABLES;" -k employee_db
```
Expected: `employee_info` (and `schema_migrations`, which `migrate` uses to track versions).

### 5. Build and run the API
```bash
make build
export GIN_MODE=release
./employee-api
```

## Verification

### Database
Container is running:

![docker ps](screenshots/01-docker-ps.png)

Node is Up/Normal:

![nodetool status](screenshots/02-nodetool-status.png)

Table created by the migration:

![cqlsh tables](screenshots/03-cqlsh-tables.png)

### API (Swagger UI: http://localhost:8080/swagger/index.html)
Health detail shows the API is connected to ScyllaDB. `redis: down` is expected because Redis is disabled in `config.yaml`.

![health detail](screenshots/04-swagger-health-detail.png)

Created a test employee:

![create](screenshots/05-swagger-create.png)

Read it back through the API:

![search all](screenshots/06-swagger-search-all.png)

### Data in the database
The same record read directly from ScyllaDB, without going through the API:

![cqlsh select](screenshots/07-cqlsh-select.png)

All data shown is test data.

## Issues found and fixed

| Problem | Cause | Fix |
|---|---|---|
| API could not reach the database | Hardcoded container IP in config files | Used `127.0.0.1` with the mapped port |
| Migration did not create the keyspace | Migration only has `CREATE TABLE` | Created the keyspace manually |
| `POST /create` returned 400 | `joining_date` sent as `01-11-2026` | ScyllaDB `date` needs `YYYY-MM-DD` |
| `redis: down` in health detail | Redis is disabled in `config.yaml` | Expected, Redis is not running yet |

## Observations for the next phase
- The `down.sql` migration contains `CREATE TABLE` instead of `DROP TABLE`, so `migrate down` would not undo anything.
- In the Dockerfile, the final stage copies the `employee-api` binary from the local folder, so `make build` must run before `docker build`.
- Database data lives inside the container. Removing the container deletes it, so a volume is needed.

## Next
Phase 02: Docker and docker-compose.

## Credits
Application: [OpsTree OT-Microservices](https://github.com/OT-MICROSERVICES)
