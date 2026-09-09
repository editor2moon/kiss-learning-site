# Runbook: SAS CAS Server — OOMKilled / CrashLoopBackOff Recovery

---

## Metadata

| Field | Value |
|-------|-------|
| **Runbook ID** | RB-001 |
| **Title** | SAS CAS Server — OOMKilled / CrashLoopBackOff Recovery |
| **Environment** | Production |
| **Service** | SAS Viya — CAS (Cloud Analytics Services) |
| **Trigger** | CloudWatch Alarm `sas-viya-cas-pod-restarts-high` OR user reports "CAS is unavailable" |
| **Severity** | P1 |
| **Owner** | CloudOps |
| **SAS Admin required?** | Yes — notify after step R3 so they can monitor user sessions |
| **Estimated duration** | 15–25 minutes |
| **Last tested** | 2026-08-15 |
| **Last updated** | 2026-09-09 |
| **Version** | v1.1 |

---

## BLUF

> CAS (Cloud Analytics Services) is the in-memory analytics engine for SAS Viya.
> When it crashes (OOMKilled or CrashLoopBackOff), all active user sessions are lost and no new SAS
> computations can start. DAMS API calls and Tableau connections to CAS will also fail.
> This runbook diagnoses the crash cause and restores CAS to a healthy running state.
> Estimated recovery time: 15–25 min. Data in-flight at the time of crash is not recoverable.

---

## Prerequisites

Before starting, confirm the following:

- [ ] `kubectl` configured for production cluster: `kubectl config current-context` returns `sas-viya-prod`
- [ ] AWS CLI access to production account: `aws sts get-caller-identity` shows account `111122223333`
- [ ] CloudWatch log access for log group `/aws/eks/sas-viya-prod/sas-viya`
- [ ] You have notified the SAS Admin (client) that a P1 incident is in progress

**Required access level:** EKS read-write on namespace `sas-viya`

---

## Symptoms / When to Use This Runbook

- CloudWatch alarm `sas-viya-cas-pod-restarts-high` fired (threshold: >3 restarts in 10 minutes)
- Users report: "I get an error when trying to run a SAS Studio job" or "CAS connection failed"
- DAMS application returns HTTP 502/503 when calling SAS Viya REST API
- Tableau dashboards show "CAS server unavailable" or fail to refresh
- `kubectl get pods -n sas-viya` shows `sas-cas-server-*` in `CrashLoopBackOff` or `OOMKilled`

---

## Impact Assessment

| Aspect | Detail |
|--------|--------|
| **User impact** | All SAS Viya users cannot run computations or start new CAS sessions. SAS Studio is partially functional (code editing works; execution fails). |
| **Data risk** | Any in-progress CAS jobs at the time of crash lose their in-memory data. Results not written to CASLIB are lost. Persisted data on EFS and RDS is not affected. |
| **Downstream impact** | DAMS REST API calls to SAS Viya fail. Tableau/Alteryx JDBC connections to CAS fail. |
| **Safe to proceed alone?** | Yes for diagnosis and restart. Notify SAS Admin (client) before step R3. |

---

## Go / No-Go Checklist

- [ ] Confirmed this is the production cluster, not DR or dev
- [ ] SAS Admin (client) notified — they will monitor user-facing impact
- [ ] No scheduled maintenance window in progress (check calendar / ServiceNow)
- [ ] EFS volume for CAS is still mounted (checked in Step D3 below)

---

## Diagnosis Steps

### Step D1 — Check pod status in the SAS Viya namespace

```bash
kubectl get pods -n sas-viya | grep cas
```

**Expected output (healthy):**
```
sas-cas-server-default-controller-0    2/2     Running   0          5d
sas-cas-server-default-worker-0        2/2     Running   0          5d
sas-cas-server-default-worker-1        2/2     Running   0          5d
```

**Abnormal output confirming the problem:**
```
sas-cas-server-default-controller-0    0/2     OOMKilled          3          18m
sas-cas-server-default-controller-0    0/2     CrashLoopBackOff   6          18m
```

Note the restart count. If restarts > 10, Kubernetes backoff delay will slow recovery — consider `kubectl delete pod` (covered in R2) to reset the backoff timer.

---

### Step D2 — Get crash details from the pod

```bash
kubectl describe pod sas-cas-server-default-controller-0 -n sas-viya
```

**What to look for in `Events` and `Last State`:**

- `OOMKilled` in `Reason` field → CAS ran out of memory. Check `Limits` vs actual usage.
- `Exit Code: 137` → OOM kill confirmed by kernel.
- `Exit Code: 1` → application-level crash. Check logs next.
- Node name in `Node:` field — note it, you will check the node in D4.

