# DevOps Study Notes — Spring Boot, Docker, CI/CD

Crisp revision notes + real interview questions. Built around the workspace-service backend.

---

## 1. Deploying a Spring Boot app (the mental model)

- React build → `dist/` = **dead static files**, need a web server (nginx) to serve them.
- Spring Boot build → a **fat JAR** = a **runnable program** with its own web server (Tomcat) inside.
- So "deploy" a Spring Boot app = just run it: `java -jar app.jar`. No separate Tomcat to install.
- Default port: **8080**.
- Manual deploy in 6 steps:
  1. `./mvnw clean package` → builds the jar
  2. Install a **JRE** (Java runtime) on the server
  3. `scp` the jar to the server
  4. Ensure a **MySQL** database is reachable
  5. `java -jar app.jar` → app starts, serves on 8080
  6. Keep it alive with **systemd** (manual) or `--restart` (Docker)

**Interview Qs**
- *How do you deploy a Spring Boot app?* Build the fat jar, put it on a server with a JRE, run `java -jar`. The embedded server means no external app server needed.
- *JAR vs WAR?* WAR deploys *into* an external server; fat JAR carries its own server and runs standalone. Spring Boot favours the fat JAR.
- *What is a fat/uber jar?* A single jar bundling your code + all dependencies + embedded Tomcat + a manifest naming the main class.
- *Why does a Spring Boot app need no external Tomcat?* Tomcat is embedded inside the jar.

---

## 2. Maven Wrapper (`mvnw`) — kill "works on my machine"

- `mvn` = the Maven installed **globally** on a machine; version varies per machine.
- `mvnw` = **Maven Wrapper**: a script + pinned-version config **committed into the repo**.
- `./mvnw` auto-downloads the exact pinned Maven version, then builds with it.
- Guarantees **identical builds** on laptop, Jenkins agent, and Docker — none need Maven pre-installed.
- Frontend analogy: like pinning npm/Node + committing `package-lock.json`.
- `pom.xml` = Maven's config (dependencies + build), the Java `package.json`.
- Caveat: `mvnw` pins **Maven only**, not the JDK — Java still has to be present (base image handles that).

**Interview Qs**
- *`mvn` vs `mvnw`?* `mvn` uses whatever Maven is installed; `mvnw` pins and auto-downloads the exact version from the repo → reproducible builds.
- *Why does the Dockerfile call `./mvnw` not `mvn`?* The builder image has no Maven installed; the wrapper brings the right version.
- *What is `pom.xml`?* The Maven project file listing dependencies and build config.

---

## 3. Building the JAR — `./mvnw clean package`

