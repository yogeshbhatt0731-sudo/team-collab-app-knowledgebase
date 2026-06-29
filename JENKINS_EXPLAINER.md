# Workspace-Service CI/CD — Beginner Guide & Evaluator Q&A

This document explains, in plain language, how the `workspace-service` (Spring Boot)
gets built into a Docker image and deployed to the dev server using Jenkins —
mirroring the same flow your frontend (`client`) already uses.

---

## 1. The big picture (say this first to your evaluator)

> "Whenever code is pushed to the **develop** branch, Jenkins automatically:
> builds a Docker image of the app, pushes it to Docker Hub, and deploys it on
> the dev server by pulling that image and running it as a container. This is a
> CI/CD pipeline written as code in a file called a **Jenkinsfile**."

- **CI (Continuous Integration)** = automatically build + test every code change.
- **CD (Continuous Deployment)** = automatically ship that build to a server.
- **Pipeline as Code** = the whole process lives in a `Jenkinsfile` in the repo,
  versioned with Git, instead of being clicked together in the Jenkins UI.

The flow is identical for frontend and backend; only **what** gets built differs:
the frontend builds a Node app served by nginx, the backend builds a Java JAR run by the JVM.

```
Git push (develop)
      │
      ▼
  Jenkins picks up the change
      │
  ┌───┴────────────────────────────────────────────┐
  │ Checkout → Build Image → Login → Push → Deploy  │
  └────────────────────────────────────────────────┘
      │
      ▼
  Dev server pulls image & runs container (port 8080)
```

---

## 2. Two files do the work

| File | Job |
|------|-----|
| `Dockerfile` | The **recipe** to package the app into a portable image. |
| `Jenkinsfile` | The **automation** that builds that image, pushes it, and deploys it. |

Think of the **Dockerfile as the recipe** and the **Jenkinsfile as the chef** who
follows the recipe, then delivers the meal to the customer (dev server).

---

## 3. The Dockerfile — line by line

We use a **multi-stage build**: one stage to compile the code, a second, smaller
stage to run it. This keeps the final image small and safe (no Maven, no source code inside it).

```dockerfile
# ===== Stage 1: Build the JAR with Maven =====
FROM maven:3.9-eclipse-temurin-21 AS builder   # base image that has Maven + Java 21
WORKDIR /app                                    # work inside /app

COPY pom.xml .                                  # copy dependency list first...
COPY .mvn .mvn
COPY mvnw .
RUN ./mvnw -B dependency:go-offline             # ...download deps (cached layer)

COPY src ./src                                  # then copy the actual source code
RUN ./mvnw -B clean package -DskipTests         # compile → produce target/*.jar

# ===== Stage 2: Run the JAR on a small JRE =====
FROM eclipse-temurin:21-jre-alpine              # tiny image: only a Java runtime
WORKDIR /app
RUN addgroup -S spring && adduser -S spring -G spring
USER spring                                     # don't run as root (security)
COPY --from=builder /app/target/*.jar app.jar   # take ONLY the jar from stage 1
EXPOSE 8080                                      # the app listens on 8080
ENTRYPOINT ["java", "-jar", "app.jar"]           # how to start the app
```

**Why two stages?** The build needs Maven + the full JDK + source code (large).
The *running* app only needs the JAR + a Java runtime. Stage 2 throws away
everything from stage 1 except the JAR, so the shipped image is much smaller and
has a smaller attack surface.

**Why copy `pom.xml` before `src`?** Docker caches each step (layer). Dependencies
change rarely, source changes often. By copying `pom.xml` and downloading
dependencies *first*, Docker can reuse that cached layer on later builds and only
re-run the slow download when dependencies actually change.

---

## 4. The Jenkinsfile — section by section

The whole file is wrapped in `pipeline { ... }` — this is **Declarative Pipeline**
syntax (the recommended, structured style).

### `agent any`
"Run this pipeline on any available Jenkins worker (node)." A Jenkins setup has a
controller and one or more agents that actually execute the work.

### `environment { ... }`
Variables reused across the pipeline, defined in one place:

```groovy
IMAGE_NAME = "yogesh177/team-collab-workspace-service"  # Docker Hub repo name
IMAGE_TAG  = "${BUILD_NUMBER}"                           # unique number per build
DEV_SERVER = "35.173.179.185"                            # where we deploy
DEV_USER   = "ubuntu"                                    # SSH user on that server
APP_NAME   = "workspace-service"                         # container name
APP_PORT   = "8080"                                      # published port
```

`BUILD_NUMBER` is a built-in Jenkins variable that increases by 1 each run
(1, 2, 3, …), so every image gets a **unique, traceable tag**.

### Stage 1 — `Checkout`
```groovy
checkout scm
```
Pulls the source code from Git (the same repo/branch that triggered the build).
`scm` = "Source Code Management", i.e. the Git config Jenkins already knows about.