```bash
kubectl logs sas-cas-server-default-controller-0 -n sas-viya --previous --tail=100
```

**What to look for in logs:**
- `ERROR: CAS server terminated due to memory limit` → OOM cause confirmed
- `FATAL: Could not connect to sas-data-server-postgres` → PostgreSQL dependency failure
- `ERROR: Failed to mount /cas/permstore` → EFS storage issue
- `License: No valid license found` → SAS license issue (less common)

---

### Step D3 — Check EFS storage (CAS permstore and data)

CAS requires two PVCs: `cas-permstore` (persistent configuration) and `cas-data` (data connector cache).

```bash
kubectl get pvc -n sas-viya | grep cas
```

Expected: both PVCs in `Bound` state.

```bash
kubectl get pvc cas-permstore-sas-cas-server-default-controller-0 -n sas-viya -o yaml | grep -A5 "status:"
```

If any PVC is in `Pending` or `Lost` state — **stop here and escalate to EFS/storage investigation before proceeding.**

---

### Step D4 — Check PostgreSQL connectivity

SAS Viya CAS depends on the SAS Infrastructure Data Server (PostgreSQL).

```bash
kubectl run pg-test --rm -it --restart=Never \
  --image=postgres:15 -n sas-viya \
  -- psql -h sas-viya-postgres.cluster-xxxx.eu-west-1.rds.amazonaws.com \
          -U sas -d SharedServices -c "SELECT 1"
```

Expected: `?column? = 1` returned. If this fails — the root cause is PostgreSQL, not CAS itself.
Check RDS instance status in AWS Console before proceeding with CAS restart.

---

### Step D5 — Check node health where CAS was running

```bash
kubectl get node [node-name-from-D2]
kubectl describe node [node-name-from-D2] | grep -A10 "Conditions:"
kubectl describe node [node-name-from-D2] | grep -A20 "Allocated resources:"
```

Check:
- Node status is `Ready`
- `MemoryPressure` is `False`
- Allocatable memory vs requested — if the node is memory-saturated, CAS will OOMKill again after restart

If node is under memory pressure → Karpenter will provision a new node. You may need to delete the pod (R2) to trigger rescheduling.

---

## Remediation Steps

> Execute in order. Validate each step before proceeding.

### Step R1 — Confirm no active DR or maintenance activity

```bash
# Check if Argo CD is mid-sync (could cause a concurrent restart conflict)
argocd app get sas-viya --grpc-web | grep -E "Sync Status|Health Status"
```

If Argo CD shows `Syncing` — wait for sync to complete before proceeding.
If Argo CD shows `Degraded` due to the CAS crash — that is expected; proceed.

---

### Step R2 — Delete the crashed pod to reset CrashLoopBackOff backoff timer

> The StatefulSet controller will immediately recreate the pod. This is safe.
> If the pod is in CrashLoopBackOff with high restart count, Kubernetes imposes exponential backoff
> (up to 5 min delay). Deleting the pod resets the timer and forces immediate recreation.

```bash
kubectl delete pod sas-cas-server-default-controller-0 -n sas-viya
```

Watch the pod come back:
```bash
kubectl get pods -n sas-viya -w | grep cas
```

**Validation:**
The pod should transition `Pending → Init → Running` within 3–5 minutes.
If it goes back to `CrashLoopBackOff` within 2 minutes — the underlying cause was not resolved by a restart.
In that case: check logs again (D2) and determine whether it is OOM or an application error.

---

### Step R3 — If OOMKilled: check and adjust CAS memory request (if authorised)

> Only perform this step if you have confirmed `OOMKilled` in D2 AND you are authorised to modify
> Helm values. If not authorised — escalate to SAS Admin / team lead.

Check current CAS memory limit:
```bash
kubectl get pod sas-cas-server-default-controller-0 -n sas-viya \
  -o jsonpath='{.spec.containers[*].resources}' | python3 -m json.tool
```

If the limit is clearly under-provisioned for current load, the permanent fix is to update `sitedefault.yaml` in the GitOps repo and commit → Argo CD sync. Do not patch the pod directly in production.

For an immediate temporary workaround while awaiting the Argo CD sync:
```bash
# Check if there is headroom on the node first (see D5)
# If node has headroom, CAS will restart without OOM once the pod is rescheduled to a larger node
# Trigger Karpenter to consider a larger instance by annotating the node claim if needed
```

---

### Step R4 — Notify SAS Admin and verify user access

Once the pod is `Running 2/2`:

1. Notify the SAS Admin (client) that CAS is back up — they will verify SAS Viya login and job execution.
2. Ask them to confirm at least one SAS Studio job completes successfully.
3. Verify DAMS to SAS Viya API connectivity:

