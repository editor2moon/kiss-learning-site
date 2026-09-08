# DR Test Runbook — SAS Viya (DAMS + RDS + EKS)
**DOC-W4-004** | Week 4 Deliverable | Owner: CloudOps Team | **v1.0 — Requires sign-off before execution**

---

> **BLUF:** Execution-ready DR test runbook covering DAMS EC2, RDS Aurora, and EKS/SAS Viya. Use the Day-of Execution Checklist (Section 6) tick-by-tick during the actual DR test. All steps must be reviewed and signed off by AWS Architect before test date.

---

## 1. Go / No-Go Criteria

All conditions must be met before DR test execution begins:

| # | Criterion | Verified By | Status |
|---|---|---|---|
| G1 | AWS Backup vaults accessible in DR account (777788889999) | CloudOps | [ ] YES / [ ] NO |
| G2 | Latest EC2 snapshot for dams-app-prod-01/02 exists (<24h old) | CloudOps | [ ] YES / [ ] NO |
| G3 | RDS Aurora MySQL latest automated snapshot verified in DR account | CloudOps | [ ] YES / [ ] NO |
| G4 | Secrets Manager cross-region replication confirmed in eu-central-1 | CloudOps | [ ] YES / [ ] NO |
| G5 | DR VPC, subnets, SGs pre-created in DR account (or confirmed existing) | AWS Architect | [ ] YES / [ ] NO |
| G6 | EKS DR cluster available in eu-central-1 OR EKS excluded from scope | EKS Team / Architect | [ ] YES / [ ] NO |
| G7 | DAMS App team notified — non-prod test window agreed | Manager | [ ] YES / [ ] NO |
| G8 | Rollback plan reviewed and signed off | AWS Architect | [ ] YES / [ ] NO |
| G9 | Participants confirmed for DR test call (Teams/Zoom bridge open) | Manager | [ ] YES / [ ] NO |

> **NO-GO:** If ANY of G1–G5 is NO — stop. Do not proceed. G6 NO is acceptable only if EKS is formally excluded from scope (document the exclusion).

---

## 2. Phase 1 — DAMS EC2 Restore

**Estimated time: 30–60 minutes**

| Step | Action | Command / Detail | Expected Result |
|---|---|---|---|
| EC2-01 | Identify latest snapshot in DR account | `aws ec2 describe-snapshots --owner-ids self --filters Name=tag:Name,Values=dams-app-prod-01 --query 'sort_by(Snapshots,&StartTime)[-1]' --profile dr` | Snapshot ID returned, StartTime < 24h |
| EC2-02 | Launch EC2 from snapshot in DR account | `aws ec2 run-instances --image-id <ami-from-snapshot> --instance-type m5.2xlarge --subnet-id <dr-subnet> --security-group-ids <dr-sg> --iam-instance-profile Name=dams-ec2-role-dr --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=dams-app-dr-01}]' --profile dr` | Instance: pending → running |
| EC2-03 | Wait for instance status checks | `aws ec2 wait instance-status-ok --instance-ids <new-instance-id> --profile dr` | Both status checks = passed |
| EC2-04 | Verify SSM registration | `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<id> --profile dr` | Instance appears in SSM |
| EC2-05 | Connect via SSM Session Manager | `aws ssm start-session --target <instance-id> --profile dr` | Shell prompt on DR instance |
| EC2-06 | Check DAMS services status | `sudo systemctl status dams-app dams-agent` | Services may be stopped — expected (DB not yet restored) |
| EC2-07 | Verify DAMS config points to DR RDS endpoint | `cat /opt/dams/conf/database.conf \| grep endpoint` | Must show DR RDS endpoint after Phase 2 |
| EC2-08 | **HOLD** — wait for Phase 2 (RDS restore) before starting services | — | Do not start dams-app until DB validated |

---

## 3. Phase 2 — RDS Aurora Restore

**Estimated time: 20–40 minutes**

| Step | Action | Command / Detail | Expected Result |
|---|---|---|---|
| RDS-01 | List available snapshots in DR account | `aws rds describe-db-cluster-snapshots --snapshot-type automated --db-cluster-identifier dams-aurora-prod --profile dr` | List of snapshots — pick latest |
| RDS-02 | Restore cluster from snapshot | `aws rds restore-db-cluster-from-snapshot --db-cluster-identifier dams-aurora-dr --snapshot-identifier <snapshot-id> --engine aurora-mysql --engine-version 8.0.mysql_aurora.3.04 --vpc-security-group-ids <dr-sg-rds> --db-subnet-group-name dams-dr-subnet-group --profile dr` | Status: creating → available |
| RDS-03 | Add DB instance to restored cluster | `aws rds create-db-instance --db-instance-identifier dams-aurora-dr-instance-1 --db-cluster-identifier dams-aurora-dr --db-instance-class db.r6g.xlarge --engine aurora-mysql --profile dr` | Status: creating → available |
| RDS-04 | Get cluster endpoint | `aws rds describe-db-clusters --db-cluster-identifier dams-aurora-dr --query 'DBClusters[0].Endpoint' --profile dr` | Endpoint string returned |
| RDS-05 | Update DAMS config on DR EC2 with new endpoint | `sudo sed -i 's/dams-aurora-prod.cluster-xxx.rds.amazonaws.com/<dr-endpoint>/g' /opt/dams/conf/database.conf` | Config updated |
| RDS-06 | Test connectivity from DR EC2 | `nc -zv <dr-rds-endpoint> 3306` | Connection succeeded |
| RDS-07 | Start DAMS services on DR EC2 | `sudo systemctl start dams-app dams-agent && sudo systemctl status dams-app` | Services: active (running) |
| RDS-08 | Run DAMS healthcheck | `sudo -u dams /opt/dams/bin/healthcheck.sh` | All checks PASS — document output |

