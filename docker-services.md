Below is a complete, single `README.md` that keeps **all secrets and dynamic configuration strictly in `.env.example` files**, uses `env_file` in every compose file, and separates each service while keeping them easily composable via a shared external network.

---

# 🧰 Modular Infrastructure Stack

This repository provides fully separated, production-ready Docker Compose templates for:

- PostgreSQL
- MySQL
- Redis
- Redis Stack
- Kafka
- MongoDB + Mongo Express
- AWS S3 Backup (optional, add-on)

Each service runs independently but shares a common external Docker network, making it trivial to run them separately or combine them into a single stack.

---

## 🧭 Index

- [1. Project Layout](#1-project-layout)
- [2. Shared Network Setup](#2-shared-network-setup)
- [3. PostgreSQL](#3-postgresql)
- [4. MySQL](#4-mysql)
- [5. Redis](#5-redis)
- [6. Redis Stack](#6-redis-stack)
- [7. Kafka](#7-kafka)
- [8. MongoDB + Mongo Express](#8-mongodb--mongo-express)
- [9. AWS S3 Backup Service](#9-aws-s3-backup-service)
- [10. Usage Guide](#10-usage-guide)
  - [Run Services Separately](#run-services-separately)
  - [Run Services Together](#run-services-together)
  - [Execute Backups](#execute-backups)

---

## 1. Project Layout

```text
infra/
├── README.md
│
├── postgres/
│   ├── docker-compose.yml
│   └── .env.example
│
├── mysql/
│   ├── docker-compose.yml
│   └── .env.example
│
├── redis/
│   ├── docker-compose.yml
│   └── .env.example
│
├── redis-stack/
│   ├── docker-compose.yml
│   └── .env.example
│
├── kafka/
│   ├── docker-compose.yml
│   └── .env.example
│
├── mongodb/
│   ├── docker-compose.yml
│   └── .env.example
│
└── backup/
    ├── docker-compose.yml
    ├── Dockerfile
    ├── backup.sh
    └── .env.example
```

---

## 2. Shared Network Setup

All services communicate over a single external Docker network.

Create it once:

```bash
docker network create shared-network-name
```

Every compose file below references this network using:

```yaml
networks:
  infra:
    external: true
    name: shared-network-name
```

Service network aliases match their folder names for seamless cross-service DNS resolution.

---

## 3. PostgreSQL

### `postgres/docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    container_name: infra-postgres
    env_file:
      - .env
    ports:
      - '${POSTGRES_PORT_HOST}:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB']
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s
    networks:
      infra:
        aliases:
          - postgres

networks:
  infra:
    external: true
    name: shared-network-name

volumes:
  postgres_data:
    name: infra_postgres_data
```

### `postgres/.env.example`

```env
POSTGRES_PORT_HOST=5432

POSTGRES_USER=app_user
POSTGRES_PASSWORD=change_me_postgres
POSTGRES_DB=app_db

# Connection strings
POSTGRES_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@localhost:${POSTGRES_PORT_HOST}/${POSTGRES_DB}
POSTGRES_DOCKER_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
```

---

## 4. MySQL

### `mysql/docker-compose.yml`

```yaml
services:
  mysql:
    image: mysql:8.0
    restart: unless-stopped
    container_name: infra-mysql
    env_file:
      - .env
    ports:
      - '${MYSQL_PORT_HOST}:3306'
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ['CMD-SHELL', 'mysqladmin ping -h localhost -uroot -p"$$MYSQL_ROOT_PASSWORD" --silent']
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 20s
    networks:
      infra:
        aliases:
          - mysql

networks:
  infra:
    external: true
    name: shared-network-name

volumes:
  mysql_data:
    name: infra_mysql_data
```

### `mysql/.env.example`

```env
MYSQL_PORT_HOST=3306

MYSQL_ROOT_PASSWORD=change_me_root
MYSQL_DATABASE=app_db
MYSQL_USER=app_user
MYSQL_PASSWORD=change_me_mysql

# Connection strings
MYSQL_URL=mysql://${MYSQL_USER}:${MYSQL_PASSWORD}@localhost:${MYSQL_PORT_HOST}/${MYSQL_DATABASE}
MYSQL_DOCKER_URL=mysql://${MYSQL_USER}:${MYSQL_PASSWORD}@mysql:3306/${MYSQL_DATABASE}
```

---

## 5. Redis

### `redis/docker-compose.yml`

```yaml
services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    container_name: infra-redis
    command: ['redis-server', '--appendonly', 'yes']
    env_file:
      - .env
    ports:
      - '${REDIS_PORT_HOST}:6379'
    volumes:
      - redis_data:/data
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 5s
    networks:
      infra:
        aliases:
          - redis

networks:
  infra:
    external: true
    name: shared-network-name

volumes:
  redis_data:
    name: infra_redis_data
```

### `redis/.env.example`

```env
REDIS_PORT_HOST=6379

REDIS_URL=redis://localhost:${REDIS_PORT_HOST}
REDIS_DOCKER_URL=redis://redis:6379
```

---

## 6. Redis Stack

### `redis-stack/docker-compose.yml`

```yaml
services:
  redis-stack:
    image: redis/redis-stack:latest
    restart: unless-stopped
    container_name: infra-redis-stack
    env_file:
      - .env
    ports:
      - '${REDIS_STACK_PORT_HOST}:6379'
      - '${REDIS_STACK_UI_PORT_HOST}:8001'
    volumes:
      - redis_stack_data:/data
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 5s
    networks:
      infra:
        aliases:
          - redis-stack

networks:
  infra:
    external: true
    name: shared-network-name

volumes:
  redis_stack_data:
    name: infra_redis_stack_data
```

### `redis-stack/.env.example`

```env
REDIS_STACK_PORT_HOST=6380
REDIS_STACK_UI_PORT_HOST=8001

REDIS_STACK_URL=redis://localhost:${REDIS_STACK_PORT_HOST}
REDIS_STACK_UI_URL=http://localhost:${REDIS_STACK_UI_PORT_HOST}
REDIS_STACK_DOCKER_URL=redis://redis-stack:6379
```

---

## 7. Kafka

### `kafka/docker-compose.yml`

```yaml
services:
  kafka:
    image: bitnami/kafka:3.7
    restart: unless-stopped
    container_name: infra-kafka
    env_file:
      - .env
    ports:
      - '${KAFKA_PORT_HOST}:9092'
    volumes:
      - kafka_data:/bitnami/kafka
    healthcheck:
      test: ['CMD-SHELL', '/opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server localhost:19092 --list']
      interval: 20s
      timeout: 10s
      retries: 10
      start_period: 30s
    networks:
      infra:
        aliases:
          - kafka

networks:
  infra:
    external: true
    name: shared-network-name

volumes:
  kafka_data:
    name: infra_kafka_data
```

### `kafka/.env.example`

```env
KAFKA_PORT_HOST=9092

KAFKA_CFG_NODE_ID=0
KAFKA_CFG_PROCESS_ROLES=controller,broker
KAFKA_CFG_LISTENERS=PLAINTEXT://:19092,EXTERNAL://:9092,CONTROLLER://:9093
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:19092,EXTERNAL://localhost:9092
KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,EXTERNAL:PLAINTEXT
KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
KAFKA_CFG_INTER_BROKER_LISTENER_NAME=PLAINTEXT
KAFKA_CFG_OFFSETS_TOPIC_REPLICATION_FACTOR=1
KAFKA_CFG_TRANSACTION_STATE_LOG_REPLICATION_FACTOR=1
KAFKA_CFG_TRANSACTION_STATE_LOG_MIN_ISR=1
KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=true
ALLOW_PLAINTEXT_LISTENER=yes

# Connection helpers
KAFKA_URL=localhost:${KAFKA_PORT_HOST}
KAFKA_DOCKER_URL=kafka:19092
```

---

## 8. MongoDB + Mongo Express

### `mongodb/docker-compose.yml`

```yaml
services:
  mongodb:
    image: mongodb/mongodb-community-server:8.0-ubi8
    restart: unless-stopped
    container_name: infra-mongodb
    env_file:
      - .env
    ports:
      - '${MONGODB_PORT_HOST}:27017'
    volumes:
      - mongodb_data:/data/db
    networks:
      infra:
        aliases:
          - mongodb

  mongo-express:
    image: mongo-express:latest
    restart: unless-stopped
    container_name: infra-mongo-express
    env_file:
      - .env
    ports:
      - '${MONGO_EXPRESS_PORT_HOST}:8081'
    depends_on:
      - mongodb
    networks:
      infra:
        aliases:
          - mongo-express

networks:
  infra:
    external: true
    name: shared-network-name

volumes:
  mongodb_data:
    name: infra_mongodb_data
```

### `mongodb/.env.example`

```env
MONGODB_PORT_HOST=27017
MONGO_EXPRESS_PORT_HOST=8081

MONGODB_INITDB_ROOT_USERNAME=app_user
MONGODB_INITDB_ROOT_PASSWORD=change_me_mongo

ME_CONFIG_MONGODB_SERVER=mongodb
ME_CONFIG_MONGODB_PORT=27017
ME_CONFIG_MONGODB_ADMINUSERNAME=${MONGODB_INITDB_ROOT_USERNAME}
ME_CONFIG_MONGODB_ADMINPASSWORD=${MONGODB_INITDB_ROOT_PASSWORD}
ME_CONFIG_BASICAUTH_USERNAME=admin
ME_CONFIG_BASICAUTH_PASSWORD=admin

# Connection helpers
MONGODB_URL=mongodb://${MONGODB_INITDB_ROOT_USERNAME}:${MONGODB_INITDB_ROOT_PASSWORD}@localhost:${MONGODB_PORT_HOST}/?authSource=admin
MONGODB_DOCKER_URL=mongodb://${MONGODB_INITDB_ROOT_USERNAME}:${MONGODB_INITDB_ROOT_PASSWORD}@mongodb:27017/?authSource=admin
MONGO_EXPRESS_URL=http://localhost:${MONGO_EXPRESS_PORT_HOST}
```

---

## 9. AWS S3 Backup Service

The backup service is completely isolated. It connects to other services via their Docker URLs and uploads dumps to S3.

### `backup/docker-compose.yml`

```yaml
services:
  backup:
    build:
      context: .
    restart: unless-stopped
    container_name: infra-backup
    env_file:
      - .env
    volumes:
      - ./backups:/backups
    command: >
      bash -c '
      while true; do
        /opt/backup/backup.sh || true;
        sleep "$${BACKUP_INTERVAL_SECONDS:-86400}";
      done'
    networks:
      infra:
        aliases:
          - backup

networks:
  infra:
    external: true
    name: shared-network-name
```

### `backup/Dockerfile`

```dockerfile
FROM python:3.12-slim

RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    bash postgresql-client default-mysql-client redis-tools ca-certificates \
  && pip install --no-cache-dir awscli \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /opt/backup
COPY backup.sh /opt/backup/backup.sh
RUN chmod +x /opt/backup/backup.sh

CMD ["/opt/backup/backup.sh"]
```

### `backup/backup.sh`

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

timestamp="$(date +%Y%m%d-%H%M%S)"
backup_dir="/backups/${timestamp}"
mkdir -p "${backup_dir}"

log() { echo "[backup] $*"; }
log "Starting backup: ${timestamp}"

# PostgreSQL
if [ -n "${POSTGRES_URL:-}" ]; then
  log "Dumping PostgreSQL..."
  pg_dump "${POSTGRES_URL}" -F c -f "${backup_dir}/postgres.dump" || log "WARNING: pg_dump failed"
fi

# MySQL
if [ -n "${MYSQL_URL:-}" ]; then
  log "Dumping MySQL..."
  mysql_url="${MYSQL_URL#mysql://}"
  creds="${mysql_url%%@*}"
  host_path="${mysql_url#*@}"
  db="${host_path##*/}"
  user="${creds%%:*}"
  pass="${creds#*:}"
  host_port="${host_path%%/*}"
  export MYSQL_PWD="${pass}"
  mysqldump -h "${host_port%%:*}" -P "${host_port##*:}" -u "${user}" \
    --single-transaction --routines --triggers --databases "${db}" \
    > "${backup_dir}/mysql_${db}.sql" || log "WARNING: mysqldump failed"
  unset MYSQL_PWD
fi

# Redis
if [ -n "${REDIS_URL:-}" ]; then
  log "Dumping Redis..."
  redis-cli -u "${REDIS_URL}" BGSAVE
  sleep "${REDIS_BGSAVE_WAIT_SECONDS:-5}"
  redis-cli -u "${REDIS_URL}" --rdb "${backup_dir}/redis_dump.rdb" || log "WARNING: redis-cli --rdb failed"
fi

# MongoDB
if [ -n "${MONGODB_URL:-}" ] && command -v mongodump &>/dev/null; then
  log "Dumping MongoDB..."
  mongodump --uri="${MONGODB_URL}" --archive="${backup_dir}/mongodb.archive" --gzip || log "WARNING: mongodump failed"
fi

# S3 Upload
if [ -n "${S3_URL:-}" ]; then
  s3_target="${S3_URL%/}"
  log "Uploading to ${s3_target}/${timestamp}/"
  aws s3 cp "${backup_dir}" "${s3_target}/${timestamp}/" --recursive --only-show-errors \
    ${S3_ENDPOINT_URL:+--endpoint-url "${S3_ENDPOINT_URL}"}
fi

# Local Retention
retention="${BACKUP_RETENTION_DAYS:-7}"
log "Cleaning backups older than ${retention} days"
find /backups -maxdepth 1 -mindepth 1 -type d -mtime +"${retention}" -exec rm -rf {} +

log "Backup completed."
```

### `backup/.env.example`

```env
BACKUP_INTERVAL_SECONDS=86400
BACKUP_RETENTION_DAYS=7
REDIS_BGSAVE_WAIT_SECONDS=5

# AWS S3
S3_URL=s3://your-backup-bucket/infra-backups
AWS_DEFAULT_REGION=us-east-1
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
# Optional: S3_ENDPOINT_URL=https://s3.example.com

# Service URLs (use DOCKER_URLs from respective services)
POSTGRES_URL=postgresql://app_user:change_me_postgres@postgres:5432/app_db
MYSQL_URL=mysql://app_user:change_me_mysql@mysql:3306/app_db
REDIS_URL=redis://redis:6379
MONGODB_URL=mongodb://app_user:change_me_mongo@mongodb:27017/?authSource=admin
```

---

## 10. Usage Guide

### Run Services Separately

First create the shared network:

```bash
docker network create shared-network-name
```

Start any service independently:

```bash
# PostgreSQL only
docker compose -f postgres/docker-compose.yml up -d

# MySQL only
docker compose -f mysql/docker-compose.yml up -d

# Redis only
docker compose -f redis/docker-compose.yml up -d

# MongoDB + Mongo Express only
docker compose -f mongodb/docker-compose.yml up -d
```

### Run Services Together

Because all compose files share the same external network and use consistent service aliases, you can combine them seamlessly:

```bash
docker compose \
  -f postgres/docker-compose.yml \
  -f mysql/docker-compose.yml \
  -f redis/docker-compose.yml \
  -f kafka/docker-compose.yml \
  -f mongodb/docker-compose.yml \
  up -d
```

To replace Redis with Redis Stack in the combined stack:

```bash
docker compose \
  -f postgres/docker-compose.yml \
  -f mysql/docker-compose.yml \
  -f redis-stack/docker-compose.yml \
  -f kafka/docker-compose.yml \
  -f mongodb/docker-compose.yml \
  up -d
```

### Execute Backups

1. Copy and configure the backup environment:
   ```bash
   cp backup/.env.example backup/.env
   # Edit backup/.env and match the *_URL variables to your running services
   ```

2. Build the backup image:
   ```bash
   docker compose -f backup/docker-compose.yml build
   ```

3. Run a one-time backup:
   ```bash
   docker compose -f backup/docker-compose.yml run --rm backup /opt/backup/backup.sh
   ```

4. Run scheduled backups (runs every `BACKUP_INTERVAL_SECONDS`):
   ```bash
   docker compose -f backup/docker-compose.yml up -d
   ```

5. View backup logs:
   ```bash
   docker compose -f backup/docker-compose.yml logs -f
   ```

---

### 🔑 Notes

- **Credentials are never hardcoded** in `docker-compose.yml`. They live exclusively in `.env` / `.env.example`.
- **Network aliases** (`postgres`, `mysql`, `redis`, `kafka`, `mongodb`) allow cross-service communication without port mapping.
- **Mongo Express** is bundled with MongoDB for convenience but can be separated by moving its service block to its own compose file if needed.
- **Kafka raw volume backups are intentionally skipped** in the backup script. Use MirrorMaker 2 or managed Kafka replication for production event stream durability.
- Always rename `.env.example` to `.env` in each folder and fill in values before running.