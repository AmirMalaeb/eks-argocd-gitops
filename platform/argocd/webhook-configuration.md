# Git Webhook Configuration for Argo CD

## Overview

By default, Argo CD polls the GitOps repository every **3 minutes** to detect changes.
For near-instant sync detection, configure a GitHub webhook to notify Argo CD on push events.

## Default Polling (No Setup Required)

Argo CD's default polling interval is 3 minutes. This satisfies Requirement 6.4 out of the box.
The interval is controlled by the `timeout.reconciliation` setting in `argocd-cm`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  timeout.reconciliation: 180s   # 3 minutes (default)
```

## GitHub Webhook Setup (Optional — Near-Instant Sync)

### 1. Create a webhook secret

```bash
kubectl -n argocd create secret generic argocd-webhook-secret \
  --from-literal=webhook.github.secret=<your-random-secret>
```

### 2. Configure the webhook in GitHub

1. Go to **Settings → Webhooks → Add webhook** in the GitOps repository.
2. Set the **Payload URL** to:
   ```
   https://<argocd-server-url>/api/webhook
   ```
3. Set **Content type** to `application/json`.
4. Set the **Secret** to the same value used in step 1.
5. Select **Just the push event**.
6. Ensure the webhook is **Active**.

### 3. Verify connectivity

After pushing a commit, check the webhook delivery log in GitHub (Settings → Webhooks → Recent Deliveries).
A `200 OK` response confirms Argo CD received the event.

## Sync Timeout for Health Check Evaluation

The sync operation timeout (used for health check evaluation) is configured at the Argo CD
controller level. For this POC, the default timeout of **5 minutes** is used:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  timeout.reconciliation: 180s
  # Sync operation timeout is controlled by the controller flag:
  # --operation-timeout=300  (5 minutes, default)
```

This means Argo CD will wait up to 5 minutes for deployed resources to become healthy
before marking the sync as degraded.
