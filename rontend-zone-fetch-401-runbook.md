````markdown
# Runbook: Failed to Fetch Zone of the Node Where the Pod is Scheduled (`compute: Received 401`)

# Service Infrastructure
# - **Tech stack**: EJS, Express
# - **Grafana Loki data source**: `loki|grafanacloud-logs|grafanacloud-martindstone-logs`
# - **Key labels**: `service_name="frontend"`

## Links
# - [Service Dashboard](https://jeffhudsonbush03.grafana.net/d/jerwfcd/frontend)
# - [Application](https://jeffhudsonbush03.grafana.net/a/grafana-app-observability-app/services/service/boutique---frontend)
# - [GitHub Repository](https://github.com/PagerDuty/orbitpay-web-2)
# - [Architecture Docs](https://github.com/PagerDuty/orbitpay-web-2/blob/main/AGENTS.md)

## Incident Workflows
- **OrbitPay — Rollback Web Frontend**: Rolls back the web frontend to the previous version

## Overview

This runbook helps troubleshoot the following Kubernetes/GKE error for the `frontend` pod in the `boutique` namespace:

```text
Failed to fetch the Zone of the node where the pod is scheduled
compute: Received 401
```

This issue commonly occurs when Kubernetes components or workloads cannot authenticate with the cloud provider API (typically Google Cloud Compute Engine API).

---

# Environment

- Namespace: `boutique`
- Deployment: `frontend`
- Platform: Kubernetes / GKE
- Error:
  ```text
  compute: Received 401
  ```

---

# Symptoms

- Frontend pod fails health checks
- Pod stuck in `CrashLoopBackOff`
- Pod logs show authentication failures
- Monitoring/metadata calls fail
- Kubernetes events contain:
  ```text
  Failed to fetch the Zone of the node where the pod is scheduled
  compute: Received 401
  ```

---

# Root Cause

A `401 Unauthorized` response indicates authentication failure when attempting to access Google Compute Engine metadata or APIs.

Common causes:

- Missing or invalid Workload Identity configuration
- Expired node credentials
- Missing IAM permissions
- Metadata server access blocked
- Incorrect service account binding
- GKE node service account disabled

---

# Troubleshooting Steps

## 1. Verify Frontend Pods

```bash
kubectl get pods -n boutique
```

Check pod status:

```bash
kubectl describe pod <frontend-pod-name> -n boutique
```

---

## 2. Check Pod Logs

```bash
kubectl logs <frontend-pod-name> -n boutique
```

Look for:

```text
401 Unauthorized
metadata server unavailable
permission denied
```

---

## 3. Verify Node Assignment

Check which node the pod is running on:

```bash
kubectl get pod <frontend-pod-name> -n boutique -o wide
```

Example output:

```text
frontend-abc123   Running   worker-node-1
```

---

## 4. Verify Node Zone Labels

```bash
kubectl get nodes --show-labels
```

Check for labels like:

```text
topology.kubernetes.io/zone
failure-domain.beta.kubernetes.io/zone
```

---

## 5. Test Metadata Server Access

SSH into the node or use a debug pod:

```bash
kubectl run debug-shell \
  --rm -it \
  --image=busybox \
  --restart=Never \
  -n boutique -- sh
```

Test metadata access:

```bash
wget -qO- \
http://metadata.google.internal/computeMetadata/v1/instance/zone \
--header="Metadata-Flavor: Google"
```

Expected result:

```text
projects/<project-id>/zones/us-central1-a
```

If you receive `401`, authentication is failing.

---

# Resolution Steps

## Option 1: Restart Frontend Deployment

Sometimes credentials refresh automatically after restart.

```bash
kubectl rollout restart deployment frontend -n boutique
```

Verify:

```bash
kubectl get pods -n boutique
```

---

## Option 2: Scale Deployment Down and Up

```bash
kubectl scale deployment frontend --replicas=0 -n boutique
```

Wait for pods to terminate.

Bring back replicas:

```bash
kubectl scale deployment frontend --replicas=1 -n boutique
```

---

## Option 3: Verify Workload Identity

Check Kubernetes service account:

```bash
kubectl get sa -n boutique
```

Describe service account:

```bash
kubectl describe sa frontend -n boutique
```

Verify annotation:

```text
iam.gke.io/gcp-service-account
```

---

## Option 4: Verify IAM Permissions

Ensure the Google service account has required roles:

- Compute Viewer
- Kubernetes Engine Developer
- Monitoring Viewer (if applicable)

Example:

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:GSA_NAME@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/compute.viewer"
```

---

## Option 5: Restart Node Pool

If node credentials are stale:

```bash
gcloud container clusters upgrade CLUSTER_NAME \
  --node-pool NODE_POOL \
  --cluster CLUSTER_NAME
```

Or recreate affected nodes.

---

# Validation

After remediation:

```bash
kubectl get pods -n boutique
```

Check frontend pod logs:

```bash
kubectl logs <frontend-pod-name> -n boutique
```

Verify no more `401` errors.

---

# Useful Commands

## Get frontend pods

```bash
kubectl get pods -n boutique -l app=frontend
```

## Describe deployment

```bash
kubectl describe deployment frontend -n boutique
```

## Check events

```bash
kubectl get events -n boutique --sort-by=.metadata.creationTimestamp
```

## Watch pod status

```bash
kubectl get pods -n boutique -w
```

---

# Escalation

Escalate to the platform/cloud team if:

- IAM permissions appear correct
- Metadata server remains inaccessible
- Multiple namespaces are impacted
- Node pools continuously fail authentication

---

# References

- Kubernetes Documentation
- Google Kubernetes Engine (GKE) Workload Identity
- Google Compute Engine Metadata Server Documentation
````
