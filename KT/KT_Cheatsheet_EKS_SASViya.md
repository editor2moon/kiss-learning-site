# KT Session Cheatsheet — EKS & SAS Viya Support Handover

> **BLUF:** Three areas to cover in this session: technical questions to ask, artifacts you must receive, and
> understanding of typical operations you are taking over. Work through each section systematically.
> Missing artifacts = open risk going into DR test.

---

## Section A — EKS Infrastructure: Architecture & Access

| Priority | Question | Why It Matters |
|----------|----------|----------------|
| **CRITICAL** | What is the EKS cluster name, AWS account ID, and region? | Required for `aws eks update-kubeconfig` — without this you have no access at all. |
| **CRITICAL** | Which IAM Roles / IRSA are used for cluster access? | `aws-auth` ConfigMap or EKS Access Entries — who has what rights, how CloudOps authenticates. |
| **CRITICAL** | What is the node pool architecture? (Managed Node Groups vs Karpenter NodePools) | SAS Viya uses dedicated node pools per component — you need to know what runs where. |
| **HIGH** | Is Karpenter or Managed Node Groups in use? Which Karpenter version? | Karpenter v0.x vs v1.x have different CRDs — `NodePool` vs `Provisioner`. |
| **HIGH** | What is the current Kubernetes version? When was the last upgrade? | EKS EOL awareness — you need to know whether a version upgrade is imminent and who owns it. |
| **HIGH** | What does the network look like? (VPC CNI, pod subnets, Security Groups for Pods?) | Networking failures are the top incident category. You need the topology to debug. |
| **MEDIUM** | Are there custom taints/tolerations on node pools? Which ones? | SAS Viya uses dedicated taints — pods won't schedule without matching tolerations. |

---

## Section B — SAS Viya: Architecture & Components

| Priority | Question | Why It Matters |
|----------|----------|----------------|
| **CRITICAL** | Which SAS Viya version is deployed? What deployment method? (SAS Deployment Operator vs viya4-deployment) | Version determines patch compatibility, CRD versions, and Helm chart structure. |
| **CRITICAL** | What are the SAS Viya namespaces in the cluster? What lives in each? | At minimum: `sas-viya` (core), `monitoring`, `ingress`. May be multi-tenant. |
| **CRITICAL** | How is storage configured? (EFS CSI driver, PVCs for CAS and SAS Viya, StorageClasses) | CAS Controller, SAS Studio, and backups all depend on storage. EFS mount targets must exist in each AZ. |
| **HIGH** | How is ingress set up? (ALB Ingress Controller or other? TLS termination?) | Ingress failure = all users lose SAS access simultaneously. |
| **HIGH** | How is the PostgreSQL database integrated? (SAS Infrastructure Data Server endpoint, account, backup) | SAS Viya metadata, schedules, and user sessions depend on PostgreSQL — it is critical path. |
| **HIGH** | How does DAMS integrate with SAS Viya? (REST/JDBC endpoints, auth) | DAMS PHP EC2 calls SAS Viya API — you need the flow to debug DAMS-side issues. |
| **MEDIUM** | Do Tableau, Alteryx, or other tools integrate with SAS Viya? How? | JDBC/ODBC to CAS — CAS failure = dead Tableau dashboards. |

---

## Section C — GitOps & CI/CD (Argo CD, Helm, Harbor)

| Priority | Question | Why It Matters |
|----------|----------|----------------|
| **CRITICAL** | How does Argo CD work? Which repo (CodeCommit / Azure DevOps)? Who has access? | Any SAS Viya config change goes through the repo — you need to know how and where. |
| **HIGH** | Where is Harbor registry? How is the pull secret configured in the SAS namespace? | Pull image failures = pod crashes. Harbor availability = SAS Viya availability. |
| **HIGH** | What is the SAS Viya update procedure? Who owns it, how often does it happen? | SAS LTS vs cadenced releases — you need to know whether updates are your responsibility or the SAS Admin's. |
| **MEDIUM** | Are there custom Helm values or Kustomize overlays for SAS Viya? | `sitedefault.yaml`, custom annotations, resource limits — non-standard config must be understood before the DR test. |

---

## Section D — Observability & Alerting