- One command, phases run **in order**: `clean` → `compile` → `test` → `package`.
- You can't skip ahead — asking for `package` auto-runs compile + test first.
- If a test fails, the build stops at `test` (that's the point of tests in CI).
- Output: `target/<artifactId>-<version>.jar` (e.g. `workspace-service-0.0.1-SNAPSHOT.jar`).
- `SNAPSHOT` = in-development, not a final release version.
- `-DskipTests` skips the test phase (used in the Docker build; tests belong in a separate CI stage).
- `-B` = batch/non-interactive mode (cleaner CI logs).

**Interview Qs**
- *What does `mvn package` do?* Runs validate→compile→test→package and produces the jar.
- *What's `-DskipTests` and is skipping tests bad?* Skips tests during this build; not bad if tests run in a dedicated earlier stage that fails the pipeline.
- *Where does Maven download libraries?* From Maven Central into the local `~/.m2` cache, then reuses them.

---

## 4. Docker basics — image vs container

- **Image** = a frozen, read-only template (sealed box on a shelf). Built by `docker build`.
- **Container** = a running instance of an image. Created by `docker run`.
- One image → many containers (like class → objects, recipe → cooked meals).
- `EXPOSE 8080` = documentation only; does **NOT** publish the port. `-p 8080:8080` at run time actually publishes it.
- Run as a **non-root user** (`USER spring`) to limit damage if the container is compromised.

**Interview Qs**
- *Image vs container?* Image = template on disk; container = running instance of it.
- *Does `EXPOSE` open/publish the port?* No — it's documentation; `-p` publishes it.
- *Why run containers as non-root?* Security — limits blast radius if compromised.

---

## 5. Multi-stage Docker build — small, secure images

- **Stage 1 (builder):** heavy base with Maven + JDK + source → compiles the jar. Temporary.
- **Stage 2 (runtime):** slim JRE base; `COPY --from=builder` pulls **only the jar**.
- Everything in stage 1 (Maven, JDK, source) is **thrown away** — never shipped.
- Result: image is small, has no build tools/source (smaller attack surface), and is clean.
- Build needs heavy tools; running needs only JRE + jar — that's the whole reason for two stages.

**Interview Qs**
- *Why a multi-stage build?* Build with heavy tools in stage 1, ship only the jar + slim JRE in stage 2 → small, secure image.
- *JDK vs JRE?* JDK compiles (build stage); JRE runs (runtime stage). JRE is smaller.
- *How do you shrink a large image?* Multi-stage build, slim/alpine base, good `.dockerignore`.

---

## 6. Docker layers & build caching

- Each Dockerfile instruction = one **layer** (a saved filesystem snapshot). An image = a stack of layers.
- **Caching rule:** a layer is reused if its instruction AND all layers above it are unchanged. Otherwise that layer **and everything below it** rebuild (the cascade).
- Put **rarely-changing** steps near the top, **frequently-changing** near the bottom.
- That's why we `COPY pom.xml` + download deps **before** `COPY src`: editing code doesn't bust the slow dependency-download layer.
- Layers are **shared/reused** across images and pulls.
- Deletions in later layers don't shrink earlier layers — download+use+delete in one `RUN`, or use multi-stage.
- `.dockerignore` keeps junk (`target/`, `.git/`) out of `COPY` → avoids cache busts + bloat.

**Interview Qs**
- *What is a Docker layer?* A snapshot of one instruction's filesystem change; image = stack of them.
- *How does build caching work / what invalidates it?* A changed instruction, changed copied files, or any changed layer above → that layer and all below rebuild.
- *Why copy `pom.xml`/`package.json` before the source?* To keep the slow dependency layer cached.
- *Deleted a file but image still big — why?* It still exists in an earlier layer.

---

## 7. Why the database is a SEPARATE container

- Docker rule: **one service per container**. The Dockerfile builds **one** image = your app only.
- DB is a **run-time dependency** the app talks to — not something baked in at build time.
- Reasons not to bundle MySQL into the app image:
  - Containers are disposable → redeploy would **wipe the database**.
  - DB needs a **persistent volume**; app containers are replaceable.
  - **Independent scaling / lifecycle** (e.g. 3 app copies, 1 DB).
  - One main process per container — two servers in one is an anti-pattern.
- How they connect (run time, not Dockerfile):
  - **Docker Compose** (dev): both services + shared network; app reaches DB by **service name** as hostname → `jdbc:mysql://mysql:3306/...`.
  - **Managed DB** (prod, e.g. AWS RDS): app points at its URL via `SPRING_DATASOURCE_URL` env var.
- `localhost` inside a container = the container itself, not the host/DB — that's why the default URL fails in Docker.

**Interview Qs**
- *Why not put MySQL in the same container/image as the app?* One service per container; data must persist across redeploys; independent scaling.
- *How do two containers talk?* Same Docker network, reach each other by container/service name.
- *What wires multiple containers together?* Docker Compose (one host); Kubernetes at scale.
- *What happens to data when a container is removed?* Lost unless in a volume or external DB.

---

## 8. ENTRYPOINT vs CMD (the note in the Dockerfile)

- Both define what runs when the container starts — but behave differently.
- **CMD** = a **default** command/args, easily **overridden** by args in `docker run image <something>`.
- **ENTRYPOINT** = the **fixed** executable; args from `docker run` are **appended** to it.
- We use `ENTRYPOINT ["java","-jar","app.jar"]` because the container's job is **always** to run the app — it should behave like an executable, not be accidentally replaced.
- Common best-practice combo: `ENTRYPOINT` = the program, `CMD` = default arguments you can override.
- **Exec form** `["java","-jar","app.jar"]` (JSON array) vs shell form `java -jar app.jar`:
  - Exec form → Java is **PID 1**, receives signals (SIGTERM) → **graceful shutdown**.
  - Shell form → wrapped in `/bin/sh -c`, Java is not PID 1, may not get stop signals.
  - Always prefer **exec form** for the main process.

**Interview Qs**
- *ENTRYPOINT vs CMD?* ENTRYPOINT = fixed executable; CMD = default args, easily overridden. Often used together.
- *Why ENTRYPOINT (not CMD) for the app?* The container should always run the app and not be trivially overridden.
- *Exec form vs shell form — why does it matter?* Exec form makes your process PID 1 so it receives signals and shuts down gracefully.
- *Can you override ENTRYPOINT?* Yes, with `docker run --entrypoint ...`, but normal args just get appended.

---

## 9. The CI/CD pipeline (how it all connects)

- **CI** = auto build + test each change. **CD** = auto ship it to a server.
- **Pipeline as code** = the whole flow lives in a `Jenkinsfile` in the repo (versioned, reviewable).
- Flow on push to `develop`: **Checkout → Build image → Docker login → Push to Docker Hub → Deploy to dev**.
- Two image tags: `:BUILD_NUMBER` (immutable, rollback-able) + `:develop-latest` (moving pointer).
- Registry (Docker Hub) = shared shelf both Jenkins and the server reach (server can't see your laptop).
- Secrets (Docker Hub creds, SSH key, DB creds) live in **Jenkins credentials**, never in the Jenkinsfile.
- DB credentials injected at run time via `-e SPRING_DATASOURCE_*` (Spring "relaxed binding" maps env vars to properties) → never baked into the image.
- `when { branch 'develop' }` = only deploy from develop (safety guard).

**Interview Qs**
- *What is a Jenkinsfile / pipeline as code?* CI/CD defined as a versioned file in the repo instead of UI clicks.
- *Why two image tags?* Immutable version for rollback + a convenient "latest" pointer.
- *Where are secrets stored?* Jenkins encrypted credential store, referenced by ID via `withCredentials`.
- *How does the app get DB config without rebuilding?* Env vars override `application.properties` at run time (relaxed binding).
- *How would you roll back?* Re-run the previous immutable tag (e.g. `:41`).

---

## 10. Debugging lessons (from real errors we hit)

- **Read stack traces bottom-up** — the last `Caused by:` is the real cause; the top line is just the symptom.
  - Example: "Failed to load ApplicationContext" (symptom) → real cause was springdoc version mismatch.
- **Version/compatibility mismatches** are a top cause of weird startup errors.
  - springdoc `2.8.16` (Boot 3) broke on Spring Boot `4.1.0` → fix: bump to `3.0.3`.
- **Missing dependency** = compile error like `package org.junit.jupiter.api does not exist` → add `spring-boot-starter-test`.
- **`java:` prefix on an error = the IDE's compiler**, not Maven. If the pom is correct, reload Maven / rebuild / invalidate caches. Verify truth on the command line with `./mvnw test-compile`.
- `@SpringBootTest` loads the **full context** (incl. DB) → in CI use an in-memory **H2** DB, Testcontainers, or `@DataJpaTest`.

**Interview Qs**
- *How do you debug a Spring Boot startup failure?* Read the stack trace bottom-up to the last `Caused by:`; check dependency versions and config.
- *App fails only in CI but works locally — why?* Often a missing DB/service; use an embedded DB or Testcontainers for tests.
- *Build works on the command line but the IDE shows missing packages — why?* IDE classpath is stale; reload the Maven project.

---

### One-line summaries to memorise
- Fat jar = app + libs + embedded server in one runnable file.
- `mvnw` pins Maven so builds are identical everywhere.
- Multi-stage build = build with heavy tools, ship only the jar.
- Layer caching = unchanged layer (and all above) reused; order cheap-to-expensive top, change-prone bottom.
- One container, one service — DB is separate, connected at run time.
- ENTRYPOINT = fixed program; CMD = default args.
- Read stack traces bottom-up.
