# Dev Environment — Implementation Guide (workspace-service)

A step-by-step, zero-to-deployed guide for the **dev** environment only.
Prod is intentionally left as **future scope**.

---

## 0. The goal in one sentence

When you merge into `develop`, Jenkins (on inst1) tests the code, builds a Docker image,
pushes it to Docker Hub, then SSHes into inst2 and runs it as a container named
`workspace-service-dev` on the `dev-net` network, talking to a `mysql-dev` container — and
the running API is reachable at `http://<inst2-ip>:8082`.

---

## 1. Machines & roles

| Machine | Role | Needs |
|---------|------|-------|
| **inst1** | Jenkins build server | Docker, Jenkins, network access to Docker Hub + inst2 |
| **inst2** | Deploy server | Docker, the `dev-net` network, the `mysql-dev` container |

---

## 2. Prerequisites on both EC2 instances

On **inst1** and **inst2**:

```bash
# install Docker
sudo apt update && sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER     # log out/in so 'docker' works without sudo
```

On **inst1** also install Jenkins (or run it as a container) and make sure the Jenkins user
can run `docker` (add it to the docker group). Jenkins also needs the **SSH Agent**,
**Credentials**, and **Pipeline** plugins (default in most installs).

> inst1 is 2 GB. A multi-stage `docker build` runs Maven inside the build container and can
> run out of memory. If builds get killed, add a swap file:
> ```bash
> sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
> sudo mkswap /swapfile && sudo swapon /swapfile
> ```

---

## 3. One-time infra on inst2

Copy `infra/inst2-setup.sh` to inst2 and run it once:

```bash
export DEV_DB_PASS='choose-a-strong-dev-password'
bash inst2-setup.sh
```

This creates the `dev-net` network and the `mysql-dev` container (with a persistent volume).
Verify:

```bash
docker network ls | grep dev-net
docker ps | grep mysql-dev
```

You do **not** create the app container by hand — the pipeline does that.

---

## 4. App prerequisites (already covered earlier)

Make sure these are in place in `workspace-service` so the build is green:

1. `pom.xml`: springdoc `3.0.3` (Spring Boot 4 compatible) + `spring-boot-starter-test`.
2. H2 test database for `./mvnw test`:
   - add `com.h2database:h2` (test scope),
   - add `src/test/resources/application.properties` pointing at an in-memory H2.
3. The multi-stage `Dockerfile` (build stage + slim JRE runtime stage).

---

## 5. Jenkins credentials (create these 3)

In Jenkins → Manage Jenkins → Credentials → (global) → Add:

| ID | Kind | What it is |
|----|------|------------|
| `dockerhub-creds` | Username/password | Your Docker Hub login (to push images) |
| `deploy-server-key` | SSH username with private key | Private key that can `ssh ubuntu@inst2` |
| `mysql-dev-password` | Secret text | The `DEV_DB_PASS` you set in step 3 |

The Jenkinsfile references these by ID only — no secret is ever written in the repo.

---

## 6. Create the Jenkins job

1. New Item → **Multibranch Pipeline** (so branch names like `develop` are detected).
2. Branch Source → Git/GitHub → your repo URL + credentials.
3. Build Configuration → *by Jenkinsfile* → **Script Path:** `workspace-service/Jenkinsfile`.
4. (Monorepo tip) In the branch source's advanced behaviours, set an **Included Region**:
   `workspace-service/.*` — so changes to other services don't trigger this job.
5. Save. Jenkins scans the repo and finds the `develop` branch.

Optional: add a GitHub webhook so a push auto-triggers the scan; otherwise click
"Scan Repository Now" / "Build Now".

---

## 7. Run it

```bash
git checkout develop
git commit --allow-empty -m "trigger dev pipeline"
git push origin develop
```

Watch the stages in Jenkins: Checkout → Build & Test → Build Image → Docker Login →
Push Image → Deploy To Dev.

---

## 8. Verify the deployment (on inst2)

```bash
docker ps                                  # see workspace-service-dev running
docker logs -f workspace-service-dev       # watch Spring Boot start, "Started ... on 8080"
```

From your laptop / browser:

```bash
curl http://<inst2-ip>:8082/actuator/health      # if actuator is enabled
# or open the API docs:
http://<inst2-ip>:8082/swagger-ui/index.html
```

> Make sure the EC2 **security group** for inst2 allows inbound TCP on **8082** (and 22 for SSH).

Check the database was created:

```bash
docker exec -it mysql-dev mysql -uroot -p -e "SHOW DATABASES;"   # expect workspace_db
```

---

## 9. How a redeploy works (mental model)

Every push to `develop` repeats: build a fresh image → push → on inst2 `docker pull` the new
`develop-latest`, `docker stop`/`rm` the old container, `docker run` the new one. The
container is disposable; your data is safe because it lives in the **mysql-dev volume**, not
in the app container.

---

## 10. Troubleshooting quick table

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Build killed during `docker build` | inst1 out of memory | Add swap (step 2) |
| `Deploy` stage: permission denied (publickey) | wrong/missing SSH key | Check `deploy-server-key` + inst2 `authorized_keys` |
| App container exits immediately | can't reach DB | Ensure it's on `--network dev-net`; URL host is `mysql-dev`, not `localhost` |
| `curl` to :8082 times out | security group closed | Open inbound 8082 on inst2 |
| `Communications link failure` in logs | mysql-dev not ready/started | `docker ps` mysql-dev; restart it; redeploy |
| Tests fail in `Build & Test` | H2/springdoc not set up | Do step 4 |

---

## 11. What to write as "future scope" in your report

- **Prod environment**: a second isolated `prod-net` + `mysql-prod` (no public DB port), a
  `main → prod` pipeline stage gated by manual approval (`input`), and prod-tagged images.
- **Build-once-promote**: deploy to prod the exact image already tested in dev, instead of
  rebuilding.
- **Secrets hardening**: Docker secrets / Vault instead of passing the DB password via `-e`.
- **Observability**: health-check gate after deploy, plus Slack/email notifications.
- The full prod design is already sketched in `docs/deployment-and-cicd.md`.