| Priority | Question | Why It Matters |
|----------|----------|----------------|
| **CRITICAL** | What does monitoring look like? (CloudWatch, Prometheus, Grafana? Where do logs go — Fluent Bit → CloudWatch / OpenSearch?) | Without this you cannot debug incidents. |
| **HIGH** | Which CloudWatch Alarms are active? Who receives notifications (SNS → who)? | After handover you need to confirm alerts route to you, not the outgoing team. |
| **MEDIUM** | Does SAS Viya have its own SAS Environment Manager monitoring? Who operates it? | SAS Environment Manager has its own alerts for CAS, jobs — these are the first application-layer signals. |

---

## Section E — Backup, DR & Security

| Priority | Question | Why It Matters |
|----------|----------|----------------|
| **CRITICAL** | How does Velero backup work? Schedule, S3 bucket location? | DR test requires knowing the Velero restore procedure. |
| **CRITICAL** | Is there a DR site / DR account? What is the failover procedure? | DR test is approximately one month out — this is the absolute top priority. |
| **HIGH** | How are secrets managed? (Secrets Manager, external Vault, Kubernetes Secrets) | DB credentials, SAS license, Harbor pull secret — you need to know how to rotate. |
| **MEDIUM** | Which IRSA roles exist per service account? Who manages IAM policies? | Karpenter, Velero, Fluent Bit, AWS LBC — each needs IRSA configured correctly. |

---

## Section F — Operational History & Procedures

| Priority | Question | Why It Matters |
|----------|----------|----------------|
| **CRITICAL** | What were the major incidents in the last 6 months? What were the resolutions? | Fastest learning before handover — incident history is a map of the environment's real failure modes. |
| **CRITICAL** | Are there documented SOPs / runbooks? Where are they stored? | If they exist — get access. If they don't — you're generating them from scratch. |
| **HIGH** | What is the SLA / RTO / RPO for the SAS Viya and DAMS environment? | Without this you cannot define success criteria for the DR test. |
| **HIGH** | Who is the SAS Administrator (client side)? What is the CloudOps vs SAS Admin boundary? | You own DAMS and infra — but who manages SAS users, CASLIBs, scheduled jobs? You must know where your responsibility ends. |
| **MEDIUM** | What is the escalation path? Who to contact if something breaks at SAS application level? (SAS support ticket, AWS support) | In a crisis there is no time to look up contacts. |

---

## Required Input Artifacts — Checklist

Artifacts you must receive from the outgoing team. If an artifact does not exist, note it explicitly.
A missing artifact is documented justification for generating it from scratch — not a silent gap.

| Artifact | Expected Format / Location | Status |
|----------|---------------------------|--------|
| kubeconfig with EKS access (or SOP to generate it via `aws eks update-kubeconfig`) | File or SOP | **REQUIRED** |
| Architecture diagram — EKS + SAS Viya (node pools, namespaces, dependencies) | Draw.io / Confluence | **REQUIRED** |
| RACI / responsibility split — CloudOps vs SAS Admin vs client | Excel / Confluence | **REQUIRED** |
| Full Helm release list (`helm list -A` with version and chart) | CLI export or document | **REQUIRED** |
| Argo CD Application list (app name, repo, target namespace, sync status) | `argocd app list` output or doc | **REQUIRED** |
| AWS account IDs and aliases — Prod, Dev, DR | Document / table | **REQUIRED** |
| Access to Secrets Manager / Vault (or SOP to retrieve operational credentials) | IAM / documentation | **REQUIRED** |
| CloudWatch Alarms list and SNS endpoints | AWS Console export / doc | **REQUIRED** |
| Velero backup schedule and S3 bucket name | `kubectl get schedule -n velero` or doc | **REQUIRED** |
| SAS license file location and renewal procedure | S3 / Kubernetes Secret | **REQUIRED** |
| Harbor registry access instructions (URL, credentials, projects) | SOP / Secret | **REQUIRED** |
| Fluent Bit → CloudWatch log group names per namespace | CLI output / doc | **REQUIRED** |
| Contacts: SAS Admin (client), AWS TAM, SAS Support | Contact list / RACI | **REQUIRED** |
| DR runbook (if it exists) | Confluence / Word | IF EXISTS |
| SOPs / operational playbooks (CAS restart, node drain, SAS Viya rollback) | Confluence / SharePoint | IF EXISTS |
| Incident history — last 6 months (tickets, postmortems, known issues) | ServiceNow / Jira / email | IF EXISTS |

---

## Typical CloudOps Activities — EKS + SAS Viya

### EKS — Daily / Weekly Operations

