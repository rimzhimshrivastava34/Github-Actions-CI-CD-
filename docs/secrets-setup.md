# Secrets & Setup Guide

All sensitive values are stored as Kubernetes Secrets or GitHub Actions Secrets —
never hardcoded in YAML.

---

## GitHub Actions Secrets

Set these under **Settings → Secrets and variables → Actions**:

| Secret | Description |
|--------|-------------|
| `KUBECONFIG_B64` | Base64-encoded kubeconfig for your cluster |
| `SLACK_WEBHOOK_URL` | Slack incoming webhook (optional) |

**Generate the kubeconfig secret:**
```bash
cat ~/.kube/config | base64 -w 0
# Paste the output as KUBECONFIG_B64
```

---

## Kubernetes Secrets

Create these once in your cluster before running any workflow:

### Git SSH key (for private repo clone)
```bash
kubectl create secret generic git-ssh-secret \
  --from-file=id_rsa=~/.ssh/your-deploy-key \
  -n argo
```

### Docker registry credentials
```bash
kubectl create secret generic docker-config-secret \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson \
  -n argo
```

### GitHub token (for opening PRs in GitOps repo)
```bash
kubectl create secret generic github-token-secret \
  --from-literal=token=ghp_your_token_here \
  -n argo
```

### Notification webhooks
```bash
kubectl create secret generic notification-secrets \
  --from-literal=teams-webhook-url="https://your-org.webhook.office.com/..." \
  --from-literal=slack-webhook-url="https://hooks.slack.com/services/..." \
  -n argo
```

---

## Argo Service Account

The workflow runs as `argo-workflow-sa`. Create it with minimum required permissions:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argo-workflow-sa
  namespace: argo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argo-workflow-role
  namespace: argo
rules:
  - apiGroups: ["argoproj.io"]
    resources: ["workflows", "workflowtemplates"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argo-workflow-rolebinding
  namespace: argo
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: argo-workflow-role
subjects:
  - kind: ServiceAccount
    name: argo-workflow-sa
    namespace: argo
EOF
```

---

## Deploy the WorkflowTemplate

```bash
kubectl apply -f argo-workflows/build-and-deploy.yaml -n argo

# Verify it was created
argo template list -n argo
```

---

## Manual trigger (without GitHub Actions)

```bash
argo submit \
  --from workflowtemplate/app-build-and-deploy \
  --namespace argo \
  --parameter BRANCH=main \
  --parameter SERVICE_NAME=your-app \
  --watch
```