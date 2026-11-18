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
*How Trustgate apps are built and deployed: GitLab CI pushes an image to Nexus, records it in trustgate/gitops, and Argo CD rolls it out. Reference project: trustgate/tgnra. Source of truth for the job YAML: `argocd/README-ci.md` in trustgate/docker on origin/master — `git fetch` and read `git show origin/master:argocd/README-ci.md` before writing or reviewing a pipeline (it changed 2026-11-12: tag `<VERSION>-<short sha>` + Nexus tag cleanup).*

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
  → build:<app>    image → Nexus <app>:<VERSION>-<CI_COMMIT_SHORT_SHA> (e.g. 1.02-0a228a0d), captures sha256 digest
                   (moby/buildkit:rootless, daemonless — no Docker-in-Docker; runner tag: build)
  → release:<app>  sed the image line in trustgate/gitops <app>/<env>/*.yaml to
                   localhost:30453/<app>:<VERSION>-<sha>@sha256:<digest>, commit to master,
                   then poll the Argo CD API (read-only token) until Synced + Healthy (max 10 min),
                   then delete all but the newest 3 tags of <app> from Nexus (NEXUS_URL http://10.5.1.42:30452, repo docker-hosted)
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
- [ ] Add rollback note: `git revert` the release commit in trustgate/gitops (old digest must still be in Nexus; the release keeps only the newest 3 tags, so rollback reaches 2 releases back)

---

## Adapting to other stacks

The template is written for Laravel. The **build** and **release** jobs are stack-independent (they only need a root `Dockerfile` and `VERSION`). What changes per stack:

| Stack | Changes |
|---|---|
| Laravel / PHP | None — template as-is |
| CodeIgniter | Sonar exclusions + `system/**,application/third_party/**,application/cache/**,application/logs/**` |
| **Java / Maven (WAR on WildFly/Tomcat)** | (1) add a `test` stage before `review`: `maven:3.9-eclipse-temurin-<jdk>` → `mvn -B clean verify`, keep `target/classes`, `target/site/jacoco/jacoco.xml` and surefire XML as artifacts (`reports: junit: target/surefire-reports/TEST-*.xml`); (2) Sonar needs `-Dsonar.java.binaries=target/classes -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml`, `-Dsonar.sources=src/java` (or `src/main/java`), `-Dsonar.tests=src/test/java`, and `needs: [test]`; (3) multi-stage Dockerfile: Maven stage builds the WAR, runtime stage copies it into the app server's deployments dir — so the buildkit job stays unchanged; (4) config dir (e.g. `/opt/mtsa` with `.properties` + `wscredentials.xml`) comes from a Secret or NFS mount, never the image; put that path in the mount guard |
| Spring Boot jar | As Java, runtime stage `eclipse-temurin:<jdk>-jre` + `java -jar` |

**Java proven 2026-11-13:** MTSA KASTAM (`trustgate/custom`) pipeline #550 passed all of test (296), review, build and release on pilot. The Maven Central downloads worked from the runner. Mark anything Java-specific as a proposal and confirm with Habib if the runner lacks Maven Central access (may need Nexus as a Maven mirror via `settings.xml`).

---

## Day to day (short — details in README)

- **Deploy**: push to the release branch. Unchanged `VERSION` still deploys (new digest) — bump it anyway for a readable tag.
- **Rollback**: `git revert` the release commit in trustgate/gitops, push. Full playbook in *Rollback & image tags* below.
- **Edit gitops by hand**: always `git pull --rebase origin master` first (releases commit there any time).
- **Change .env/config**: edit master copy `docker/<app>/<env>/.env` → recreate Secret with `--dry-run=client -o yaml | kubectl apply -f -` → `kubectl rollout restart deploy/<app>`.
- **kubectl on Argo apps**: `applications.argoproj.io`, not `app` (Rancher steals the short name).
- **selfHeal is on**: manual `kubectl edit` of managed objects gets reverted — change gitops instead.

## Rollback & image tags (answers Dejul asked for, 2026-11-13..17, MTSA KASTAM)

Explain these in Malay with the real example. Example from KASTAM pipeline #550:

```
localhost:30453/custom:1.07-26b046f4@sha256:442df0ad…
└─registry────┘ └repo┘ └ver┘ └commit┘ └image digest┘
                       └────tag────┘
```

| Part | Source | Meaning |
|---|---|---|
| `1.07` | `VERSION` file in the app repo | Human label. It changes only when Dejul edits the file. |
| `26b046f4` | `$CI_COMMIT_SHORT_SHA` | The exact app commit the image was built from (`git show 26b046f4`). |
| tag `1.07-26b046f4` | version + sha | One tag per push, so builds never overwrite each other. |
| `@sha256:…` | Content hash computed at build time | What Kubernetes actually pulls (the tag is ignored once a digest is present). It can't be faked and changes with any byte. |

**Every push to the release branch deploys**, whatever `VERSION` says. This includes a docs-only push: it gets a new tag and usually a new digest (`yum update` and timestamps change the image), the pod restarts, and one of the 3 kept Nexus tags is used up. If wanted, offer `rules: changes:` to skip `*.md`/`deploy/` pushes.

**Dejul never edits the `image:` line by hand.** The release job clones gitops, seds the line, commits and pushes. Example: gitops `da08b43` was made by `release:custom-pilot`, not by Dejul. Dejul edits gitops by hand only for:
- a rollback
- non-image changes: env, probes, mounts, resources

Run `git pull --rebase origin master` before either.

**Rollback playbook:**

1. **Restore the running app (gitops only, no app code touched).**

   ```
   cd "C:\PROJECTS\DOCKER GITLAB\gitops"; git pull --rebase origin master
   git log --oneline -5 -- <app>/<env>      # path filter: only this app's commits
   git show --stat <bad-release-sha>        # must list ONLY <app>/<env>/deployment.yaml
   git revert <bad-release-sha> --no-edit; git push origin master
   ```

   - Argo CD syncs within about 1 minute. Only the app whose folder changed rolls out; other apps in the shared gitops repo stay untouched.
   - If a commit touched several apps (manual commits), don't revert it. Instead: `git checkout <good-sha> -- <app>/<env>/deployment.yaml`, commit, push.
   - Check `git diff HEAD~1 --stat` before pushing.
   - Make sure no pipeline is still running on the release branch first, or its release overwrites the revert.

2. **Stop the bug from coming back (app repo).** The release branch still holds the bad code, and the next push redeploys it. Either fix forward (bump VERSION), or `git revert` in the app repo before pushing again.

3. **When the old image is gone from Nexus.** Cleanup keeps the newest 3 tags of the repo across ALL versions. Example: pushing 1.04 three times (`1.04-b1..b3`) deletes every `1.03-*` tag.
   - A gitops revert then gives `ImagePullBackOff`. The old pod keeps running, so the app is not down.
   - Roll back in the app repo instead:

     ```
     git revert --no-commit <last-good-sha>..HEAD
     git commit -m "Revert to 1.03"
     git push
     ```

     This rebuilds as `1.03-<new sha>`. Later, revert that revert to resume work on 1.04.
   - Rolling back to an earlier attempt of the same version (b3 bad, b2 good) still works via gitops, within the last 3 tags.

Don't use these to roll back:
- `kubectl set image` or `kubectl edit`: selfHeal reverts them within seconds.
- Argo CD UI "Rollback": not allowed while auto-sync is on.

**Advice for iterating many times:**
- Work on a feature branch. The pipeline runs only on the release branch, so branch pushes don't deploy.
- Test locally (`mvn verify`, `docker build`/run), then merge.
- Proposed but not adopted, ask Habib: cleanup keeps the newest 3 **plus the newest tag of the previous VERSION**.

**Before adding a prod release job when pilot and prod share one image repo** (KASTAM: `custom`): pilot pushes would make the cleanup delete the image prod is running. The running pod survives, but a reschedule to another node fails the pull. This must be solved first: protect the image that prod is running, or use separate repos.

**Unit-test log noise:** `ERROR` lines and stack traces in `test:<app>` come from negative tests that deliberately feed bad input. Judge only by `Tests run: N, Failures: 0, Errors: 0` and `BUILD SUCCESS`.

**Coverage (JaCoCo):**
- It is the share of code lines executed by unit tests. It shows untested logic and feeds Sonar Quality Gates.
- Where to look:
  - SonarQube project → Coverage. This is the main view, with red/green lines per file.
  - GitLab pipeline → Tests tab, for JUnit results.
  - The job artifact `jacoco.xml`: raw, expires after 1 day.
  - Locally: `mvn verify`, then `target/site/jacoco/index.html`.
- Watch the trend and coverage on new code, not 100%.

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
**Lv.2** (2026-11-17) — added the first Java pipeline (KASTAM #550, green) and the rollback & image-tag playbook from Dejul's questions.