| Frequency | Activity | Key Commands / Notes |
|-----------|----------|----------------------|
| Daily | Node health monitoring — respond to `NotReady` nodes | `kubectl get nodes`, `kubectl describe node`, cordon/drain, trigger Karpenter replacement via `NodeClaim` deletion |
| Ad-hoc (P1) | Pod crash / OOMKilled / CrashLoopBackOff triage | `kubectl describe pod`, `kubectl logs --previous`, check resource limits vs requests, CAS memory, license status |
| Quarterly | EKS control plane version upgrade | Upgrade order: control plane → managed addons (CoreDNS, kube-proxy, VPC CNI) → node groups / Karpenter AMI. SAS Viya requires compatibility testing before upgrade. |
| Weekly | Storage monitoring — EFS, PVC capacity, CSI driver | PVC near-full = CAS crash. CloudWatch metric filters for EFS `BurstCreditBalance`. Check PVC binding status. |
| Weekly | Karpenter NodePool scaling review | Check pending pods, NodeClaims status, EC2 API throttling. Verify Spot interruptions (SAS uses Spot only in compute tier). |
| Monthly / Quarterly | Kubernetes secret rotation (TLS, pull secrets) | Harbor pull secret, ingress TLS cert, DB credentials rotation. `kubectl rollout restart` affected deployments after rotation. |

### SAS Viya — Core Operations

| Frequency | Activity | Key Commands / Notes |
|-----------|----------|----------------------|
| Ad-hoc (P1) | CAS Server health — restart, session cleanup | CAS crash = all user sessions lost. `kubectl rollout restart deployment/sas-cas-server`. Check CAS logs in CloudWatch. Verify PVC still mounted. |
| Ad-hoc (P1) | Ingress / ALB troubleshooting — no access to SAS | Check AWS Load Balancer Controller, Ingress rules, target group health. DNS propagation after changes. TLS cert expiry. |
| Every few weeks | SAS Viya deployment update (Argo CD sync) | Commit to GitOps repo → Argo CD auto-sync or manual sync. `argocd app get sas-viya`, review diff. Rollback: `argocd app rollback`. |
| Weekly / Monthly | Velero backup verification and test restore | `velero backup get`, check `CompletionTimestamp` and size. Before DR test: `velero restore --from-backup [name]` in DR environment. |
| Weekly | PostgreSQL (SAS Infrastructure DS) health and backup | RDS connection count, storage growth, automated backup status. Aurora PITR window. SAS Viya does not start without reachable PostgreSQL. |
| Weekly | Fluent Bit / CloudWatch log pipeline verification | Check Fluent Bit pods. Confirm logs are flowing to correct log groups. Verify CloudWatch Insights queries for SAS namespace. |
| Weekly | Argo CD / Harbor health check | `argocd app list` — all apps synced / healthy. Harbor: check disk usage, GC run, pull errors in logs. Pull secret not expired. |

### DAMS (EC2 + PHP) — Inherited Activities

| Frequency | Activity | Key Commands / Notes |
|-----------|----------|----------------------|
| Ad-hoc (P1) | EC2 DAMS instance monitoring and restart | Use SSM Session Manager instead of bastion SSH. Restart Apache/PHP-FPM if DAMS is unresponsive. Check EFS mount and SAS Viya API connectivity. |
| Monthly | OS patching — DAMS EC2 + sva-jump-host | SSM Patch Manager or manual. Verify DAMS application starts after reboot. Check EFS remount (`/etc/fstab`). |
| Weekly | RDS Aurora MySQL (DAMS DB) — backup and monitoring | Automated backup status, storage growth, slow query log. Check DAMS application error log for connection pool exhaustion to MySQL. |

### Incident Triage Pattern — Any EKS / SAS Viya Incident

```
1. CloudWatch Alarm → SNS → you receive alert
2. kubectl get pods -n [namespace]         -- visible pod-level issues?
3. kubectl describe / logs                 -- where is the root cause?
4. Check: node healthy? EFS accessible? PostgreSQL responding? Ingress OK?
5. If application-layer (SAS CAS / job):   escalate to client SAS Admin
6. If infrastructure:                      fix yourself or escalate to AWS Support
```

---

## Key Contacts Template (fill in during KT session)

| Role | Name | Contact | Availability |
|------|------|---------|--------------|
| SAS Administrator (client) | | | |
| AWS Technical Account Manager | | | |
| SAS Technical Support | | support.sas.com / case portal | Business hours |
| Outgoing support team lead | | | |
| Your team escalation point | | | |

---

*Document version: v1.0 | Created: KT Session | Status: Working draft — validate and update after session*
