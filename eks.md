# EKS + SAS Viya — Configuration Audit

> **Purpose:** Inventory of the current EKS/SAS Viya environment configuration, compared against documented state.  
> **Instructions:** Fill in the `Found` and `Delta` columns during the review.  
> **Last updated:** <!-- DATE -->  
> **Performed by:** <!-- NAME/NICK -->  
> **Cluster:** <!-- CLUSTER NAME -->  
> **AWS Region:** <!-- e.g. eu-central-1 -->

---

## Table of Contents

1. [EKS Cluster — Basics](#1-eks-cluster--basics)
2. [Node Groups / Karpenter](#2-node-groups--karpenter)
3. [Networking — VPC CNI, Security Groups, Load Balancer](#3-networking--vpc-cni-security-groups-load-balancer)
4. [Storage — EBS CSI, EFS CSI, StorageClasses](#4-storage--ebs-csi-efs-csi-storageclasses)
5. [Authorization — IRSA / EKS Access Entries / aws-auth](#5-authorization--irsa--eks-access-entries--aws-auth)
6. [SAS Viya — Namespace and Core Components](#6-sas-viya--namespace-and-core-components)
7. [Argo CD — GitOps State](#7-argo-cd--gitops-state)
8. [Observability — Fluent Bit, CloudWatch Agent, Harbor](#8-observability--fluent-bit-cloudwatch-agent-harbor)
9. [Resource Quotas, LimitRanges, Priority Classes](#9-resource-quotas-limitranges-priority-classes)
10. [Summary Health Check](#10-summary-health-check)
11. [Delta Table — Deviation Summary](#11-delta-table--deviation-summary)

---

## 1. EKS Cluster — Basics

### Commands

```bash
# Cluster version, endpoint, status
aws eks describe-cluster --name <CLUSTER_NAME> \
  --query 'cluster.{version:version,endpoint:endpoint,status:status,roleArn:roleArn}'

# List managed add-ons
aws eks list-addons --cluster-name <CLUSTER_NAME>

# Describe a specific add-on
aws eks describe-addon --cluster-name <CLUSTER_NAME> --addon-name <ADDON_NAME>

# OIDC provider
aws iam list-open-id-connect-providers
```

### Findings

| Parameter | Found | Documented | Delta |
|---|---|---|---|
| Kubernetes version | | | |
| Cluster status | | | |
| Endpoint access | `Public` / `Private` / `Both` | | |
| Cluster IAM Role ARN | | | |
| OIDC Provider ARN | | | |

### Managed Add-ons

| Add-on | Found version | Documented version | Status | Delta |
|---|---|---|---|---|
| `vpc-cni` | | | | |
| `coredns` | | | | |
| `kube-proxy` | | | | |
| `aws-ebs-csi-driver` | | | | |
| `aws-efs-csi-driver` | | | | |

### Notes

<!-- Enter observations, anomalies, open questions -->

---

## 2. Node Groups / Karpenter

### Commands

```bash
# Managed Node Groups
aws eks list-nodegroups --cluster-name <CLUSTER_NAME>
aws eks describe-nodegroup --cluster-name <CLUSTER_NAME> --nodegroup-name <NG_NAME>

# Karpenter — NodePools
kubectl get nodepool -o yaml

# Karpenter — EC2NodeClass
kubectl get ec2nodeclass -o yaml

# Nodes with labels
kubectl get nodes -o wide --show-labels

# Node taints and SAS workload class
kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,TAINTS:.spec.taints[*].key,SAS_CLASS:.metadata.labels.workload\.sas\.com/class'
```

### NodePools — Karpenter

| NodePool | Instance types | SAS class (`workload.sas.com/class`) | Taint | CPU limit | RAM limit | Delta |
|---|---|---|---|---|---|---|
| | | | | | | |
| | | | | | | |

### SAS Viya Workload Classes

| Class | Purpose | NodePool | Current nodes | Delta |
|---|---|---|---|---|
| `cas` | CAS Server (in-memory analytics) | | | |
| `compute` | SAS Compute Server | | | |
| `stateless` | Stateless services (REST, UI) | | | |
| `stateful` | Stateful services (databases, queues) | | | |
| `system` | Cluster infrastructure | | | |

### Nodes — Current State

```
# Paste output: kubectl get nodes -o wide
```

### Notes

<!-- Enter observations, anomalies, open questions -->

---

## 3. Networking — VPC CNI, Security Groups, Load Balancer

### Commands

```bash
# VPC CNI — environment variables
kubectl describe daemonset aws-node -n kube-system | grep -E "ENABLE|PREFIX|POD_SECURITY|WARM"

# Security Group per Pod policies
kubectl get securitygrouppolicies -A -o yaml

# AWS Load Balancer Controller — version
kubectl get deployment -n kube-system aws-load-balancer-controller \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Ingress resources
kubectl get ingress -A

# LoadBalancer services
kubectl get svc -A --field-selector spec.type=LoadBalancer
```

### VPC CNI

| Parameter | Found | Documented | Delta |
|---|---|---|---|
| `ENABLE_PREFIX_DELEGATION` | | | |
| `ENABLE_POD_ENI` (SGPP) | | | |
| `WARM_PREFIX_TARGET` | | | |
| `WARM_ENI_TARGET` | | | |

### AWS Load Balancer Controller

| Parameter | Found | Documented | Delta |
|---|---|---|---|
| Image version | | | |
| Replicas | | | |

### Security Group per Pod

| Policy name | Namespace | Pod selector | SG ID | Delta |
|---|---|---|---|---|
| | | | | |

### Load Balancers (Services / Ingress)

| Name | Namespace | Type (NLB/ALB) | DNS | Internal/External | Purpose | Delta |
|---|---|---|---|---|---|---|
| | | | | | | |
| | | | | | | |

### Notes

<!-- Enter observations, anomalies, open questions -->

---

## 4. Storage — EBS CSI, EFS CSI, StorageClasses

### Commands

```bash
# StorageClasses
kubectl get storageclass -o wide

# PersistentVolumes
kubectl get pv -o wide

# PVCs — all namespaces
kubectl get pvc -A

# Unbound PVCs (potential issue)
kubectl get pvc -A --field-selector=status.phase!=Bound

# EFS — file systems
aws efs describe-file-systems \
  --query 'FileSystems[*].{ID:FileSystemId,Name:Name,State:LifeCycleState,Size:SizeInBytes.Value}'

# EFS — Access Points
aws efs describe-access-points \
  --query 'AccessPoints[*].{ID:AccessPointId,FS:FileSystemId,Path:RootDirectory.Path,State:LifeCycleState}'
```

### StorageClasses

| SC name | Provisioner | ReclaimPolicy | VolumeBindingMode | Default | Delta |
|---|---|---|---|---|---|
| | | | | | |
| | | | | | |

### EFS

| FileSystem ID | Name | State | Access Point ID | Path | SAS purpose | Delta |
|---|---|---|---|---|---|---|
| | | | | | | |

### PVCs — SAS Viya (key volumes)

| Namespace | PVC name | SC | AccessMode | Size | State | Purpose | Delta |
|---|---|---|---|---|---|---|---|
| | | | RWO/RWX | | | | |
| | | | RWO/RWX | | | | |

### Notes

<!-- Enter observations, anomalies, open questions -->

---

## 5. Authorization — IRSA / EKS Access Entries / aws-auth

### Commands

```bash
# Access Entries (new model)
aws eks list-access-entries --cluster-name <CLUSTER_NAME>
aws eks list-associated-access-policies --cluster-name <CLUSTER_NAME> --principal-arn <ARN>

# aws-auth ConfigMap (legacy model)
kubectl get configmap aws-auth -n kube-system -o yaml

# Service Accounts with IRSA annotations
kubectl get serviceaccount -A -o json | jq '
  .items[]
  | select(.metadata.annotations."eks.amazonaws.com/role-arn")
  | {
      namespace: .metadata.namespace,
      sa: .metadata.name,
      role: .metadata.annotations."eks.amazonaws.com/role-arn"
    }'
```

### Authorization Model

| Parameter | Found | Documented | Delta |
|---|---|---|---|
| Active model | Access Entries / aws-auth / mixed | | |
| Platform version supporting AE | | | |

### IRSA — Service Accounts with IAM Role

| Namespace | Service Account | IAM Role ARN | Purpose | Delta |
|---|---|---|---|---|
| `kube-system` | `ebs-csi-controller-sa` | | EBS CSI | |
| `kube-system` | `efs-csi-controller-sa` | | EFS CSI | |
| `kube-system` | `aws-load-balancer-controller` | | LB Controller | |
| `karpenter` | `karpenter` | | Karpenter | |
| `<SAS_NS>` | | | SAS | |

### Access Entries / aws-auth — Permissions

| Principal ARN | Type | Policy / Groups | Access level | Delta |
|---|---|---|---|---|
| | `STANDARD` / `EC2` | | `cluster-admin` / other | |

### Notes

<!-- Enter observations, anomalies, open questions -->

---

## 6. SAS Viya — Namespace and Core Components

### Commands

```bash
# SAS namespace
kubectl get namespace | grep -i sas

# Workload overview
kubectl get deploy,sts,ds -n <SAS_NAMESPACE>

# Pods not in Running state
kubectl get pods -n <SAS_NAMESPACE> \
  --field-selector=status.phase!=Running,status.phase!=Succeeded

# CASDeployment (SAS CRD)
kubectl get casdeployment -n <SAS_NAMESPACE> -o yaml

# SAS Operator — logs
kubectl get pods -n <SAS_NAMESPACE> | grep operator
kubectl logs -n <SAS_NAMESPACE> <SAS_OPERATOR_POD> --tail=100

# SAS Viya version (cadence)
kubectl get cm -n <SAS_NAMESPACE> sas-deployment-metadata -o yaml 2>/dev/null || \
kubectl get cm -n <SAS_NAMESPACE> | grep -i version
```

### SAS Viya Version

| Parameter | Found | Documented | Delta |
|---|---|---|---|
| Cadence (LTS/Stable) | | | |
| Version / month | | | |
| Namespace | | | |

### CASDeployment

| Parameter | Found | Documented | Delta |
|---|---|---|---|
| Primary nodes count | | | |
| Worker nodes count | | | |
| Node selector (`workload.sas.com/class`) | `cas` | | |
| PVC (backstore) | | | |
| PVC size | | | |

### Workloads — Overview

| Type | Found count | Documented count | Unhealthy | Delta |
|---|---|---|---|---|
| Deployments | | | | |
| StatefulSets | | | | |
| DaemonSets | | | | |
| Pods `Running` | | | | |
| Pods `CrashLoop`/`Error` | | | | |

### Unhealthy Pods

```
# Paste output: kubectl get pods -n <SAS_NAMESPACE> --field-selector=status.phase!=Running,status.phase!=Succeeded
```

### Notes

<!-- Enter observations, anomalies, open questions -->

---

## 7. Argo CD — GitOps State

### Commands

```bash
# Applications — list and status
kubectl get applications -n argocd

# Applications — detailed JSON
kubectl get applications -n argocd -o json | jq '.items[] | {
  name: .metadata.name,
  sync: .status.sync.status,
  health: .status.health.status,
  repo: .spec.source.repoURL,
  path: .spec.source.path,
  revision: .spec.source.targetRevision
}'

# ApplicationSets
kubectl get applicationset -n argocd

# Argo CD version
kubectl get deployment argocd-server -n argocd \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### Argo CD — Basics

| Parameter | Found | Documented | Delta |
|---|---|---|---|
| Argo CD version | | | |
| Installation namespace | | | |
| Number of Applications | | | |
| Number of ApplicationSets | | | |

### Applications — Status

| Application name | Repo / Path | Revision | Sync status | Health | Delta |
|---|---|---|---|---|---|
| | | | Synced/OutOfSync | Healthy/Degraded | |
| | | | | | |

### Applications in Bad State

```
# Paste output: kubectl get applications -n argocd | grep -v "Synced.*Healthy"
```

### Notes

<!-- Enter observations, anomalies, open questions -->

---

## 8. Observability — Fluent Bit, CloudWatch Agent, Harbor

### Commands

```bash
# Fluent Bit — version and config
kubectl get ds -n amazon-cloudwatch fluent-bit \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl get configmap -n amazon-cloudwatch fluent-bit-config -o yaml

# CloudWatch Agent — version
kubectl get ds -n amazon-cloudwatch cloudwatch-agent \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Container Insights — check
kubectl get configmap -n amazon-cloudwatch cwagentconfig -o yaml | grep -i metrics

# Harbor — services and ingress
kubectl get svc,ingress -n harbor

# Harbor — version
kubectl get deployment -n harbor harbor-core \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### Fluent Bit

| Parameter | Found | Documented | Delta |
|---|---|---|---|
| Image version | | | |
| CloudWatch Log Group (application) | | | |
| CloudWatch Log Group (system) | | | |
| Parser (Docker/CRI) | | | |

### CloudWatch Agent

| Parameter | Found | Documented | Delta |
|---|---|---|---|
| Image version | | | |
| Container Insights enabled | Yes/No | | |
| Custom metrics (SAS) | | | |

### Harbor — Container Registry

| Parameter | Found | Documented | Delta |
|---|---|---|---|
| URL / Ingress host | | | |
| Harbor version | | | |
| Projects (SAS images) | | | |
| SAS pulling from Harbor | Yes/No | | |

### Notes

<!-- Enter observations, anomalies, open questions -->

---

## 9. Resource Quotas, LimitRanges, Priority Classes

### Commands

```bash
# ResourceQuotas
kubectl get resourcequota -A -o custom-columns=\
'NS:.metadata.namespace,NAME:.metadata.name,HARD:.spec.hard'

# LimitRanges
kubectl get limitrange -A -o yaml

# PriorityClasses
kubectl get priorityclass -o custom-columns=\
'NAME:.metadata.name,VALUE:.value,GLOBAL-DEFAULT:.globalDefault,PREEMPTION:.preemptionPolicy'
```

### ResourceQuotas — SAS Namespace

| Namespace | Resource | Limit | Used | % | Delta |
|---|---|---|---|---|---|
| | CPU requests | | | | |
| | CPU limits | | | | |
| | Memory requests | | | | |
| | Memory limits | | | | |
| | PVC count | | | | |

### PriorityClasses

| Name | Value | Global default | SAS purpose | Delta |
|---|---|---|---|---|
| | | | | |
| | | | | |

### Notes

<!-- Enter observations, anomalies, open questions -->

---

## 10. Summary Health Check

### Commands

```bash
# Node state
kubectl get nodes -o wide

# All pods not in Running/Succeeded state
kubectl get pods -A \
  --field-selector=status.phase!=Running,status.phase!=Succeeded \
  | grep -v "Completed"

# Recent Warning events
kubectl get events -A \
  --sort-by='.lastTimestamp' \
  --field-selector type=Warning \
  | tail -40

# Component versions
kubectl version --short 2>/dev/null || kubectl version

# Karpenter — recent provisioning decisions
kubectl logs -n karpenter deployment/karpenter --tail=50 \
  | grep -E "ERROR|WARN|launched|terminated"
```

### Node State

```
# Paste output: kubectl get nodes -o wide
```

### Warning Events — Last 40

```
# Paste output: kubectl get events -A --sort-by='.lastTimestamp' --field-selector type=Warning | tail -40
```

### Notes

<!-- Enter observations, anomalies, open questions -->

---

## 11. Delta Table — Deviation Summary

> Fill in after completing all sections.  
> Priority: `HIGH` = blocking / production risk · `MED` = non-compliance requiring fix · `LOW` = cosmetic / documentation gap.

| # | Area | Parameter | Documented state | Actual state | Delta | Priority | Action |
|---|---|---|---|---|---|---|---|
| 1 | | | | | | HIGH/MED/LOW | |
| 2 | | | | | | | |
| 3 | | | | | | | |

---

## Appendices

> Place full command outputs here as code blocks when too large for tables.

### A. `kubectl get nodes -o wide`

```
# paste here
```

### B. `kubectl get pvc -A`

```
# paste here
```

### C. `kubectl get applications -n argocd`

```
# paste here
```

### D. `aws eks describe-cluster` (full JSON)

```json
// paste here
```

---

*Generated as an audit template — fill in during environment review.*
