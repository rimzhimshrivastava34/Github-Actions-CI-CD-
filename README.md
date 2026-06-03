# GitHub Actions + Argo Workflows — CI/CD Pipeline

> A production-grade CI/CD setup where **GitHub Actions handles the trigger & test layer**
> and **Argo Workflows handles the build, push, and GitOps deploy** on Kubernetes.

[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat-square&logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![Argo Workflows](https://img.shields.io/badge/Argo_Workflows-EF7B4D?style=flat-square&logo=argo&logoColor=white)](https://argoproj.github.io/workflows)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docker.com)

---

## How It Works

```
Developer pushes to GitHub
        │
        ▼
GitHub Actions (CI layer)
  ├── Lint & test code
  ├── Submit Argo Workflow via CLI
  └── Wait & report result
        │
        ▼
Argo Workflows (CD layer, runs on K8s)
  ├── Step 1: git-clone        → clones repo via SSH
  ├── Step 2: generate-tag     → creates branch-YYYYMMDD-HHMMSS tag
  ├── Step 3: build-image      → docker build + push to registry
  ├── Step 4: update-values    → updates Helm values, opens GitOps PR
  └── onExit: notify           → Teams/Slack alert on any outcome
```

**Why split the pipeline this way?**

GitHub Actions is great for lightweight triggers and repo-level CI (lint, test, scan).
Argo Workflows runs the heavy K8s work — Docker builds with shared PVCs, parallel steps,
retry logic, real-time DAG UI, and full audit logs. Each tool does what it's best at.

---

## Repo Structure

```
.
├── .github/
│   └── workflows/
│       └── trigger-argo-deploy.yml   # GitHub Actions: test + trigger Argo
├── argo-workflows/
│   └── build-and-deploy.yaml         # Argo WorkflowTemplate with all steps
├── docs/
│   └── secrets-setup.md              # How to create all required secrets
└── README.md
```

---

## Pipeline Steps (Argo side)

| Step | Template | What it does |
|------|----------|--------------|
| 1 | `git-clone` | Clones private repo via SSH deploy key |
| 2 | `generate-tag` | Builds image tag: `branch-YYYYMMDD-HHMMSS` |
| 3 | `build-image` | Docker multi-stage build + push to registry |
| 4 | `update-values-and-pr` | Updates Helm values in GitOps repo, opens PR |
| onExit | `notify-teams` | Posts to Teams/Slack regardless of outcome |
| onExit | `update-release-tracker` | Updates internal release log |

---

## Key Patterns Used

**Tag generation from clone timestamp**
The image tag is derived from when the git clone step started — not the current time.
This makes tags deterministic and traceable back to the exact workflow run.

**sprig.trimPrefix for branch names**
Webhook systems send `refs/heads/main` instead of `main`. The expression
`{{=sprig.trimPrefix('refs/heads/', inputs.parameters.BRANCH)}}` strips this
automatically so the tag and PR branch are always clean.

**Shared PVC as workspace**
All steps share a single `ReadWriteOnce` PVC (`workdir`). The cloned source code
written in step 1 is available to the Docker build in step 3 — no re-cloning needed.

**onExit handler**
`onExit: send-notification-onexit` runs even if the workflow is manually stopped,
times out, or hits an error. The team is always notified.

**Secrets never in YAML**
All credentials (SSH key, Docker config, GitHub token, webhook URLs) are mounted
from Kubernetes Secrets — never hardcoded.

---

## Quick Start

### 1. Apply the WorkflowTemplate to your cluster

```bash
kubectl apply -f argo-workflows/build-and-deploy.yaml -n argo
```

### 2. Create required secrets

See [docs/secrets-setup.md](docs/secrets-setup.md) for all `kubectl create secret` commands.

### 3. Set GitHub Actions secrets

| Secret | Value |
|--------|-------|
| `KUBECONFIG_B64` | `cat ~/.kube/config \| base64 -w 0` |
| `SLACK_WEBHOOK_URL` | Your Slack webhook (optional) |

### 4. Push to a deploy branch

```bash
git push origin main
# GitHub Actions triggers → Argo Workflow runs → PR opened in GitOps repo
```

### 5. Manual trigger via Argo CLI

```bash
argo submit \
  --from workflowtemplate/app-build-and-deploy \
  --namespace argo \
  --parameter BRANCH=main \
  --parameter SERVICE_NAME=your-app \
  --watch
```

---

## Customising for Your Project

| What to change | Where |
|----------------|-------|
| Branches allowed | `arguments.parameters.BRANCH.enum` in `build-and-deploy.yaml` |
| Docker registry | `IMAGE_NAME` parameter + `docker-config-secret` |
| GitOps repo | `GITOPS_REPO_URL` env var in `update-values-and-pr` template |
| Helm values key | `TAG_KEY` parameter (default: `tag`) |
| Notification target | Uncomment Teams or Slack block in `notify-teams` template |
| Storage size | `volumeClaimTemplates[0].spec.resources.requests.storage` |

---

## Freelance Availability

I design and implement CI/CD pipelines like this one for product teams.

- GitHub Actions + Argo Workflows setup from scratch
- GitOps with ArgoCD — Helm-based app deployment
- Docker build optimisation (multi-stage, layer caching)
- Kubernetes-native workflow automation

📬 [LinkedIn](https://linkedin.com/in/rimzhim-shrivastava) · [Upwork](https://upwork.com) · rimzhimshrivastava2@gmail.com