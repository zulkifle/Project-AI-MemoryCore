---
name: trustgate-gitops-cicd
description: "MUST use when setting up or changing CI/CD for a Trustgate project on
             gitlab.msctrustgate.com — writing a .gitlab-ci.yml, adding the
             review/build/release pipeline, onboarding an app to Argo CD /
             trustgate/gitops, or debugging a failed release job. Trigger on
             'buat CI/CD', 'setup CI CD', 'gitlab ci', '.gitlab-ci.yml', 'pipeline',
             'follow tgnra', 'macam tgnra', 'gitops', 'argocd', 'argo cd',
             'release job failed', 'GITOPS_TOKEN is missing', 'not releasing',
             'Degraded on', or when Dejul names a project and asks for CI/CD."
---

# Trustgate GitOps CI/CD — review → build → release via Argo CD
*How Trustgate apps are built and deployed: GitLab CI pushes an image to Nexus, records it in trustgate/gitops, and Argo CD rolls it out. Reference project: trustgate/tgnra.*

## Activation

When this skill activates, output:

`🚀 Trustgate GitOps CI/CD — loading the tgnra pattern...`

Then execute the protocol below.

## Context Guard

| Context | Status |
|---------|--------|
| **New CI/CD for a Trustgate app (any stack)** | ACTIVE — full protocol |
| **Editing an existing Trustgate `.gitlab-ci.yml` / gitops manifest** | ACTIVE — Steps 1, 4, 6 |
| **A release job failed** | ACTIVE — *When it fails* table |
| **GitHub Actions / personal projects outside gitlab.msctrustgate.com** | DORMANT |
| **tgekyc liveness image push** (`kubectl apply` by hand, no gitops) | DORMANT — use `tgekyc-liveness-deployment-checklist` |

---

## Source of Truth (read these first, every time)

The pattern is maintained by Habib (DevOps) and evolves. **Never write the pipeline from memory — copy it from these files:**

| File | What it is |
|---|---|
| `C:\PROJECTS\DOCKER GITLAB\docker\argocd\README-ci.md` | The official guide: setup steps, **the template**, "What to change" table, day-to-day, failure table |
| `C:\PROJECTS\TGNRA\Development\tgnra\.gitlab-ci.yml` | Reference app, live in production (branch `production`) |
| `C:\PROJECTS\DOCKER GITLAB\docker\argocd\project-tgnra.yaml`, `app-tgnra-prod.yaml` | Argo CD AppProject + Application to copy for a new app |
| `git show 7dca7e9:deploy/prod/deployment.yaml` (in the tgnra repo) | Example prod Deployment: imagePullSecrets, `.env` Secret mount, NFS readiness probe, PVC `Prune=false` (the live copy is in trustgate/gitops `tgnra/prod/`) |

