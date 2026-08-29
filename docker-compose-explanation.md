# Kestra Docker Compose Explanation

This document provides a complete copy of the `docker-compose.yml` file used to run Kestra locally in this module, followed by a detailed breakdown of its services, configuration parameters, volumes, and secret ingestion.

---

## The `docker-compose.yml` File

```yaml
volumes:
  kestra_postgres_data:
    driver: local
  kestra_data:
    driver: local
  kestra_tmp:
    driver: local

services:
  kestra_postgres:
    image: postgres:18
    volumes:
      - kestra_postgres_data:/var/lib/postgresql
    environment:
      POSTGRES_DB: kestra
      POSTGRES_USER: kestra
      POSTGRES_PASSWORD: k3str4
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      interval: 30s
      timeout: 10s
      retries: 10

  kestra:
    image: kestra/kestra:v1.3.21
    pull_policy: always
    user: "root"
    command: server standalone
    volumes:
      - kestra_data:/app/storage
      - /var/run/docker.sock:/var/run/docker.sock
      - /tmp/kestra-wd:/tmp/kestra-wd
    environment:
      SECRET_GEMINI_API_KEY: ${SECRET_GEMINI_API_KEY}
      SECRET_TAVILY_API_KEY: ${SECRET_TAVILY_API_KEY}
      # SECRET_OPENAI_API_KEY: ${SECRET_OPENAI_API_KEY}
      KESTRA_CONFIGURATION: |
        datasources:
          postgres:
            url: jdbc:postgresql://kestra_postgres:5432/kestra
            driverClassName: org.postgresql.Driver
            username: kestra
            password: k3str4
        kestra:
          server:
            basicAuth:
              username: "admin@kestra.io"
              password: Admin1234!
          repository:
            type: postgres
          storage:
            type: local
            local:
              basePath: "/app/storage"
          queue:
            type: postgres
          tasks:
            tmpDir:
              path: /tmp/kestra-wd/tmp
          url: http://localhost:8080/
          ai:
            type: gemini
            gemini:
              model-name: gemini-3.5-flash
              api-key: ${GEMINI_API_KEY}
    ports:
      - "8080:8080"
      - "8081:8081"
    depends_on:
      kestra_postgres:
        condition: service_started
```

---

## Detailed Breakdown

### 1. Persistent Volumes Block
```yaml
volumes:
  kestra_postgres_data:
    driver: local
  kestra_data:
    driver: local
  kestra_tmp:
    driver: local
```
Docker Compose uses these named volumes to persist your database state, flows, and working files on your host machine even if you stop, delete, or recreate the containers (e.g., using `docker compose down` and `docker compose up`).
*   **`kestra_postgres_data`**: Stores Postgres' internal database state, which keeps your flow definitions, execution history, and logs persistent.
*   **`kestra_data`**: Stores files uploaded/created during task runs and internal cached inputs/outputs.
*   **`kestra_tmp`**: Used for temporary execution tasks.

---

### 2. Database Service (`kestra_postgres`)
```yaml
  kestra_postgres:
    image: postgres:18
    volumes:
      - kestra_postgres_data:/var/lib/postgresql
    environment:
      POSTGRES_DB: kestra
      POSTGRES_USER: kestra
      POSTGRES_PASSWORD: k3str4
```
This is a standard PostgreSQL database (version 18) dedicated to Kestra.
*   **No Port Mapping**: Notice that it does not map port `5432` to the host. This ensures database connections are only allowed from other containers on the same Docker network (like Kestra itself) for security.
*   **Volumes**: Mounts `/var/lib/postgresql` (where Postgres stores database files) to the persistent volume `kestra_postgres_data`.

#### Database Healthcheck
```yaml
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      interval: 30s
      timeout: 10s
      retries: 10
```
This runs `pg_isready` inside the Postgres container every 30 seconds to ensure the database server is ready and accepting queries before Kestra attempts to boot up.

---

### 3. Kestra Service (`kestra`)
```yaml
  kestra:
    image: kestra/kestra:v1.3.21
    pull_policy: always
    user: "root"
    command: server standalone
```
*   **`image: kestra/kestra:v1.3.21`**: Runs the Kestra orchestrator server container.
*   **`pull_policy: always`**: Ensures Docker checks and pulls the latest image tag version on every startup.
*   **`user: "root"`**: Runs Kestra as the root user. This is necessary so Kestra can interact with the host's Docker socket daemon.
*   **`command: server standalone`**: Starts Kestra in standalone mode, which runs all core services (Web UI, Executor, Queue, Scheduler, Worker) in a single process.

#### Volumes and Bind Mounts
```yaml
    volumes:
      - kestra_data:/app/storage
      - /var/run/docker.sock:/var/run/docker.sock
      - /tmp/kestra-wd:/tmp/kestra-wd
```
*   **`/var/run/docker.sock:/var/run/docker.sock`**: Mounts your host machine's Docker daemon socket inside the Kestra container. This is extremely important because it allows Kestra to spin up containerized tools or Python sub-containers on your local machine (such as executing flows with `DockerMcpClient`).
*   **`/tmp/kestra-wd:/tmp/kestra-wd`**: Mounts a working directory directory from the host to keep task execution runtimes consistent.

#### Environment Variables & Credentials Injection
```yaml
    environment:
      SECRET_GEMINI_API_KEY: ${SECRET_GEMINI_API_KEY}
      SECRET_TAVILY_API_KEY: ${SECRET_TAVILY_API_KEY}
```
Injects base64-encoded environment variables from your host terminal. Kestra automatically treats variables prefixed with `SECRET_` as secure. Inside flow definitions, they can be securely referenced using `{{ secret('GEMINI_API_KEY') }}` and `{{ secret('TAVILY_API_KEY') }}` (dropping the `SECRET_` prefix).

#### Embedded Kestra Configuration (`KESTRA_CONFIGURATION`)
*   **`datasources.postgres`**: Configures the JDBC connection pool settings to connect to the database service named `kestra_postgres`.
*   **`kestra.server.basicAuth`**: Configures UI credentials (`admin@kestra.io` / `Admin1234!`) to secure your Kestra console page.
*   **`kestra.repository.type` & `kestra.queue.type`**: Directs Kestra to store workflow schedules, logs, and queue tasks inside Postgres.
*   **`kestra.storage`**: Configures local workspace directories mapped to `/app/storage` (stored on the persistent volume).
*   **`kestra.ai`**: Enables Kestra's built-in **AI Copilot** features. Using `gemini-3.5-flash` and the provided `GEMINI_API_KEY`, it gives you AI-assisted flow generation directly inside the Web UI's flow editor.

#### Port Bindings
```yaml
    ports:
      - "8080:8080"
      - "8081:8081"
```
*   `8080:8080`: Exposes the Kestra Web UI interface at `http://localhost:8080`.
*   `8081:8081`: Exposes Kestra's internal management, health-checks, and telemetry endpoints.

#### Startup Sequence (`depends_on`)
```yaml
    depends_on:
      kestra_postgres:
        condition: service_started
```
Tells Docker Compose not to start the main Kestra process until the PostgreSQL server is up.
