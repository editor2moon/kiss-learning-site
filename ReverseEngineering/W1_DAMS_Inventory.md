# DAMS CloudOps — Inventory Report
**DOC-W1-001** | Week 1 Deliverable | Owner: CloudOps Team | v0.1 DRAFT

---

> **BLUF:** DAMS (Data Access and Management Service) runs on EC2 across 3 AWS accounts. This document provides the baseline inventory required for CloudOps support handover. All data collected via AWS CLI + Console — no existing documentation was available.

---

## 1. AWS Account Overview

| Account ID | Alias | Environment | Region | Status |
|---|---|---|---|---|
| 111122223333 | sas-viya-prod | Production | eu-west-1 | ACTIVE |
| 444455556666 | sas-viya-dev | Development | eu-west-1 | ACTIVE |
| 777788889999 | sas-viya-dr | Disaster Recovery | eu-central-1 | STANDBY |

---

## 2. EC2 Instance Inventory — DAMS

**Collected via:**
```bash
aws ec2 describe-instances \
  --filters "Name=tag:Application,Values=DAMS" \
  --profile <profile>
```

| Instance ID | Name Tag | Type | AMI ID | Account | State | Private IP |
|---|---|---|---|---|---|---|
| i-0a1b2c3d4e5f | dams-app-prod-01 | m5.2xlarge | ami-0abc123def456 | PROD | running | 10.10.1.45 |
| i-0b2c3d4e5f6a | dams-app-prod-02 | m5.2xlarge | ami-0abc123def456 | PROD | running | 10.10.1.46 |
| i-0c3d4e5f6a7b | dams-app-dev-01 | m5.xlarge | ami-0abc123def456 | DEV | running | 10.20.1.10 |
| i-0d4e5f6a7b8c | dams-bastion-prod | t3.micro | ami-0def456abc123 | PROD | running | 10.10.0.5 |

> **NOTE:** AMI IDs are placeholders — validate against actual instances. Confirm whether patch management uses SSM Patch Manager or manual process.

---

## 3. RDS Database Inventory

| Cluster ID | Engine | Version | Endpoint | Account | Multi-AZ |
|---|---|---|---|---|---|
| dams-aurora-prod | Aurora MySQL | 8.0.mysql_aurora.3.04 | dams-aurora-prod.cluster-xxxx.eu-west-1.rds.amazonaws.com | PROD | YES |
| dams-aurora-dev | Aurora MySQL | 8.0.mysql_aurora.3.04 | dams-aurora-dev.cluster-yyyy.eu-west-1.rds.amazonaws.com | DEV | NO |
| dams-pg-prod | Aurora PostgreSQL | 15.4 | dams-pg-prod.cluster-zzzz.eu-west-1.rds.amazonaws.com | PROD | YES |

---

## 4. Networking — VPC / Subnets / Security Groups

| Resource | ID | CIDR / Description | Account | Notes |
|---|---|---|---|---|
| VPC | vpc-0prod1234 | 10.10.0.0/16 | PROD | DAMS Production VPC |
| Subnet A (App) | subnet-0app01 | 10.10.1.0/24 | PROD | AZ: eu-west-1a, private |
| Subnet B (App) | subnet-0app02 | 10.10.2.0/24 | PROD | AZ: eu-west-1b, private |
| Subnet (DB) | subnet-0db01 | 10.10.10.0/24 | PROD | RDS subnet group |
| SG — DAMS App | sg-0damsapp1 | In: 443 from 10.10.0.0/8; Out: all | PROD | Attached to EC2 |
| SG — RDS | sg-0damsrds1 | In: 3306/5432 from sg-0damsapp1 | PROD | Attached to RDS |

---

## 5. IAM Roles & SSM Registration

| Role Name | Attached To | Key Policies | SSM Registered | Verified |
|---|---|---|---|---|
| dams-ec2-role-prod | EC2 DAMS Prod | S3ReadWrite-DAMS, SecretsManagerRead, SSMManagedInstanceCore | YES | YES |
| dams-ec2-role-dev | EC2 DAMS Dev | S3ReadWrite-DAMS, SecretsManagerRead, SSMManagedInstanceCore | YES | YES |
| dams-rds-monitoring | RDS Enhanced Monitoring | AmazonRDSEnhancedMonitoringRole | N/A | YES |

---

## 6. Open Questions — Pending Clarification

| # | Question | Target | Priority |
|---|---|---|---|
| OQ-01 | Confirm AMI IDs — are instances patched from a golden AMI or ad-hoc? | AWS Architect | HIGH |
| OQ-02 | Is there a Patch Manager baseline configured in SSM? | Previous Support | HIGH |
| OQ-03 | What is the DAMS application startup sequence after OS boot? | Previous Support | HIGH |
| OQ-04 | Are RDS credentials stored in Secrets Manager or SSM Parameter Store? | Previous Support | HIGH |
| OQ-05 | Are there EventBridge rules or Lambda triggers tied to DAMS EC2 lifecycle? | AWS Architect | MED |
| OQ-06 | DR account (777788889999) — is DAMS deployed or just AMI snapshots exist? | AWS Architect | MED |
| OQ-07 | Who owns the DAMS application layer (restart, config changes)? | AWS Architect / RACI | HIGH |

> **ACTION REQUIRED:** Send Open Questions to AWS Architect and Previous Support team by EOD Week 1 Friday.

---

*Document version: v0.1 DRAFT | Created: Week 1 | Next review: End of Week 4*
