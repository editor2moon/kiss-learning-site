# DAMS Operations Playbook + SOPs
**DOC-W2-002** | Week 2 Deliverable | Owner: CloudOps Team | v0.1 DRAFT

---

> **BLUF:** This playbook defines CloudOps support procedures for DAMS EC2 and RDS components. Four SOPs cover the daily operational scenarios. RACI at the end defines ownership boundaries.

---

## SOP #1 — Instance Lifecycle (Start / Stop / Restart)

**Scope:** CloudOps-initiated instance operations for DAMS EC2 in PROD and DEV.

### Pre-conditions
- Approved change ticket or incident number in hand
- Notify DAMS App Owner at least 30 min before planned restart
- Confirm RDS Aurora cluster is healthy before EC2 restart
- Check active user sessions: CloudWatch metric or SSM Session Manager

### Restart Procedure — PROD

| Step | Action | Command / Detail |
|---|---|---|
| 1 | Log in via SSM Session Manager | `aws ssm start-session --target i-0a1b2c3d4e5f --profile prod` |
| 2 | Check running DAMS services | `sudo systemctl status dams-app dams-agent` |
| 3 | Graceful stop | `sudo systemctl stop dams-app dams-agent` |
| 4 | Confirm stop | `sudo systemctl is-active dams-app  # expect: inactive` |
| 5 | Restart OS if required | `sudo reboot` OR `aws ec2 reboot-instances --instance-ids i-0a1b2c3d4e5f` |
| 6 | Post-restart: verify services | `sudo systemctl status dams-app dams-agent` |
| 7 | Verify DB connectivity | `sudo -u dams /opt/dams/bin/healthcheck.sh` |
| 8 | Confirm with app team | Notify in Teams channel: dams-support |

> **ROLLBACK:** If services fail to start after restart — do NOT attempt manual config edits. Escalate to previous support team (OQ-07 owner) immediately.

---

## SOP #2 — Troubleshooting Guide

### Diagnostic Checklist — Where to Look First

| Layer | Where to Check | Tool / Command |
|---|---|---|
| EC2 Health | EC2 Console > Instance Status Checks | `aws ec2 describe-instance-status --instance-ids i-xxx` |
| OS / App Logs | /var/log/dams/*.log, journalctl | `sudo journalctl -u dams-app --since '1 hour ago'` |
| CloudWatch Logs | Log Group: /aws/ec2/dams-prod | `aws logs tail /aws/ec2/dams-prod --follow` |
| RDS Connectivity | Test port from EC2 | `nc -zv dams-aurora-prod.cluster-xxx.rds.amazonaws.com 3306` |
| SSM Agent | SSM console or agent status on EC2 | `sudo systemctl status amazon-ssm-agent` |
| Disk Space | EC2 terminal | `df -h && du -sh /var/log/dams/*` |
| Memory/CPU | CloudWatch Metrics | Namespace: CWAgent, Metrics: mem_used_percent, cpu_usage_active |

### Known Issues Register

| KI # | Symptom | Root Cause | Resolution | Severity |
|---|---|---|---|---|
| KI-01 | DAMS API returns 503 after RDS failover | Connection pool not refreshed | Restart dams-app service | HIGH |
| KI-02 | SSM Session Manager timeout | ssm-agent crash (memory leak) | `sudo systemctl restart amazon-ssm-agent` | MED |
| KI-03 | High CPU on dams-app-prod-01 during business hours | Batch export job runs without throttle | Coordinate with app team to schedule batch off-peak | MED |
| KI-04 | CloudWatch Logs gap (~15 min) | CW Agent restart loop | `sudo systemctl restart amazon-cloudwatch-agent` | LOW |

---

## SOP #3 — Access & Credentials

### Access Method — SSM Session Manager (Preferred)
- No SSH keys required — IAM role + SSM Agent handles auth
- Requires: AWS CLI + Session Manager Plugin installed locally
- Command: `aws ssm start-session --target <instance-id> --profile <profile>`
- Session logs sent to S3: `s3://cloudops-ssm-logs-prod/sessions/`
- Do NOT use bastion host unless SSM is unavailable — bastion is break-glass only

### Credentials Storage

| Credential Type | Storage Location | Who Can Access | Rotation Policy |
|---|---|---|---|
| RDS Master Password | AWS Secrets Manager: prod/dams/rds-master | CloudOps + App Owner | 90 days — manual |
| DAMS App DB User | AWS Secrets Manager: prod/dams/app-db-user | App team only | TBD |
| API Keys (DAMS) | SSM Parameter Store: /dams/prod/api-key | CloudOps + App Owner | TBD |
| AWS Console Access | AWS IAM Identity Center (SSO) | CloudOps via SSO | Per IAM Identity Center policy |

---

## SOP #4 — Database Operations (RDS Aurora)

### Health Check Procedure
- RDS Console: check cluster status — must be 'Available' on both writer and reader
- Verify latest automated snapshot exists (RDS > Snapshots, sort by date)
- Check Enhanced Monitoring for anomalies: RDS > Monitoring tab
- From EC2: `nc -zv <rds-endpoint> 3306` — expect connection accepted

### Point-in-Time Restore Procedure (Break-Glass)

| Step | Action | Notes |
|---|---|---|
| 1 | Open RDS Console > Select cluster > Actions > Restore to point in time | Requires approved incident ticket |
| 2 | Select target time (UTC) — max 5-min granularity | RPO = 5 min for Aurora |
| 3 | Launch into same VPC, same subnet group, same SG as original | Do NOT change networking |
| 4 | After restore: update Secrets Manager endpoint if cluster ID changes | Or use Route 53 CNAME to abstract endpoint |
| 5 | Validate from DAMS EC2: healthcheck.sh or manual connection test | |
| 6 | Notify app team and document in incident ticket | |

---

## RACI Matrix — DAMS Support

**R=Responsible, A=Accountable, C=Consulted, I=Informed**

| Activity | CloudOps | DAMS App Team | AWS Architect | Management |
|---|---|---|---|---|
| EC2 Start / Stop / Restart | R/A | C | I | I |
| OS Patching | R/A | I | C | I |
| RDS Snapshot / Restore | R | C | A | I |
| Application config changes | I | R/A | C | I |
| Security Group changes | R | C | A | I |
| IAM Role changes | C | I | R/A | I |
| DR Test Execution | R | C | A | I |
| Incident Response (infra layer) | R/A | C | C | I |
| Incident Response (app layer) | I | R/A | C | I |
| Documentation maintenance | R/A | C | I | I |

---

*Document version: v0.1 DRAFT | Created: Week 2 | Pending validation from AWS Architect*
