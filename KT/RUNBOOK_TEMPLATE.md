# Runbook: [RUNBOOK TITLE]

---

## Metadata

| Field | Value |
|-------|-------|
| **Runbook ID** | RB-[NUMBER] |
| **Title** | [Short descriptive title] |
| **Environment** | Production / DR / Dev |
| **Service** | EKS / SAS Viya / DAMS / All |
| **Trigger** | [What triggers this runbook — alarm name, alert, user report, scheduled task] |
| **Severity** | P1 / P2 / P3 |
| **Owner** | CloudOps |
| **SAS Admin required?** | Yes / No |
| **Estimated duration** | [X minutes] |
| **Last tested** | YYYY-MM-DD |
| **Last updated** | YYYY-MM-DD |
| **Version** | v1.0 |

---

## BLUF

> **[One or two sentences maximum.]**
> State what this runbook does, when to use it, and the expected outcome.
> Example: "This runbook covers restarting the SAS CAS Server after an OOMKilled or unresponsive CAS pod.
> Outcome: CAS is running, user sessions restored. Estimated time: 10–15 min."

---

## Prerequisites

Before starting, confirm the following:

- [ ] You have `kubectl` configured for the correct cluster (`kubectl config current-context`)
- [ ] You have AWS CLI access to the correct account (`aws sts get-caller-identity`)
- [ ] [Any other access requirement — Argo CD, SSM, Secrets Manager, etc.]
- [ ] [Note any maintenance window requirement if applicable]

**Required access level:** [e.g., EKS read-write, Secrets Manager read, SSM Session Manager]

---

## Symptoms / When to Use This Runbook

- [Symptom 1 — e.g., CloudWatch alarm X fired]
- [Symptom 2 — e.g., users report they cannot log in to SAS Viya]
- [Symptom 3 — e.g., `kubectl get pods -n sas-viya` shows CrashLoopBackOff on pod Y]

---

## Impact Assessment

| Aspect | Detail |
|--------|--------|
| **User impact** | [Who is affected and how — e.g., all SAS Viya users cannot start sessions] |
| **Data risk** | [None / Low / Medium / High — e.g., in-progress CAS jobs will be lost] |
| **Downstream impact** | [e.g., DAMS API calls to SAS Viya will fail; Tableau dashboards will show errors] |
| **Safe to proceed alone?** | [Yes / No — if No, state who to notify before starting] |

---

## Go / No-Go Checklist

Complete this before executing any steps:

- [ ] Confirmed which environment this applies to (Prod / DR / Dev)
- [ ] Notified stakeholders if P1 (SAS Admin, client contact)
- [ ] Verified this is the correct runbook for the symptom
- [ ] Backup / snapshot exists and is recent (if data change is involved)

---

## Diagnosis Steps

> Run these first to confirm the problem before taking action.

### Step D1 — [Diagnosis step title]

```bash
# [Command to run]
kubectl get pods -n [namespace]
```

**Expected output (healthy):**
```
NAME                          READY   STATUS    RESTARTS   AGE
[pod-name]                    1/1     Running   0          2d
```

**Abnormal output indicating the problem:**
```
NAME                          READY   STATUS             RESTARTS   AGE
[pod-name]                    0/1     CrashLoopBackOff   5          10m
```

### Step D2 — [Next diagnosis step]

```bash
kubectl describe pod [pod-name] -n [namespace]
kubectl logs [pod-name] -n [namespace] --previous
```

**What to look for:** [Specific error strings, keywords, or log patterns that confirm the diagnosis]

### Step D3 — Check upstream dependencies

```bash
# Check if the dependency (e.g., PostgreSQL, EFS) is reachable
[command]
```

---

## Remediation Steps

> Execute these steps in order. Do not skip steps.
> Each step includes a validation check — confirm it passes before moving to the next.

### Step R1 — [Action title]

**What this does:** [One sentence explaining why this step is taken]

```bash
[exact command to run]
```

**Validation:**
```bash
[command to verify the step succeeded]
```

Expected result: `[what success looks like]`

⚠️ **If this step fails:** [What to do — skip to escalation, try alternative, check X]

---

### Step R2 — [Next action title]

**What this does:** [Explanation]

```bash
[command]
```

**Validation:**
```bash
[validation command]
```

Expected result: `[success indicator]`

---

### Step R3 — [Continue for each required action]

```bash
[command]
```

**Validation:** [Description of expected state]

---

## Post-Remediation Validation

After all remediation steps are complete, confirm the service is fully healthy:

- [ ] `kubectl get pods -n [namespace]` — all pods in `Running` / `Completed` state
- [ ] [Application-level check — e.g., SAS Viya login page accessible at https://[url]]
- [ ] [Downstream check — e.g., DAMS API returning 200, Tableau dashboard loading]
- [ ] [Monitoring check — e.g., CloudWatch alarm returned to OK state]
- [ ] No new errors in CloudWatch log group `[log-group-name]` in the last 5 minutes

---

## Rollback Procedure

> If remediation made things worse, execute this rollback before escalating.

```bash
[rollback command — e.g., argocd app rollback, kubectl rollout undo, velero restore]
```

Confirm rollback succeeded:
```bash
[validation command]
```

---

## Escalation

If the issue is not resolved after completing this runbook:

| Escalation path | Contact | When |
|-----------------|---------|------|
| SAS Admin (client) | [name / Teams handle] | SAS application layer — jobs, users, CASLIBs |
| AWS Support | console.aws.amazon.com/support | EKS control plane, RDS, EFS infrastructure |
| SAS Technical Support | support.sas.com | SAS Viya product defects, licensing |
| Your team lead | [name] | After 30 min without resolution on P1 |

**When escalating, provide:**
1. Runbook ID and steps completed
2. Output of diagnosis steps D1–D3
3. CloudWatch alarm name and timestamp
4. Any error messages from `kubectl describe` / logs

---

## Notes & Known Issues

- [Any environment-specific quirks — e.g., "CAS always takes 3–4 min to fully initialize after restart; the first 2 health check failures are expected"]
- [Known false positives — e.g., "Alarm X fires briefly after every scheduled maintenance window"]
- [Dependencies or timing considerations]

---

## Change Log

| Date | Author | Change |
|------|--------|--------|
| YYYY-MM-DD | [name] | Initial version |

---

*Runbook ID: RB-[NUMBER] | Owner: CloudOps | Review cycle: After each use + quarterly*
