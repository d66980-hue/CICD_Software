# spring-bdd

BDD-tested Spring Boot REST API using Cucumber and REST Assured, backed by MySQL/PostgreSQL in Docker and H2 for local tests.

Based on https://github.com/krushnaDash/spring-bdd

---

## Project Structure

```
spring-bdd/
├── src/
│   ├── main/
│   │   ├── java/com/test/springbdd/   # Controllers, entities, application
│   │   └── resources/
│   │       └── application.properties # Env-var driven config (H2 default)
│   └── test/
│       ├── java/com/test/springbdd/   # Cucumber step definitions, runner
│       └── resources/
│           ├── application-test.properties  # H2 in-memory config for tests
│           └── *.feature                    # Cucumber feature files
├── .github/workflows/
│   └── cicd.yml                       # GitHub Actions CI/CD pipeline
├── Dockerfile                         # Single-stage JRE image
├── docker-compose.yml                 # App + MySQL stack for local Docker use
└── pom.xml
```

---

## CI/CD Pipeline

The pipeline is defined in `.github/workflows/cicd.yml` and runs automatically on every push to `master`. It has three sequential jobs — each job only starts if the previous one succeeds.

```
Push to master
    │
    ▼
┌─────────────────┐
│  build-and-test │  Compile, run Cucumber/REST Assured tests against PostgreSQL
└────────┬────────┘
         │ success
         ▼
┌─────────────────┐
│     docker      │  Build JAR → build Docker image → push to Docker Hub
└────────┬────────┘
         │ success
         ▼
┌─────────────────┐
│     deploy      │  SSH into deves.xdi.uevora.pt, pull image, restart container
└─────────────────┘
```

### Stage 1 — Build and Test

- Spins up a `postgres:15` service container using credentials from GitHub Secrets.
- Checks out the code and sets up JDK 17 (Eclipse Temurin).
- Runs `mvn clean verify`, which compiles the project and executes all Cucumber BDD tests.
- Spring Boot connects to the ephemeral PostgreSQL container via environment variables injected into Maven — no production database is touched during CI.
- Surefire test reports are uploaded as a workflow artifact (available even on failure).

### Stage 2 — Docker Build and Push

- Builds the production JAR with `mvn clean package -DskipTests` (tests already ran in Stage 1).
- Builds the Docker image from `Dockerfile`.
- Pushes two tags to Docker Hub:
  - `saadahmedjaaml/spring-bdd:<commit-sha>` — immutable, traceable to the exact commit.
  - `saadahmedjaaml/spring-bdd:latest` — always points to the most recent successful build.

### Stage 3 — Deploy via SSH

- Connects to `deves.xdi.uevora.pt` using an SSH private key stored as a GitHub Secret.
- Pulls `saadahmedjaaml/spring-bdd:latest` from Docker Hub.
- Stops and removes any existing container named `spring-bdd` (safe on first deploy thanks to `|| true` guards).
- Starts the new container on port 8080 with `--restart always` and production database credentials injected as environment variables.

---

## Configuration — GitHub Secrets

All sensitive values are stored as GitHub Actions Secrets (repository → Settings → Secrets and variables → Actions). No credentials are hardcoded in the workflow file.

| Secret | Description |
|---|---|
| `DOCKER_USERNAME` | Docker Hub username (`saadahmedjaaml`) |
| `DOCKER_PASSWORD` | Docker Hub password or access token |
| `DB_NAME` | PostgreSQL database name used by the CI service container |
| `DB_USERNAME` | PostgreSQL username (used in CI and passed to production container) |
| `DB_PASSWORD` | PostgreSQL password (used in CI and passed to production container) |
| `DB_URL` | Full JDBC URL for the production database, e.g. `jdbc:postgresql://host:5432/dbname` |
| `SSH_HOST` | Deployment server hostname — `deves.xdi.uevora.pt` |
| `SSH_USERNAME` | OS login username on the remote server |
| `SSH_PRIVATE_KEY` | PEM-encoded ED25519 private key for SSH authentication (see setup below) |
| `SSH_PORT` | SSH port on the remote server (typically `22`) |