If the README and the tgnra pipeline disagree, the README template wins (it's the generalised version). Mention the difference to Dejul.

---

## The Concept

```
push to release branch (protected, e.g. production)
  → review:<app>   SonarQube scan            (sonar-scanner-cli, runner tag: review)
  → build:<app>    image → Nexus <app>:<VERSION>, captures sha256 digest
                   (moby/buildkit:rootless, daemonless — no Docker-in-Docker; runner tag: build)
  → release:<app>  sed the image line in trustgate/gitops <app>/<env>/*.yaml to
                   localhost:30453/<app>:<VERSION>@sha256:<digest>, commit to master,
                   then poll the Argo CD API (read-only token) until Synced + Healthy (max 10 min)
  → Argo CD        auto-sync (polls every 60s) applies <app>/<env>/ to the cluster
```

**The pipeline never touches the cluster** (no kubectl, no kubeconfig in CI). The app repo gets no commits from CI — only gitops does.

### Why each design choice (explain these to Dejul when asked "kenapa")

| Choice | Reason |
|---|---|
| Pin by **digest**, not just tag | An unbumped `VERSION` re-pushes the same tag; the digest makes every build a distinct, exact deploy (and rollback target) |
| Release **waits** for Argo CD Healthy | A failed rollout turns the pipeline red → GitLab emails the pusher. Without it, CI is green while prod is broken |
| "new-commit" check via `git merge-base --is-ancestor` | Other apps also commit to gitops master; Argo CD may be on a *later* commit that contains ours — that counts |
| Health must be read **after** the last sync finished (`reconciledAt > finishedAt`) | Health read earlier is the old rollout's |
| Push loop with `git pull --rebase` ×3 | All apps release to the same gitops branch; `resource_group` only serialises *this* app |
| Mount guard (`for p in … mountPath`) | A Deployment that lost its volume mounts still starts — and silently loses uploads/config. Refuse to release instead |
| No `.env`/config in the image | Prod mounts it from the `<app>-env` Secret (created by hand). `.dockerignore` must list every tracked `.env*` |
| Argo CD never manages Secrets | AppProject whitelist allows only Deployment/Service/PVC/CronJob; Secrets are created by hand on kvm7 |
| PVC annotation `argocd.argoproj.io/sync-options: Prune=false` | Data must survive even if the PVC vanishes from the repo |
| NFS readinessProbe `grep -q ' <path> nfs' /proc/mounts` | A missing NFS mount on hostPath gives an empty local dir, not an error |
| `retry` only on runner/GitLab failures | A Degraded/timed-out rollout would fail the same way 10 min later and delay the email |
| Protected branch + Protected variables | `GITOPS_TOKEN`/`ARGOCD_AUTH_TOKEN` only reach protected branches |

### Fixed infrastructure values (leave alone)

| Item | Value |
|---|---|
| GitLab | `http://gitlab.msctrustgate.com`, group `trustgate/` |
| Runner tags | `review`, `build` |
| Nexus push | `$NEXUS_REGISTRY` (group var) · in-cluster pull `localhost:30453` (old registry `localhost:30445` lines get rewritten on first release) |
| Gitops repo | `trustgate/gitops`, branch `master`, folder `<app>/<env>/` (or `<app>/` for single-env apps like tgwebsite) |
| Argo CD | `https://10.5.1.42:30219` (self-signed, `curl -k`), namespace `argocd`, account `ci-readonly` (get applications only) |
| Group/instance variables | `NEXUS_REGISTRY/USER/PASSWORD`, `SONAR_HOST_URL/TOKEN`, `GITOPS_TOKEN` (group), `ARGOCD_AUTH_TOKEN` (instance) — inherited by every `trustgate/` project |
| Cluster admin host | kvm7 (Secrets, `kubectl apply` of Argo CD app/project) |

---

## Protocol

### Step 1: Read the source of truth
- [ ] Read `README-ci.md` in full and the current tgnra `.gitlab-ci.yml`
- [ ] Note anything changed since this skill was written (2026-11-10) — tell Dejul

### Step 2: Inspect the target project
- [ ] Stack (Laravel/PHP, CodeIgniter, Java/Maven WAR, Spring Boot, Node...)
- [ ] Is it a git repo with a `trustgate/` GitLab remote? (personal project → needs its own `GITOPS_TOKEN`, see README *Tokens*)
- [ ] Existing Dockerfile? Existing K8s manifests (`C:\PROJECTS\DOCKER GITLAB\docker\<app>\`)? Already running in the cluster?
- [ ] Every runtime mount: config files, uploads, NFS paths → these become the mount-guard list and Secrets
- [ ] Hardcoded secrets in source/tests → must be removed before the first push (Sonar + repo history)

### Step 3: App repo files (Jessy writes these)
- [ ] `Dockerfile` at repo root, builds from `.` — **no `.env`/config/credentials baked in**
- [ ] `.dockerignore` — `.git`, `.gitlab-ci.yml`, `.sonar`, `build.env`, every `.env*` the repo tracks (`git ls-files '.env*'`), build output
- [ ] `VERSION` — one line, e.g. `1.00`
- [ ] `.gitlab-ci.yml` — copy **the template from README-ci.md**, then apply its *What to change* table: `PIPELINE_BRANCH`, `<app>` in job names + `needs` + both `IMAGE_REPO`s, `SONAR_PROJECT_KEY`, `-Dsonar.exclusions`, `GITOPS_PATH`, `ARGOCD_APP`, the `for p in …` mount list, `environment: name`, `resource_group`
- [ ] Anything containing `": "` inside script goes in a `- |` block
- [ ] Stack-specific additions (see *Adapting to other stacks*)

### Step 4: Gitops + cluster side (hand to Dejul / Habib — Jessy prepares, never applies)
- [ ] `<app>/<env>/deployment.yaml` (+ Service, PVC, CronJob) for trustgate/gitops, modelled on tgnra prod: `imagePullSecrets: [{name: docker-registry-secret}]`, config/.env from `<app>-env` Secret, PVC `Prune=false`, NFS readinessProbe
- [ ] App already running → **two gitops commits**: (1) live manifests unchanged + PVC annotation, check `kubectl diff` shows only that; (2) after Argo CD adopts it: imagePullSecrets + Secret mount
- [ ] Secrets on kvm7: `<app>-env` (from `docker/<app>/<env>/.env`) and `docker-registry-secret` (copy from tgnra namespace — command in README step 2)
- [ ] `project-<app>.yaml` + `app-<app>-<env>.yaml` from the tgnra copies; trim `namespaceResourceWhitelist` to the kinds actually used; `kubectl apply` on kvm7; wait Synced + Healthy (`kubectl -n argocd get applications.argoproj.io <app>-<env>`)

### Step 5: GitLab settings (Dejul)
- [ ] Protect the release branch: push + merge = Developers + Maintainers, force push off
- [ ] Project under `trustgate/` → tokens inherited, nothing to add

### Step 6: First release
- [ ] Push `.gitlab-ci.yml` to the release branch — **this push releases**
- [ ] Prefer pilot env first if the app has one
- [ ] Watch: review → build (digest printed) → release (state lines until `is running …`)
- [ ] On failure: README *When it fails* table

### Step 7: Confirm
- [ ] Report: pipeline URL/state, image `<app>:<VERSION>@sha256:…`, gitops commit, Argo CD Healthy
- [ ] Add rollback note: `git revert` the release commit in trustgate/gitops (old digest must still be in Nexus)

---

## Adapting to other stacks

The template is written for Laravel. The **build** and **release** jobs are stack-independent (they only need a root `Dockerfile` and `VERSION`). What changes per stack:

| Stack | Changes |
|---|---|
| Laravel / PHP | None — template as-is |
| CodeIgniter | Sonar exclusions + `system/**,application/third_party/**,application/cache/**,application/logs/**` |
| **Java / Maven (WAR on WildFly/Tomcat)** | (1) add a `test` stage before `review`: `maven:3.9-eclipse-temurin-<jdk>` → `mvn -B clean verify`, keep `target/classes`, `target/site/jacoco/jacoco.xml` and surefire XML as artifacts (`reports: junit: target/surefire-reports/TEST-*.xml`); (2) Sonar needs `-Dsonar.java.binaries=target/classes -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml`, `-Dsonar.sources=src/java` (or `src/main/java`), `-Dsonar.tests=src/test/java`, and `needs: [test]`; (3) multi-stage Dockerfile: Maven stage builds the WAR, runtime stage copies it into the app server's deployments dir — so the buildkit job stays unchanged; (4) config dir (e.g. `/opt/mtsa` with `.properties` + `wscredentials.xml`) comes from a Secret or NFS mount, never the image; put that path in the mount guard |
| Spring Boot jar | As Java, runtime stage `eclipse-temurin:<jdk>-jre` + `java -jar` |

**Unverified for Java until the first real pipeline runs** — tgnra/tgwebsite (PHP) are the only proven apps as of 2026-11-10. Mark anything Java-specific as a proposal and confirm with Habib if the runner lacks Maven Central access (may need Nexus as a Maven mirror via `settings.xml`).

---

## Day to day (short — details in README)

- **Deploy**: push to the release branch. Unchanged `VERSION` still deploys (new digest) — bump it anyway for a readable tag.
- **Rollback**: `git revert` the release commit in trustgate/gitops, push.
- **Edit gitops by hand**: always `git pull --rebase origin master` first (releases commit there any time).
- **Change .env/config**: edit master copy `docker/<app>/<env>/.env` → recreate Secret with `--dry-run=client -o yaml | kubectl apply -f -` → `kubectl rollout restart deploy/<app>`.
- **kubectl on Argo apps**: `applications.argoproj.io`, not `app` (Rancher steals the short name).
- **selfHeal is on**: manual `kubectl edit` of managed objects gets reverted — change gitops instead.

## When it fails (most common)

| Symptom | Cause |
|---|---|
| `GITOPS_TOKEN is missing` / `ARGOCD_AUTH_TOKEN is missing` | Release branch not protected, personal project, or variable not Protected |
| `HTTP Basic: Access denied` | `GITOPS_TOKEN` expired/revoked (one token for all projects) |
| `no image line for <app> found` | `IMAGE_REPO` mismatch or line doesn't start `image: localhost:` |
| `has no volumeMount for …, not releasing` | Mount removed from gitops deployment.yaml |
| `Argo CD answered HTTP 401/403/404` | Token expired or `ARGOCD_APP` misspelt |
| timeout `old-commit …` | Argo never synced: auto-sync off, wrong `GITOPS_PATH`, sync error |
| timeout `new-commit Synced Progressing …` | Pod not Ready: crash loop, NFS probe, image pull |
| `ImagePullBackOff` 401 | `docker-registry-secret` missing / not in `imagePullSecrets` |

---

## Mandatory Rules

1. **Copy the template from `README-ci.md`**, never retype the pipeline from memory — the Argo CD wait logic is subtle and was tuned against real failures.
2. **Never run `kubectl apply`, create Secrets, or change GitLab/Argo CD settings from this session** — prepare the files and commands, Dejul/Habib run them on kvm7 (same rule as the other Trustgate deploy skills).
3. **No secrets in the repo, image, chat, or pipeline logs** — tokens live only in GitLab Protected + Masked variables; config/.env only in K8s Secrets. Scan the project for hardcoded credentials before the first push.
4. **The first push of `.gitlab-ci.yml` to the release branch deploys** — say so explicitly and get Dejul's go before pushing (git push only on his "save"/explicit instruction anyway).
5. **Every mounted path goes in the mount guard** — if a path is mounted in the Deployment, it must be in `for p in …`.
6. **Never claim a release is done** until the release job printed `<app> is running <image>` (verification-before-completion).

---

## Skill Level

**Lv.1** (2026-11-10) — extracted from trustgate/tgnra (`production`, image 1.06) + `docker/argocd/README-ci.md`; first target: MTSA KASTAM (Java 8 / Maven WAR / WildFly 18).
