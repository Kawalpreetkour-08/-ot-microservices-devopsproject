# 01 - Employee API: Local Setup

Running the [OT-Microservices employee-api](https://github.com/OT-MICROSERVICES/employee-api) locally with ScyllaDB in Docker.

> The application code belongs to OpsTree. This repo documents my DevOps setup and steps.

## Stack
- employee-api: Go (Gin)
- Database: ScyllaDB (Docker container)
- Tools: Docker, golang-migrate, jq, Go 1.27

## Steps

### 1. Start ScyllaDB in Docker
```bash
docker run -d --name employee-scylla -p 9042:9042 scylladb/scylla:latest \
  --smp 1 --memory 1G --overprovisioned 1 --developer-mode 1
docker exec -it employee-scylla nodetool status   # expect: UN
```

### 2. Create keyspace
```bash
docker exec -it employee-scylla cqlsh -e "CREATE KEYSPACE IF NOT EXISTS employee_db WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};"
```

### 3. Point config to localhost
The repo's `config.yaml` and `migration.json` had a hardcoded Docker IP (172.17.0.3).
The container port is mapped to the laptop (`-p 9042:9042`), so I replaced it with `127.0.0.1`.
```bash
sed -i 's/172.17.0.3/127.0.0.1/' config.yaml migration.json
```

### 4. Run the migration
```bash
migrate -source file://migration -database "$(jq -r '.database' migration.json)" up
docker exec -it employee-scylla cqlsh -e "DESCRIBE TABLES;" -k employee_db
```

### 5. Build and run the app
```bash
make build
export GIN_MODE=release
./employee-api
```

### 6. Verify
```bash
curl localhost:8080/api/v1/employee/health
curl localhost:8080/api/v1/employee/health/detail
```
Swagger UI: http://localhost:8080/swagger/index.html

## Issues found and fixed
| Problem | Cause | Fix |
|---|---|---|
| App could not reach the database | Hardcoded IP in config files | Used `127.0.0.1` with the mapped port |
| Migration did not create the keyspace | Migration only has `CREATE TABLE` | Created the keyspace manually |
| POST /create returned 400 | `joining_date` sent as `01-11-2026` | ScyllaDB `date` needs `YYYY-MM-DD` |
| `redis: down` in health detail | Redis disabled in `config.yaml` | Expected, Redis is not running yet |

## Credits
Application: [OpsTree OT-Microservices](https://github.com/OT-MICROSERVICES)
