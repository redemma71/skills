# Docker Compose Debugging (cdc_app)

## Purpose

Diagnose and resolve Docker Compose issues in the cdc_app platform.

The platform consists of:

* FastAPI BFF
* Spring Boot backend
* PostgreSQL
* Redis
* Frontend (React/Vite)

Common failure modes include:

* Container startup failures
* Authentication failures
* Network connectivity issues
* Environment variable issues
* Volume initialization issues
* Health check failures
* Inter-service communication failures

---

## Required Diagnostic Workflow

Always follow these steps in order.

Do not skip steps.

Do not recommend deleting volumes until root cause has been identified.

### Step 1: Gather State

Collect:

```bash
docker compose ps

docker compose logs --tail=200

docker compose config

docker network ls

docker volume ls
```

Record all failing containers.

---

### Step 2: Classify Failure

Determine which category applies.

#### Startup Failure

Examples:

* Container exits immediately
* CrashLoopBackoff
* Restart loop

Indicators:

* exit code != 0
* stack trace in logs

#### Authentication Failure

Examples:

* password authentication failed
* login rejected
* access denied

Indicators:

* PostgreSQL authentication errors
* Spring Security failures
* Redis AUTH failures

#### Network Failure

Examples:

* connection refused
* timeout
* host not found

Indicators:

* unable to resolve hostname
* service unavailable

#### Configuration Failure

Examples:

* variable not set
* missing secret
* invalid URL

Indicators:

* WARN variable not set
* malformed JDBC URL
* invalid environment configuration

---

### Step 3: Validate Environment Variables

Inspect effective configuration:

```bash
docker compose config
```

Verify:

* POSTGRES_USER
* POSTGRES_PASSWORD
* POSTGRES_DB
* SPRING_DATASOURCE_URL
* SPRING_DATASOURCE_USERNAME
* SPRING_DATASOURCE_PASSWORD

Check for:

* blank values
* typo mismatches
* missing variables

Example:

Bad:

```yaml
POSTGRES_USER: ${DB_USERNAME}
```

.env:

```text
POSTGRES_USERNAME=cdc_user
```

Result:

Variable resolves to empty string.

---

### Step 4: Validate Container Networking

Inspect:

```bash
docker network inspect shared-network
```

Verify:

* expected containers attached
* unexpected containers attached

Look specifically for:

* Airflow
* legacy projects
* abandoned containers

Known cdc_app issue:

An Airflow container attached to shared-network repeatedly attempted PostgreSQL logins and generated misleading authentication errors.

---

### Step 5: Validate PostgreSQL

Inspect:

```bash
docker exec -it postgres psql -U $POSTGRES_USER
```

Verify:

```sql
SELECT current_user;

SELECT datname
FROM pg_database;

SELECT usename
FROM pg_stat_activity;
```

Check:

* expected database exists
* expected role exists
* unexpected clients

---

### Step 6: Validate Volumes

Inspect:

```bash
docker volume ls
```

Determine:

* new database initialization
* existing database reuse

Never recommend:

```bash
docker compose down -v
```

until it is confirmed that:

* data is disposable
* stale volume is root cause

---

### Step 7: Validate Service Discovery

From backend:

```bash
docker exec -it backend sh
```

Test:

```bash
ping postgres

ping redis
```

and:

```bash
nc -vz postgres 5432

nc -vz redis 6379
```

Expected:

Successful connection.

---

### Step 8: Validate Health Checks

Inspect:

```bash
docker inspect postgres

docker inspect backend
```

Review:

Health.Status

Expected:

healthy

Not acceptable:

starting indefinitely

unhealthy

---

## Common Root Causes

### Environment Variable Mismatch

Symptoms:

```text
variable not set
```

Cause:

.env key names differ from compose references.

---

### Shared Network Pollution

Symptoms:

```text
password authentication failed
```

even when application credentials are correct.

Cause:

Another container is attempting login.

Action:

Inspect shared-network.

---

### Stale PostgreSQL Volume

Symptoms:

```text
Role does not exist
```

or

```text
Database already initialized
```

Cause:

Database initialized with old credentials.

---

### Incorrect Service Name

Symptoms:

```text
Connection refused
```

or

```text
Unknown host
```

Cause:

Using localhost inside containers.

Correct:

```text
postgres
```

Incorrect:

```text
localhost
```

---

## Reporting Format

Always provide:

### Root Cause

One sentence.

### Evidence

Relevant log entries.

### Fix

Exact commands or configuration changes.

### Validation

Commands demonstrating successful remediation.

Example:

Root Cause:
Airflow container attached to shared-network was attempting PostgreSQL authentication using invalid credentials.

Evidence:
PostgreSQL logs showed repeated login attempts from Airflow container IP.

Fix:
Disconnect Airflow from shared-network.

Validation:
Authentication errors cease and backend connects successfully.