### Setting up the SSH key (one-time)

**Windows (CMD):**

```cmd
rem 1. Create the .ssh directory if it doesn't exist
mkdir %USERPROFILE%\.ssh

rem 2. Generate a dedicated key pair
ssh-keygen -t ed25519 -C "github-actions-deploy" -f %USERPROFILE%\.ssh\deploy_key

rem 3. Print the public key — copy this value
type %USERPROFILE%\.ssh\deploy_key.pub

rem 4. Print the private key — paste this as the SSH_PRIVATE_KEY GitHub Secret
type %USERPROFILE%\.ssh\deploy_key
```

**Linux / macOS:**

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/deploy_key
cat ~/.ssh/deploy_key.pub   # copy this to the server
cat ~/.ssh/deploy_key       # paste as SSH_PRIVATE_KEY secret
```

**Copy the public key to the server** (Windows has no `ssh-copy-id`; do this manually):

```bash
# SSH into the server once with your password
ssh <SSH_USERNAME>@deves.xdi.uevora.pt

# On the server, append the public key to authorized_keys
mkdir -p ~/.ssh
echo "<paste deploy_key.pub content here>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

Add `SSH_PRIVATE_KEY` in GitHub: Settings → Secrets and variables → Actions → New repository secret.

---

## Docker Image

Public repository: **https://hub.docker.com/r/saadahmedjaaml/spring-bdd**

```bash
# Pull the latest image
docker pull saadahmedjaaml/spring-bdd:latest

# Run with a PostgreSQL database
docker run -d -p 8080:8080 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://host:5432/springbdd \
  -e SPRING_DATASOURCE_USERNAME=springuser \
  -e SPRING_DATASOURCE_PASSWORD=springpassword \
  -e SPRING_DATASOURCE_DRIVER=org.postgresql.Driver \
  -e SPRING_JPA_DATABASE_PLATFORM=org.hibernate.dialect.PostgreSQLDialect \
  -e SPRING_JPA_HIBERNATE_DDL_AUTO=update \
  saadahmedjaaml/spring-bdd:latest
```

---

## Running locally with Docker Compose

**Prerequisites:** Docker Desktop (or Docker + Docker Compose plugin).

```bash
# 1. Clone the repository
git clone <your-repo-url>
cd spring-bdd

# 2. Build and start the app + MySQL database
docker-compose up --build

# 3. API is available at
http://localhost:8080

# 4. Stop containers (data volume preserved)
docker-compose down

# 5. Stop and delete the database volume
docker-compose down -v
```

The `docker-compose.yml` starts:

| Service | Image | Port | Role |
|---|---|---|---|
| `app` | Built from `Dockerfile` | `8080` | Spring Boot REST API |
| `db` | `mysql:8.0` | `3306` | Persistent MySQL database |

The app waits for the MySQL healthcheck before starting (`depends_on` + `condition: service_healthy`).

---

## Running tests locally (without Docker)

Tests use H2 in-memory and require no database setup:

```bash
mvn test
```

---

## Design decisions

- **SSH key over password**: GitHub Actions documentation recommends key-based SSH auth. The private key is stored as a secret and never appears in logs.
- **Dual Docker tags**: Every push produces a `latest` tag (for deployment) and a commit-SHA tag (for rollback and traceability).
- **Environment-variable-driven config**: `application.properties` uses `${ENV_VAR:default}` for every datasource property. The default is H2 (works locally with no setup); CI overrides to PostgreSQL; production overrides to whatever DB is configured via `DB_URL`.
- **`|| true` guards**: The deploy script uses `docker stop spring-bdd || true` and `docker rm spring-bdd || true` so the first-ever deployment doesn't fail when no container exists yet.
- **Test reports always uploaded**: The artifact upload step uses `if: always()` so reports are available for debugging even when tests fail.
