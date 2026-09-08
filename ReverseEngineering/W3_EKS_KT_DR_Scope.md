# EKS Knowledge Transfer Checklist + DR Scope Matrix
**DOC-W3-003** | Week 3 Deliverable | Owner: CloudOps Team | v0.1 DRAFT

---

> **BLUF:** Two deliverables: (1) EKS KT Checklist — structured list of information to extract from outgoing support team. (2) DR Scope Matrix — defines what is in scope for DR test, backup method per component, restore procedure status, and open gaps.

---

# PART 1 — EKS Knowledge Transfer Checklist

Use during KT sessions with the outgoing EKS support team.  
Status values: **RECEIVED** / **PARTIAL** / **MISSING**

---

## A. Cluster Basics

| Item | Expected Info | Status | Owner / ETA |
|---|---|---|---|
| Cluster name(s) | Name per account (prod/dev/dr) | RECEIVED | — |
| Kubernetes version | Current version + upgrade schedule | RECEIVED | — |
| EKS add-ons | VPC-CNI, CoreDNS, kube-proxy, EBS CSI driver versions | PARTIAL | EKS Team / W4 |
| Node groups / Karpenter | Node group names, instance types, min/max, labels/taints | MISSING | EKS Team / W4 |
| AWS account hosting EKS | Account ID, region per environment | RECEIVED | — |
| Access method | How to get kubeconfig — `aws eks update-kubeconfig` | RECEIVED | — |
| RBAC setup | Who has cluster-admin, what IAM roles map to K8s roles | MISSING | EKS Team / W4 |

---

## B. SAS Viya Components on EKS

| Item | Expected Info | Status | Owner / ETA |
|---|---|---|---|
| Namespaces | List all namespaces — which belongs to SAS Viya | RECEIVED | — |
| Helm releases | `helm list -A` — release names, chart versions, values files location | PARTIAL | EKS Team / W4 |
| Key pods / services | CAS Server, SAS Studio, Logon Manager, Consul, Postgres operator | PARTIAL | EKS Team / W4 |
| SAS Viya version | Cadence + version (e.g. Stable 2024.07) | RECEIVED | — |
| Image registry | Harbor or ECR — URL, auth method, image pull secret name | MISSING | EKS Team / W4 |
| Argo CD apps | List of Argo CD applications managing SAS Viya | MISSING | EKS Team / W4 |
| Ingress / Load Balancer | ALB/NLB, ingress class, TLS certs (ACM ARN or cert-manager) | PARTIAL | EKS Team / W4 |

---

## C. Storage & Data

| Item | Expected Info | Status | Owner / ETA |
|---|---|---|---|
| StorageClasses | Names, provisioner (EBS/EFS), reclaim policy | PARTIAL | EKS Team / W4 |
| PVC inventory | `kubectl get pvc -A` — which PVCs are critical (CAS, Postgres) | MISSING | EKS Team / W4 |
| EFS mount points | EFS File System ID, access points per namespace | MISSING | EKS Team / W4 |
| Database (external) | Is Crunchy Data used OR external Aurora PostgreSQL? | PARTIAL | EKS Team / W4 |
| Backup for PVCs | Is Velero configured? Backup schedule? S3 bucket for backups? | MISSING | EKS Team / W4 |

---

## D. Monitoring & Alerting

| Item | Expected Info | Status | Owner / ETA |
|---|---|---|---|
| CloudWatch Container Insights | Enabled or not, log groups structure | RECEIVED | — |
| Fluent Bit config | Where logs are routed (CW Logs / Splunk / both) | PARTIAL | EKS Team / W4 |
| Grafana / Prometheus | Is cluster-level monitoring deployed in EKS? | MISSING | EKS Team / W4 |
| Alerting | CloudWatch Alarms or PagerDuty for pod failures | MISSING | EKS Team / W4 |

---

## EKS Component Inventory (Discovered via kubectl)

```bash
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running
helm list -A
```