### Stage 2 — `Build Docker Image`
```groovy
dir('workspace-service') {
    docker build -t IMAGE:BUILD_NUMBER -t IMAGE:develop-latest .
}
```
- `dir('workspace-service')` — step into the backend folder (the repo holds both
  `client` and `workspace-service`, so we must point at the right one).
- `docker build ... .` — read the `Dockerfile` in this folder and build the image.
- **Two tags** on the same image:
  - `:BUILD_NUMBER` → an immutable version (e.g. `:42`) you can roll back to.
  - `:develop-latest` → a moving pointer that always means "newest on develop".

### Stage 3 — `Docker Login`
```groovy
withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', ...)]) {
    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
}
```
- `withCredentials` pulls the Docker Hub username/password from Jenkins'
  encrypted credential store — **secrets are never written in the Jenkinsfile**.
- `--password-stdin` pipes the password in instead of putting it on the command
  line (so it won't leak into process lists or logs).

### Stage 4 — `Push Image`
```groovy
docker push IMAGE:BUILD_NUMBER
docker push IMAGE:develop-latest
```
Uploads both tags to Docker Hub so the dev server can later pull them.

### Stage 5 — `Deploy To Dev`
```groovy
when { branch 'develop' }
```
- `when` is a **guard**: this stage only runs if the branch is `develop`. A feature
  branch can build & push an image but will **not** deploy — that protects the dev server.

```groovy
withCredentials([ string(... 'workspace-db-url' ...), ... ]) {
  sshagent(['dev-server-key']) {
    ssh ubuntu@DEV_SERVER '
        docker pull  IMAGE:develop-latest
        docker stop  workspace-service || true
        docker rm    workspace-service || true
        docker run -d --name workspace-service \
            -p 8080:8080 \
            -e SPRING_DATASOURCE_URL="$DB_URL" \
            -e SPRING_DATASOURCE_USERNAME="$DB_USER" \
            -e SPRING_DATASOURCE_PASSWORD="$DB_PASS" \
            --restart unless-stopped \
            IMAGE:develop-latest
    '
  }
}
```
Step by step on the dev server:
1. `sshagent(['dev-server-key'])` — loads the private SSH key (stored in Jenkins)
   so Jenkins can log into the dev server without a password.
2. `docker pull` — download the freshly pushed image.
3. `docker stop` + `docker rm` — stop and remove the old running container.
   `|| true` means "don't fail the build if it doesn't exist yet" (e.g. first deploy).
4. `docker run -d` — start the new container in the background (`-d` = detached).
   - `-p 8080:8080` maps host port → container port.
   - `-e SPRING_DATASOURCE_*` injects DB connection settings at runtime. Spring Boot
     automatically reads these environment variables and overrides
     `application.properties`. **This is why DB credentials are NOT baked into the image.**
   - `--restart unless-stopped` auto-restarts the container if it crashes or the
     server reboots.

### `post { ... }`
Runs after all stages, regardless of result:
- `always { docker logout }` — always log out of Docker Hub to clean up the session.
- `success` / `failure` — print a clear status message.

---

## 5. IMPORTANT: the database

The app's `application.properties` points at `jdbc:mysql://localhost:3306/...`.
Inside a container, `localhost` means **the container itself**, not the host or a
MySQL server — so the default value will not work in production.

That is exactly why the deploy stage **overrides** the datasource via
`-e SPRING_DATASOURCE_URL=...`. On the dev server you need a reachable MySQL. Two
common options to mention:
- Run MySQL as another container on a shared Docker network and point the URL at
  `jdbc:mysql://mysql:3306/workspace_db` (using the container name as hostname).
- Use a managed MySQL (e.g. AWS RDS) and put its URL in the Jenkins credential.

Store the real URL/username/password as Jenkins **Secret text** credentials named
`workspace-db-url`, `workspace-db-username`, `workspace-db-password`.

---

## 6. One-time setup the evaluator may ask about

These must exist in Jenkins for the pipeline to run:
1. **Credentials**
   - `dockerhub-creds` — Username/Password for Docker Hub.
   - `dev-server-key` — SSH private key for the dev server.
   - `workspace-db-url`, `workspace-db-username`, `workspace-db-password` — Secret text.
2. **A Pipeline (or Multibranch) job** pointing at this repo, with
   "Script Path" set to `workspace-service/Jenkinsfile`.
3. **Docker installed** on both the Jenkins agent and the dev server.
4. A **webhook** (GitHub/GitLab → Jenkins) so a push to `develop` triggers the job.

---

## 7. Evaluator Q&A — basic, tricky, and cross questions

**Q: What is a Jenkinsfile?**
A text file in the repo that defines the CI/CD pipeline as code, so the process is
versioned, reviewable, and reproducible instead of configured by clicking in the UI.

**Q: Declarative vs Scripted pipeline — which is this?**
Declarative (it starts with `pipeline { }` and uses structured `stages`/`steps`).
It's simpler and more readable. Scripted pipelines start with `node { }` and are
full Groovy — more flexible but harder.

**Q: What is `agent any`?**
It tells Jenkins to run the pipeline on any available agent/node. You could pin it
to a labeled agent (e.g. `agent { label 'docker' }`) if only some nodes have Docker.

**Q: What does `checkout scm` do?**
Checks out the exact repo and branch/commit that triggered this build, using the
Git settings already configured on the job.

**Q: Why two tags (`BUILD_NUMBER` and `develop-latest`)?**
`BUILD_NUMBER` gives an immutable, traceable version you can roll back to.
`develop-latest` is a convenient moving pointer the deploy step always pulls.

**Q: Why a multi-stage Dockerfile?**
To keep the final image small and secure: the build tools (Maven, JDK, source) stay
in stage 1 and are discarded; only the JAR + a slim JRE ship in stage 2.

**Q: Difference between JDK and JRE here?**
The build stage needs the JDK (compiler) to produce the JAR. The run stage only
needs the JRE (runtime) to execute it — smaller and enough to run the app.

**Q: Why `-DskipTests` in the Dockerfile build?**
To keep the image build fast and focused on packaging. Best practice is to run
tests in a **separate Jenkins stage** before building the image, so a failing test
stops the pipeline. (Good improvement to suggest — see section 8.)

**Q: Where are secrets stored? Are they in the Jenkinsfile?**
No. They live in Jenkins' encrypted credential store and are injected at runtime
via `withCredentials`. The Jenkinsfile only references their IDs.

**Q: Why `--password-stdin` for docker login?**
So the password isn't passed as a visible command-line argument (which can leak in
logs or `ps` output). It's read from standard input instead.

