# Deployment & CI/CD — dev + prod on 2 EC2 instances

How the team-collab microservices are built and deployed with Jenkins + Docker,
self-managed MySQL (no RDS), and Docker networks.

---

## 1. The two machines

| Instance | Role | Runs |
|----------|------|------|
| **inst1** | Jenkins build server | Jenkins controller/agent; runs `docker build`, pushes images to Docker Hub |
| **inst2** | Deploy server | All running containers: dev + prod copies of every service, plus the MySQL containers |

inst1 never runs your app; it only builds images and SSHes into inst2 to deploy.
inst2 never builds anything; it only pulls images and runs them.

---

## 2. Branch → environment

GitFlow drives everything:

- `feature/*` → merged into `develop` (no deploy on feature branches).
- `develop` → builds and deploys to the **dev** environment on inst2.
- `main` → builds and deploys to the **prod** environment on inst2 (after a manual approval click).

One `Jenkinsfile` per service handles both: a `Configure environment` stage reads
`BRANCH_NAME` and sets the network, host port, DB host, and image tag accordingly.

---

## 3. Networks — why and how

Each environment gets its **own isolated Docker network**:

- `dev-net` — all dev containers (services + `mysql-dev`).
- `prod-net` — all prod containers (services + `mysql-prod`).

Inside a network, containers reach each other **by container name** (Docker's built-in DNS):

- service → its database: `jdbc:mysql://mysql-dev:3306/workspace_db`
- service → service: `http://auth-service-dev:8080`

Isolation is the point: a dev container **cannot** reach `mysql-prod`, and prod cannot
see dev. That prevents a dev mistake from ever touching production data.

> `localhost` inside a container = that container itself. That's why services must use
> the DB **container name** as the host, never `localhost`.

---

## 4. MySQL — self-managed, database-per-service

No RDS, so MySQL runs as a container on inst2 — **one MySQL server per environment**:

- `mysql-dev` (on dev-net) holds: `auth_db`, `workspace_db`, `notification_db`
- `mysql-prod` (on prod-net) holds the same three databases

This keeps "database-per-service" (each service owns its own logical database and only
knows its own URL/credentials) while staying light on resources — one MySQL process per
env instead of three. Each service auto-creates its database on first connect because the
JDBC URL has `createDatabaseIfNotExist=true`, so no manual SQL is needed.

Data lives in a **named volume** (`mysql-dev-data`, `mysql-prod-data`), so it survives
container removal and app redeploys.

> Stricter alternative (more isolation, more RAM): run a separate MySQL container per
> service. For your hardware, one-per-env is the pragmatic choice.

---

## 5. Port map (on inst2)

Container port is always **8080** internally; host ports are unique so dev and prod don't clash.

| Service | dev container | dev host port | prod container | prod host port |
|---------|---------------|---------------|----------------|----------------|
| auth-service | `auth-service-dev` | 8081 | `auth-service-prod` | 9081 |
| workspace-service | `workspace-service-dev` | 8082 | `workspace-service-prod` | 9082 |
| notification-service | `notification-service-dev` | 8083 | `notification-service-prod` | 9083 |
| MySQL | `mysql-dev` | 3307 (optional) | `mysql-prod` | not published |

Rule of thumb used here: dev = 808x, prod = 908x. Host ports only matter for reaching a
service from outside; service-to-service traffic uses the internal `:8080` over the network.

---

## 6. One-time inst2 setup

Run `infra/inst2-setup.sh` **once** on inst2 (it creates the networks, volumes, and both
MySQL containers). The Jenkins pipelines assume these already exist — they only deploy the
app containers. See that script for the exact commands.

---

## 7. Jenkins prerequisites (per service)

Create these once in Jenkins:

**Credentials**
- `dockerhub-creds` — Docker Hub username/password (shared by all services).
- `deploy-server-key` — SSH private key for inst2 (shared).
- `mysql-dev-password`, `mysql-prod-password` — Secret text (DB root passwords per env).

**Jobs** — one **Multibranch Pipeline** per service, each pointed at the monorepo with
*Script Path* set to that service's Jenkinsfile (e.g. `workspace-service/Jenkinsfile`).

**Monorepo path filtering (important)** — so a change in one service doesn't rebuild all
three: in each job's branch source, set an *Included Region* (e.g. `workspace-service/.*`).
That way pushing only to `workspace-service/` triggers only that job.

---

## 8. How ALL services' pipelines look (the final picture)

Every service uses the **same Jenkinsfile template**. Only four things differ per service:

| Knob | auth-service | workspace-service | notification-service |
|------|--------------|-------------------|----------------------|
| `IMAGE_NAME` | …-auth-service | …-workspace-service | …-notification-service |
| container name | `auth-service-<env>` | `workspace-service-<env>` | `notification-service-<env>` |
| host port (dev/prod) | 8081 / 9081 | 8082 / 9082 | 8083 / 9083 |
| DB name | `auth_db` | `workspace_db` | `notification_db` |

Everything else — branch→env logic, build, test, login, push, approval, deploy, networks,
DB host (`mysql-dev`/`mysql-prod`) — is identical. So the whole system is:

```
3 services × 1 shared pipeline template
  → each a Multibranch job watching its own folder
  → develop builds → deploys to dev-net on inst2
  → main builds → (approval) → deploys to prod-net on inst2
  → shared infra (networks + MySQL) provisioned once by inst2-setup.sh
```

The frontend (`client`) follows the same shape but builds a Node/nginx image and publishes
port 80 instead of 8080.

---

## 9. Practical notes / gotchas

- **Resources.** inst2 will run up to 6 app containers + 2 MySQL containers. That needs real
  RAM — aim for 4 GB minimum, ideally more, and add swap. Consider only running prod once dev
  is validated, rather than all six simultaneously, if the box is small.
- **inst1 is 2 GB.** A multi-stage `docker build` runs Maven inside the build container and is
  memory-hungry; 2 GB can OOM. Add a swap file on inst1 if builds get killed.
- **Secrets over SSH.** The deploy passes the DB password via `-e` inside the SSH command.
  Fine for dev/small prod; for stronger prod, use Docker secrets or a vault.
- **`createDatabaseIfNotExist=true` in prod** is convenient but auto-creates schemas; some
  teams disable it in prod and manage schema with migrations (Flyway/Liquibase).
- **Build once, promote** (advanced): rather than rebuilding on `main`, you can promote the
  exact image already tested on `develop`. The current setup rebuilds per branch, which is
  simpler to reason about while learning.