| Namespace | Key Components Found | Replicas | Status | Notes |
|---|---|---|---|---|
| sas-viya | cas-server, sas-studio-app, sas-logon-app | 1 / 1 / 2 | Running | CAS single-node |
| sas-viya | sas-consul-server, sas-postgres | 3 / 1 | Running | Consul HA |
| karpenter | karpenter controller | 2 | Running | EC2 spot provisioning |
| harbor | harbor-core, harbor-portal, harbor-registry | 1 / 1 / 1 | Running | Image registry |
| argocd | argocd-server, argocd-application-controller | 1 / 1 | Running | GitOps |
| monitoring | TBD — not yet explored | — | UNKNOWN | Pending KT |

---

# PART 2 — DR Scope Matrix

---

## DR Objectives

| Parameter | Target | Notes |
|---|---|---|
| RTO (Recovery Time Objective) | < 4 hours | Full SAS Viya operational — TBC with management |
| RPO (Recovery Point Objective) | < 1 hour | Aurora PITR 5-min granularity |
| DR Test Frequency | Quarterly | First test: ~1 month from now |
| DR Account | 777788889999 (eu-central-1) | Separate region from PROD |
| Test Type | Partial restore test | Not full prod cutover — targeted component restore |

---

## DR Component Scope Matrix

| Component | In DR Scope? | Backup Method | Restore Procedure | Doc Status | Gap / Action |
|---|---|---|---|---|---|
| DAMS EC2 (dams-app-prod-01/02) | YES | AWS Backup — EC2 snapshot daily | Launch from snapshot, reconfig SG/IAM | DRAFT | Validate snapshot retention in DR account |
| DAMS EC2 (bastion) | NO | — | Redeploy from AMI | NOT STARTED | Low priority — SSM preferred |
| RDS Aurora MySQL (dams-aurora-prod) | YES | Automated backup + PITR | PITR restore to DR cluster | DRAFT | Test endpoint switch mechanism |
| RDS Aurora PostgreSQL (dams-pg-prod) | YES | Automated backup + PITR | PITR restore to DR cluster | NOT STARTED | Confirm credentials migration to DR |
| EKS Cluster | PARTIAL | No Velero confirmed yet | TBD — pending KT completion | NOT STARTED | **CRITICAL GAP** — confirm if EKS is in DR scope |
| SAS Viya workloads (Helm) | PARTIAL | values files in Git (Argo CD) | Re-deploy via Argo CD to DR cluster | NOT STARTED | DR cluster must exist — confirm with EKS team |
| Harbor Registry | NO | Images also in ECR | Re-point image pull to ECR | NOT STARTED | Confirm ECR as fallback |
| EFS (PVC storage) | PARTIAL | AWS Backup EFS policy | Restore EFS to DR, re-mount | NOT STARTED | Verify cross-region replication enabled |
| Argo CD | NO | GitOps — Git is source of truth | Re-install Argo CD on DR cluster | NOT STARTED | Low effort if DR cluster exists |
| Route 53 DNS | YES | Hosted Zone records | Update A/CNAME records to DR endpoints | NOT STARTED | Define DNS cutover runbook step |
| AWS Secrets Manager | YES | Cross-region replication | Confirm replication to eu-central-1 | NOT STARTED | Validate secrets available in DR account |

> **CRITICAL GAP:** EKS DR posture is unknown until KT completes. If EKS is fully in scope — a DR EKS cluster must exist or be provisionable in eu-central-1. Escalate to AWS Architect immediately if not confirmed.

---

## DR Planning Next Steps (into Week 4)

| # | Action | Owner | Deadline |
|---|---|---|---|
| 1 | Schedule DR Test Planning call — confirm scope, go/no-go criteria | Krystian + Manager | Week 3 Friday |
| 2 | Validate AWS Backup vaults accessible from DR account | CloudOps | Week 3 Thursday |
| 3 | Confirm Secrets Manager cross-region replication to eu-central-1 | CloudOps | Week 3 Thursday |
| 4 | Get EKS DR confirmation from EKS team (cluster exists? scope?) | EKS Team KT | Week 4 Monday |
| 5 | Write detailed DAMS EC2 DR restore steps (SOP format) | CloudOps | Week 4 Tuesday |
| 6 | Write detailed RDS DR restore steps | CloudOps | Week 4 Tuesday |
| 7 | Dry-run DR steps on paper with AWS Architect review | CloudOps + Architect | Week 4 Thursday |

---

*Document version: v0.1 DRAFT | Created: Week 3 | EKS sections pending KT completion*