```bash
# From the DAMS EC2 instance via SSM Session Manager:
aws ssm start-session --target i-0a1b2c3d4e5f --region eu-west-1
curl -k -s https://sas-viya.internal.example.com/cas-shared-default-http/cas/sessions \
  -H "Authorization: Bearer [token]" | python3 -m json.tool
```

Expected: JSON response listing active CAS sessions (may be empty if no users are connected yet — that is normal).

---

### Step R5 — Confirm CloudWatch alarm returns to OK

```bash
aws cloudwatch describe-alarms \
  --alarm-names "sas-viya-cas-pod-restarts-high" \
  --region eu-west-1 \
  --query "MetricAlarms[0].StateValue"
```

Expected: `"OK"` — may take up to 5 minutes after CAS is running for the alarm to clear.

---

## Post-Remediation Validation

- [ ] `kubectl get pods -n sas-viya | grep cas` — all CAS pods `Running`, `READY 2/2`, `RESTARTS 0`
- [ ] SAS Viya login page accessible at `https://sas-viya.internal.example.com`
- [ ] SAS Admin (client) confirms at least one job executed successfully
- [ ] DAMS API returning HTTP 200 for SAS Viya endpoint calls
- [ ] Tableau dashboards refreshing correctly (if applicable)
- [ ] CloudWatch alarm `sas-viya-cas-pod-restarts-high` status is `OK`
- [ ] No new `ERROR` or `FATAL` entries in CloudWatch log group `/aws/eks/sas-viya-prod/sas-viya` in the last 5 minutes

```bash
# Quick log check
aws logs filter-log-events \
  --log-group-name "/aws/eks/sas-viya-prod/sas-viya" \
  --filter-pattern "ERROR FATAL" \
  --start-time $(date -d '5 minutes ago' +%s000) \
  --region eu-west-1 \
  --query "events[*].message" \
  --output text
```

---

## Rollback Procedure

If the CAS restart made the situation worse (e.g., other pods are now also crashing):

```bash
# Check if Argo CD can roll back the last sync
argocd app rollback sas-viya --grpc-web

# Or: if the issue is a recent Argo CD deployment, get the previous revision
argocd app history sas-viya --grpc-web
argocd app rollback sas-viya [REVISION_NUMBER] --grpc-web
```

Confirm rollback:
```bash
argocd app get sas-viya --grpc-web | grep -E "Sync|Health"
kubectl get pods -n sas-viya
```

---

## Escalation

| Escalation path | Contact | When |
|-----------------|---------|------|
| SAS Admin (client) | [name] / [Teams handle] | SAS application layer — user sessions, job failures, CASLIB access |
| AWS Support | console.aws.amazon.com/support | EKS control plane issues, RDS failure, EFS mount failure |
| SAS Technical Support | support.sas.com | SAS Viya product defect, OOM that persists after resource increase |
| Your team lead | [name] | P1 unresolved after 30 minutes |

**When escalating, provide:**
1. RB-001, steps D1–D5 completed
2. Output of `kubectl describe pod sas-cas-server-default-controller-0 -n sas-viya`
3. Last 100 lines of `kubectl logs ... --previous`
4. CloudWatch alarm state and timestamp
5. EFS and PostgreSQL check results (D3, D4)

---

## Notes & Known Issues

- **CAS initialization takes 3–5 minutes.** The pod reaches `Running` state but CAS internally takes another 2–3 minutes to become fully ready. The first few CloudWatch health check failures after pod restart are expected — wait before concluding the restart failed.
- **Worker pods restart after controller restart.** `sas-cas-server-default-worker-*` pods will automatically restart when the controller is recreated. This is expected behaviour — do not restart workers independently.
- **Alarm may fire during scheduled maintenance.** If `sas-viya-cas-pod-restarts-high` fires during a known update window, verify the Argo CD sync completed successfully before treating it as an incident.
- **DAMS connection pool:** After CAS recovery, DAMS PHP may need up to 5 minutes to re-establish its connection pool to SAS Viya. If DAMS still returns errors 5 min after CAS is healthy, restart the DAMS PHP-FPM service via SSM.

```bash
# DAMS PHP-FPM restart if needed
aws ssm start-session --target i-0a1b2c3d4e5f --region eu-west-1
# Then inside the session:
sudo systemctl restart php8.1-fpm
sudo systemctl status php8.1-fpm
```

---

## Change Log

| Date | Author | Change |
|------|--------|--------|
| 2026-09-09 | CloudOps | Initial version based on KT session findings |
| 2026-08-15 | CloudOps | Tested during DR rehearsal — added DAMS PHP-FPM note |

---

*Runbook ID: RB-001 | Owner: CloudOps | Review cycle: After each P1 use + quarterly*
