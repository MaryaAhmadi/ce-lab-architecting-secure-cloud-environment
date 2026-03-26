# AWS Security Baseline — Verification Report

**Date:** 2026-03-26T16:36:28Z
**Engineer:** Sahebejan Ahmadi
**Account ID:** 
**Region:** 

---

## Controls Implemented

| Control | Description | Status | Command to Verify |
|---|---|---|---|
| Least-privilege SG | SSH from /32 only | ✅ Active | `aws ec2 describe-security-groups --group-ids ` |
| VPC Flow Logs | ALL traffic captured to CloudWatch | ✅ Active | `aws ec2 describe-flow-logs --filter "Name=resource-id,Values="` |
| EBS encryption by default | New volumes auto-encrypted | ✅ Enabled | `aws ec2 get-ebs-encryption-by-default` |
| CloudTrail | All API calls logged to S3 | ✅ Logging | `aws cloudtrail get-trail-status --name lab-m8-08-security-trail` |

---

## Resources Created

- Security Group: 
- Default VPC: 
- VPC Flow Logs: /aws/vpc/flow-logs (CloudWatch)
- CloudTrail: lab-m8-08-security-trail
- CloudTrail S3 Bucket: 
- EC2 Instance: 
- EC2 Public IP: 

---

## Test Results

- SSH connection to EC2 instance: ✅ Successful
- VPC Flow Logs status: ✅ ACTIVE
- CloudTrail RunInstances event visible: ✅ Confirmed
- EBS volume on new instance expected encrypted: ✅ Confirmed by default encryption setting
- SSH access restricted to my public IP only: ✅ Confirmed

---

## Security Design Decisions

| Decision | Rationale |
|---|---|
| SSH restricted to /32 | Least privilege: access limited to one known public IP |
| ALL traffic type in Flow Logs | Captures both accepted and rejected traffic for visibility and troubleshooting |
| Multi-region CloudTrail | Covers API activity across regions |
| 30-day log retention | Balances cost with investigation needs |
| Account-level EBS encryption | Prevents accidental creation of unencrypted new EBS volumes |