**Q: What does `|| true` mean in `docker stop x || true`?**
"If this command fails, treat it as success." On the first deploy there's no old
container to stop/remove, and we don't want that to fail the build.

**Q: Why `when { branch 'develop' }`?**
A safety guard so only the develop branch deploys to the dev server. Other branches
can still build/push images but won't touch the running environment.

**Q: How does Spring Boot pick up `SPRING_DATASOURCE_URL` from `-e`?**
Spring Boot's "relaxed binding" maps the env var `SPRING_DATASOURCE_URL` to the
property `spring.datasource.url`, and environment variables override
`application.properties`. So we configure the DB without rebuilding the image.

**Q: Why does `localhost` in the DB URL not work inside Docker?**
Inside a container, `localhost` refers to the container itself, not the host. You
must point to the DB's real hostname (another container's name on a shared network,
or an external DB host).

**Q: What triggers the pipeline?**
A webhook from Git on push to `develop` (or a poll/manual run). Jenkins then runs
the stages top to bottom.

**Q: What happens if the build fails midway?**
The pipeline stops at the failing stage; later stages don't run. The `post { always }`
block still runs (so we still `docker logout`), and `failure` prints a message.

**Q: How would you roll back a bad deploy?**
Re-run `docker run` with a previous immutable tag, e.g. `IMAGE:41` instead of
`develop-latest`, on the dev server.

**Q: Is `--restart unless-stopped` important?**
It makes Docker restart the container automatically after a crash or server reboot,
unless you deliberately stopped it. Improves uptime.

**Q: How is this different from the frontend pipeline?**
Same stages and flow. Differences: the backend image is built from a Java JAR
(multi-stage Maven build) and runs on the JVM exposing 8080; the frontend builds a
Node bundle served by nginx on port 80. The backend also needs DB env vars.

---

## 8. Honest shortcomings (be ready to volunteer these — evaluators love it)

These apply to **both** the frontend and backend pipelines as written:

1. **No test / quality stage.** Nothing runs unit tests, linting, or a security scan
   before building the image. A bad commit can still be deployed. *Fix:* add a
   `Test` stage (`./mvnw test`) and fail fast.
2. **No build/deploy separation for non-develop branches isn't fully clean** — the
   image is built and pushed even on branches that never deploy, which wastes
   Docker Hub space. *Fix:* gate build/push too, or use PR builds.
3. **`:develop-latest` is mutable.** Pulling "latest" can be ambiguous; pinning to
   the immutable `BUILD_NUMBER` tag in deploy is safer/traceable.
4. **No health check before declaring success.** The pipeline ends once the
   container starts, not once the app is actually serving. *Fix:* curl an actuator
   health endpoint after `docker run` and fail if it's not `UP`.
5. **No image cleanup.** Old images pile up on the Jenkins agent and dev server.
   *Fix:* prune old images, or keep only the last N.
6. **`StrictHostKeyChecking=no`** disables SSH host-key verification — convenient but
   weakens security (susceptible to man-in-the-middle). *Fix:* pre-load known_hosts.
7. **Single environment, no approval gate.** Goes straight to dev; there's no
   manual approval step before any higher environment. *Fix:* add an `input` step
   for staging/prod promotion.
8. **No rollback automation.** Rollback is manual. *Fix:* keep the previous tag and
   add a rollback path on failure.
9. **No notifications.** No Slack/email on success/failure. *Fix:* add notifications
   in the `post` block.
10. **Secrets at runtime via `-e` can appear in `docker inspect`** on the host.
    Acceptable for dev; for prod prefer Docker secrets or a vault.
