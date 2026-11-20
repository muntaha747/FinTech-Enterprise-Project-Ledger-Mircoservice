# FinTech Ledger · CI Build Pipeline (FinTech-Ledger-CI-MicroService)

> **Programming language: Java (Spring Boot).** This repo is a **Java Spring Boot Microservice**: the Ledger service, a REST API for books and ledger entries backed by MySQL.
>
> This repo is the **CI side** (build, test, security scan, image push). Deployment is handled by the companion repo [`FinTech-Ledger-Deployment-MicroService`](https://github.com/muntaha747/FinTech-Ledger-Deployment-MicroService) through **Argo CD** (GitOps).

---

## 🧭 At a Glance

| Item | Value |
|---|---|
| Application | Java 17 · Spring Boot 3.1.2 · MySQL · Maven |
| CI workflow | `.github/workflows/ledgerbuild.yml` (name: *ledger CI Build Pipeline*) |
| Jobs | **2** (`build`, `deployment`) |
| Steps | **20** in total: **19** in `build`, **1** in `deployment` |
| CI/CD tools | **19** (listed in [Tool Stack](#-tool-stack)) |
| Container registry | Amazon ECR (`fintech-ledgerservice`) |
| Deploy targets | functional · regression · integration · uat · release |
| Branching | Trunk-based: short-lived `feature` branch → PR → `main` |

---

## 🗺️ Pipeline Diagram

![CI/CD flow: 20 steps, 19 tools](docs/diagrams/devsecops-full-flow.svg)

The top row is the **CI pipeline** in five phases (19 steps). The bottom row is the **CD handoff** (step 20 and the deployment repo). The bottom toolbox row lists the security and build tools.

---

## 🔀 When the Pipeline Runs

| Event | CI (`build`) | CD (`deployment`) | Image tag |
|---|---|---|---|
| Push to `feature` | ✅ runs | ✅ deploys to **functional** | `fb-<run>-<sha8>` |
| Pull request opened, updated or reopened (to `main` or `feature`) | ✅ runs | ✅ deploys to **functional** | `fb-<run>-<sha8>` |
| Pull request **merged** from `feature` into `main` | ✅ runs | ✅ deploys to **regression** and **integration** | `rel-<run>-<sha8>` |
| Pull request **closed without merge** | ❌ skipped | ❌ skipped | none |
| Manual run (`workflow_dispatch`) | ✅ runs on the chosen branch | ✅ deploys to the environment you pick | `rel-` on `main`, otherwise `fb-` |

**Why the first condition works:** the `build` job has an `if:` condition. It is true for pushes, manual runs, and any pull request event except "closed without merge." A push to `feature` sets `github.event_name` to `push`, so the job always runs for pushes.

---

## 🧩 Step-by-Step Guide

The table lists every step in the order it runs. The **Phase** column matches the diagram.

### Phase 1 · Checkout and secret scan

| # | Step | What it does (plain words) | Tool |
|---|---|---|---|
| 1 | Checkout repository | Downloads the code with its full Git history (`fetch-depth: 0`) so later scans can see every commit | `actions/checkout@v4` |
| 2 | Gitleaks secret scan | Looks through the repo files for passwords, tokens and keys. Writes a SARIF report to `reports/gitleaks.sarif`. Continues even if it finds something; step 9 decides | **Gitleaks** v8.21.2 (Docker) |

### Phase 2 · Build and test

| # | Step | What it does | Tool |
|---|---|---|---|
| 3 | Setup Java 21 | Installs Temurin JDK 21 and caches Maven downloads | `actions/setup-java@v4` · **Temurin JDK** |
| 4 | Docker API version for Testcontainers | Writes `api.version=1.44` into `~/.docker-java.properties` so the Testcontainers library can talk to the runner's Docker | Shell |
| 5 | Maven build + tests (JaCoCo) | `mvn clean verify`: compiles, runs unit and integration tests (MySQL runs in a temporary container), and packages the Spring Boot JAR | **Maven**, **JUnit 5**, **Testcontainers**, **JaCoCo** |
| 6 | Upload JaCoCo report | Saves the coverage report as a downloadable artifact | `actions/upload-artifact@v4` |

### Phase 3 · Security scans

| # | Step | What it does | Tool |
|---|---|---|---|
| 7 | Trivy filesystem scan | Checks `pom.xml` dependencies for known CVEs (skips `target/`). Writes JSON and converts it to SARIF for SonarCloud | **Trivy** v0.50.0 (filesystem) |
| 8 | SonarCloud scan | Static code analysis (SAST) for bugs and security issues. Also imports the Gitleaks and Trivy SARIF reports. Waits for the **quality gate** result | **SonarCloud** (`sonar-maven-plugin`) |
| 9 | Fail if Gitleaks found secrets | If step 2 failed, this step fails the job. It runs after SonarCloud so the findings still reach the dashboard | Shell (`exit 1`) |

### Phase 4 · Build the image and scan it

| # | Step | What it does | Tool |
|---|---|---|---|
| 10 | Decide image tag | Builds the tag from the event, run number and first 8 characters of the commit SHA. `rel-` for merges to `main`, `fb-` for feature builds | Shell |
| 11 | Set up Docker Buildx | Prepares the advanced Docker builder | `docker/setup-buildx-action@v3` · **Docker Buildx** |
| 12 | Build image (local, not pushed) | Builds the image from the `Dockerfile` (Eclipse Temurin 17 JRE on Alpine, runs as a non-root user). Keeps it on the runner for scanning | `docker/build-push-action@v6` · **Docker** |
| 13 | Trivy image scan: full report | Scans the image for OS and library CVEs. Saves all findings to JSON. **Never blocks** | **Trivy** v0.50.0 (image) |
| 14 | Trivy image scan: gate | Fails on fixable **CRITICAL** CVEs. **Temporarily non-blocking** (`continue-on-error`) until the base image and Spring Boot are upgraded | **Trivy** v0.50.0 (image) |

### Phase 5 · Push to ECR and report

| # | Step | What it does | Tool |
|---|---|---|---|
| 15 | Configure AWS credentials | Logs in to AWS with the access keys from GitHub secrets, in `us-east-1` | `aws-actions/configure-aws-credentials@v4` · **AWS IAM** |
| 16 | Login to Amazon ECR | Gets a temporary Docker login token for the ECR registry | `aws-actions/amazon-ecr-login@v2` · **Amazon ECR** |
| 17 | Push image to ECR | Pushes `…/fintech-ledgerservice:<tag>` to the registry | `docker push` |
| 18 | ECR push details and scan report | Waits for ECR's **scan-on-push** result. Saves the image digest, size, push time and findings. Writes a summary on the run page | **AWS CLI**, **jq**, **Amazon ECR scanning** |
| 19 | Upload security reports | Saves every report in `reports/` as an artifact. Runs even if an earlier step failed (`if: always()`) | `actions/upload-artifact@v4` |

### Phase 6 · CD handoff (`deployment` job)

| # | Step | What it does | Tool |
|---|---|---|---|
| 20 | Trigger CD repo for each environment | Works out the target environments from the event (see the table above). For each one, calls the GitHub API to start `deploy.yml` in the deployment repo, then waits 60 seconds | **GitHub REST API**, **GitHub PAT** (`GH_PAT`), `curl` |

After step 20, the deployment repo writes the new image repo and tag into `envs/<env>/values.yaml`, commits, and pushes. **Argo CD** sees the commit and syncs the Helm chart to Kubernetes. See the deployment repo's README for details.

---

## 🛡️ DevSecOps Controls

| Control | Where | Behaviour |
|---|---|---|
| Secret detection | Step 2 + step 9 | **Blocking.** Any secret found fails the build |
| Dependency CVEs (SCA) | Step 7 | Reports only (`continue-on-error`) |
| Code quality and security (SAST) | Step 8 | **Quality gate** on SonarCloud. Step is non-blocking in the workflow, so a failed gate shows on the run but does not stop later steps |
| Image CVEs, full report | Step 13 | Reports only |
| Image CVEs, CRITICAL gate | Step 14 | **Temporarily non-blocking**. Plan: make it blocking after upgrading the base image and Spring Boot |
| ECR scan-on-push | Steps 17–18 | Second, independent scan. Results saved and summarised |
| Least privilege in the image | `Dockerfile` | Runs as a dedicated non-root user (`wezvatech`) with a no-login shell |
| Health probes | Spring Boot Actuator | `/actuator/health/liveness` and `/readiness` for Kubernetes |
| Dependency-Check (OWASP) | `pom.xml` | Configured in the pom but **skipped** in CI with `-Ddependency-check.skip=true` |

---

## 🧰 Tool Stack

| # | Tool | Category | Used in | Why it's here |
|---|---|---|---|---|
| 1 | **GitHub** (repos, PRs) | Source control | All | Code, reviews and triggers |
| 2 | **GitHub Actions** | CI orchestration | All steps | Runs the pipeline as code |
| 3 | **Gitleaks** v8.21.2 | Secret scanning | Steps 2, 9 | Stops secrets reaching `main` |
| 4 | **Temurin JDK 21** | Build runtime | Steps 3, 5 | Compiles the Java 17 project |
| 5 | **Maven** | Build tool | Step 5 | Builds, tests and packages the JAR |
| 6 | **JUnit 5** | Testing | Step 5 | Unit and integration tests |
| 7 | **Testcontainers** | Testing | Steps 4, 5 | Real MySQL in a throwaway container |
| 8 | **JaCoCo** | Code coverage | Steps 5, 6 | Shows how much code the tests cover |
| 9 | **Trivy** (filesystem) | SCA | Step 7 | Known CVEs in `pom.xml` dependencies |
| 10 | **SonarCloud** | SAST, quality gate | Step 8 | Bugs, vulnerabilities, code smells |
| 11 | **Docker** | Containers | Steps 12, 17 | Builds and pushes the image |
| 12 | **Docker Buildx** | Image builds | Step 11, 12 | Modern builder used by build-push-action |
| 13 | **Trivy** (image) | Container scan | Steps 13, 14 | CVEs inside the container image |
| 14 | **AWS IAM** (access keys) | Identity | Step 15 | Gives the runner access to AWS |
| 15 | **Amazon ECR** | Container registry | Steps 16, 17, 18 | Private image storage and scan-on-push |
| 16 | **AWS CLI** | AWS automation | Step 18 | Reads the ECR scan results |
| 17 | **jq** | JSON processing | Step 18 | Turns ECR JSON into the run summary |
| 18 | **GitHub REST API + PAT** | Cross-repo trigger | Step 20 | Starts the deployment repo's workflow |
| 19 | **Argo CD** | GitOps CD | After step 20 | Syncs Git to Kubernetes |

Plus the **Helm** chart in the deployment repo, which Argo CD renders.

**Secrets this repo needs** (GitHub → Settings → Secrets and variables → Actions): `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `SONARCLOUD`, `GH_PAT`.

---

## 🏗️ Application Architecture

The Ledger service is a Spring Boot REST API with three layers:

- **Controller** (`BookController`): receives HTTP requests and returns JSON.
- **Service** (`BookService`): holds the business logic.
- **Repository** (`BookRepository`): Spring Data JPA, which talks to MySQL.

Spring Boot Actuator exposes health and Prometheus metrics. The app reads its MySQL settings from `application.properties`. In the image, `startup.sh` copies in the production config and starts `app.jar`.

---

## 🚀 Run It Locally

```bash
# 1. Build the JAR and skip tests (quick local build)
mvn clean package -Dmaven.test.skip=true

# 2. Start MySQL in a container
docker run --name mydb -p 3306:3306 \
  -e MYSQL_USER=ledger -e MYSQL_PASSWORD=password \
  -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=ledgerdb \
  -d mysql:8.1

# 3. Check MySQL
docker exec -it mydb mysql -uroot -ppassword -e "show databases;"

# 4. Start the application
nohup java -jar target/*.jar > app.log 2>&1 &

# 5. Call the API
curl -s http://localhost:8080/books
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"title":"Book A","price":49.99,"publishDate":"2024-04-12"}' \
  http://localhost:8080/books

# 6. Health probes
curl -s http://localhost:8080/actuator/health/liveness
curl -s http://localhost:8080/actuator/health/readiness
```

---

## 📁 Repository Layout

```
.
├── .github/workflows/ledgerbuild.yml   # the CI pipeline (20 steps, 2 jobs)
├── Dockerfile                          # Temurin 17 JRE image, non-root user
├── startup.sh                          # copies config, starts app.jar
├── pom.xml                             # Maven build, plugins, dependencies
├── src/main/java/com/mkyong/           # Spring Boot application code
├── src/test/java/com/mkyong/           # tests (JUnit 5, Testcontainers)
├── docs/diagrams/                      # diagrams used in this README
└── notes/                              # course notes
```

---

## 🎤 Interview Talking Points

- **Shift left:** secrets, dependencies and code are checked before the image is built.
- **Build once, deploy many:** one image is built per run and promoted by tag. It is not rebuilt for each environment.
- **Traceability:** each tag holds the run number and commit SHA. The image also carries the commit SHA as a label.
- **CI and CD are separate:** GitHub Actions builds and scans. Argo CD deploys from Git. Nobody changes the cluster by hand.
- **Defence in depth:** Gitleaks (secrets), Trivy (dependencies), SonarCloud (code), Trivy (image) and ECR scan-on-push each check a different layer.
- **Honest gaps:** the Trivy CRITICAL gate is non-blocking for now, the OWASP Dependency-Check is skipped, and the workflow builds with JDK 21 while the project targets Java 17. Each one is a known follow-up.
- **A lesson from a real failure:** an image push failed because the build step tried to write to the GitHub Actions cache without the right token scope. Removing the cache setting fixed it.

---

## 🧱 Branching

Trunk-based development: `main` is the trunk. Each change goes through a short-lived `feature` branch:

1. Push to `feature`. CI builds an `fb-` image and deploys to **functional**.
2. Open a PR from `feature` to `main`. CI runs on the PR.
3. Merge the PR. CI builds a `rel-` image and deploys to **regression** and **integration**.