---

## 4. Phase 3 — EKS / SAS Viya (Conditional)

> **CONDITIONAL:** This phase executes ONLY if EKS DR cluster exists in eu-central-1 AND EKS was confirmed in DR scope at Go/No-Go (G6). If excluded — mark as SKIPPED and document reason.

| Step | Action | Command / Detail | Expected Result |
|---|---|---|---|
| EKS-01 | Get kubeconfig for DR cluster | `aws eks update-kubeconfig --name sas-viya-dr --region eu-central-1 --profile dr` | Context switched to DR cluster |
| EKS-02 | Verify node readiness | `kubectl get nodes -o wide` | All nodes Ready |
| EKS-03 | Check SAS Viya namespace | `kubectl get pods -n sas-viya` | Pods visible in DR namespace |
| EKS-04 | Trigger Argo CD sync for SAS Viya apps | `argocd app sync sas-viya --prune` | Sync status: Synced, Health: Healthy |
| EKS-05 | Monitor pod startup | `kubectl get pods -n sas-viya -w` | All critical pods Running within 15–20 min |
| EKS-06 | Verify SAS Viya Logon page accessible | `curl -k https://<dr-ingress-endpoint>/SASLogon/login` | HTTP 200 or redirect to login page |
| EKS-07 | Run SAS Viya smoke test (if available) | Per SAS Viya DR smoke test script (TBD from app team) | All smoke tests PASS |

---

## 5. Validation & Rollback

### Validation Criteria — DR Test PASS Conditions

| # | Validation Check | Method | Pass Condition |
|---|---|---|---|
| V1 | DAMS EC2 in DR is running | `systemctl status dams-app` | active (running) |
| V2 | DAMS connected to DR RDS | `healthcheck.sh` | All checks PASS |
| V3 | RDS Aurora DR cluster Available | `aws rds describe-db-clusters` | Status = available |
| V4 | SAS Viya pods running in DR EKS (if in scope) | `kubectl get pods -n sas-viya` | All critical pods Running |
| V5 | SAS Viya UI accessible | Browser / curl to DR ingress | HTTP 200, login page renders |
| V6 | RTO target met | Timestamp: start → V5 pass | < 4 hours elapsed |
| V7 | RPO verified | Check RDS snapshot age at RDS-01 | Snapshot age < 1 hour |

### Rollback — Cleanup After DR Test

```bash
# Terminate DR EC2 instances
aws ec2 terminate-instances --instance-ids <dr-instance-ids> --profile dr

# Delete DR RDS cluster
aws rds delete-db-cluster \
  --db-cluster-identifier dams-aurora-dr \
  --skip-final-snapshot \
  --profile dr
```

- If EKS: Delete Argo CD apps in DR namespace (or scale down to 0) — do NOT delete DR cluster itself
- Document all timings, observations, and failures in DR Test Report
- Send DR Test Report to manager and AWS Architect within 24h of test

---

## 6. Day-of Execution Checklist

> **Print this or open on secondary screen during DR test. Record timestamps + notes in real time.**

| # | Checkpoint | Target Time | Done? | Actual / Notes |
|---|---|---|---|---|
| C01 | Go/No-Go call — all G1–G9 confirmed | T+0:00 | [ ] | |
| C02 | EC2-01: Snapshot identified | T+0:05 | [ ] | |
| C03 | EC2-02: DR instance launching | T+0:10 | [ ] | |
| C04 | EC2-03: Instance status checks OK | T+0:20 | [ ] | |
| C05 | EC2-04/05: SSM session open | T+0:25 | [ ] | |
| C06 | RDS-02: DR cluster restore started | T+0:30 | [ ] | |
| C07 | RDS-03: DB instance created | T+0:45 | [ ] | |
| C08 | RDS-04/05/06: Endpoint updated, connectivity OK | T+1:00 | [ ] | |
| C09 | EC2-07/08: DAMS config updated, services started | T+1:10 | [ ] | |
| C10 | V1+V2: DAMS running + healthcheck PASS | T+1:15 | [ ] | |
| C11 | EKS-01–04: Argo CD sync triggered (if in scope) | T+1:20 | [ ] | |
| C12 | EKS-05/06: Pods running + UI accessible (if in scope) | T+1:40 | [ ] | |
| C13 | V3–V7: All validation criteria checked | T+2:00 | [ ] | |
| C14 | RTO confirmed — time elapsed recorded | T+X:XX | [ ] | |
| C15 | Rollback executed — DR resources cleaned up | T+X+1:00 | [ ] | |
| C16 | DR Test Report notes completed | EOD | [ ] | |

### DR Test Outcome Record

| Field | Value |
|---|---|
| Test Date | ___________________________ |
| Test Participants | ___________________________ |
| RTO Achieved | ___ hours ___ minutes |
| RPO Achieved | Snapshot age: ___ minutes |
| Components Tested | DAMS EC2: YES/NO  \|  RDS: YES/NO  \|  EKS: YES/NO/SKIPPED |
| Overall Result | PASS / FAIL / PARTIAL |
| Failed Steps | ___________________________ |
| Action Items for Next DR Test | ___________________________ |
| Signed off by | Manager: ___________  \|  AWS Architect: ___________ |

---

*Document version: v1.0 | Created: Week 4 | Requires sign-off before DR test execution*
