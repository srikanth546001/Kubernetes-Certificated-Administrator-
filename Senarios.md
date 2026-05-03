You are a Senior AWS DevOps/Infrastructure Engineer. You have been handed 3 Jira tickets (SI-164, SI-385, SI-386) for decommissioning legacy AWS infrastructure (ASA-PRD-Stack1) in account 458797919735 / us-east-1 after a successful migration to the new Totara stack. You must execute this safely, in order, with zero data loss.
PHASE 1 — SI-386: Document Everything First
Prompt:
Code

You are documenting legacy AWS infrastructure before decommission.
Account: 458797919735 | Region: us-east-1 | Stack: ASA-PRD-Stack1

Generate AWS CLI commands AND a Confluence-ready markdown template to capture:

EC2 (8 instances):
- asa-prd-openvpn-nat1 (i-0a99cbb011eafd855) t3.micro - RUNNING
- asa-prd-file1 (i-02c50177d74c3a135) t3.large - RUNNING
- asa-prd-application1 through application6 (m4.large) - STOPPED

For each instance capture:
1. AMI ID → create final AMI backup
2. Attached EBS volume IDs + snapshot IDs
3. Elastic IP addresses
4. Security group IDs and rules
5. IAM instance profile/role ARN
6. VPC ID, subnet ID, availability zone
7. Tags

ALB (ASA-PROD-ALB):
1. ARN and DNS name
2. All listener rules (ports, protocols, certs)
3. Target group ARNs + health check config
4. Route53 alias records pointing to it
5. SSL certificate ARNs

RDS (asa-prddb-totara):
1. Instance identifier + ARN
2. Engine version (postgres) + instance class (db.m5.2xlarge)
3. Parameter group name
4. Option group name
5. Backup retention + backup window
6. Final snapshot ID (take one now)
7. Multi-AZ config
8. Security groups attached

Network:
1. VPC ID + CIDR
2. All subnet IDs + CIDRs
3. Route table IDs
4. NAT Gateway ID
5. Internet Gateway ID

Output format:
- AWS CLI commands to extract all above
- Confluence markdown table: Old Resource | Old ID | New Replacement | New ID | Status
- JSON inventory file for Terraform state reference


PHASE 2 — SI-385: Change Control Document
Prompt:
Code

You are writing a formal Change Control Request (CR) document for
decommissioning AWS infrastructure. Use the inventory from SI-386.

Generate a complete CR document covering:

1. CHANGE SUMMARY
   - Title: Decommission ASA-PRD-Stack1 legacy infrastructure
   - Account: 458797919735 / us-east-1
   - Scope: 8 EC2, 1 ALB (ASA-PROD-ALB), 1 RDS (asa-prddb-totara)
   - Reason: Migration to Totara stack complete

2. RISK ASSESSMENT
   - Risk: CMEIntegration.jar still running on asa-prd-file1
     → Mitigation: Confirm Tomcat XML config updated, process killed
   - Risk: asa-prd-openvpn-nat1 may still serve admin VPN (2022 vintage)
     → Mitigation: Confirm replacement VPN path active before termination
   - Risk: RDS primary-enc cutover not confirmed
     → Mitigation: Verify rds-use1-pzd-asa-totara-primary(-enc) is live
   - Risk: Stale ALB connections after deregistration
     → Mitigation: Drain targets with 300s deregistration delay

3. ROLLBACK PLAN
   - EC2: Restore from AMI snapshots taken in Phase 1
   - RDS: Restore from final snapshot (asa-prddb-totara-final-YYYYMMDD)
   - ALB: Re-register targets, repoint Route53 alias
   - DNS: TTL lowered before change, revert Route53 records
   - Terraform: Keep state backup before destroy

4. MAINTENANCE WINDOW
   - Proposed: Low-traffic window (suggest Sunday 02:00-06:00 UTC)
   - Required approvals: list stakeholders
   - Notification timeline: 5 days prior

5. EXECUTION CHECKLIST (pre/during/post)
   Pre-change gates:
   □ SI-386 documentation complete
   □ Final snapshots verified
   □ Rollback tested in non-prod
   □ Stakeholders notified
   □ Monitoring alerts silenced for decom resources

   During change:
   □ Steps executed in exact sequence
   □ Each step verified before proceeding
   □ Rollback decision point after each phase

   Post-change:
   □ Cost Explorer verified - monthly savings confirmed
   □ No orphaned resources remain
   □ Audit trail complete in Confluence

Output as: Word-ready document with section headers


PHASE 3 — SI-164: Actual Decommission Execution
Prompt:

You are executing the AWS infrastructure decommission for ASA-PRD-Stack1.
All pre-checks passed. Generate step-by-step runbook with exact commands.

STEP 1 — Final Snapshots (before anything else)
Generate commands to:
- Create final RDS snapshot: asa-prddb-totara-final-[DATE]
- Create EFS snapshot if filesystem attached to legacy stack
- Create AMIs for both RUNNING instances:
  * asa-prd-openvpn-nat1 (i-0a99cbb011eafd855)
  * asa-prd-file1 (i-02c50177d74c3a135)
- Verify all snapshots reach 'available' state before proceeding

STEP 2 — Drain & Disable (zero traffic first)
- Deregister all targets from ASA-PROD-ALB with 300s drain wait
- Stop all asa-prd-* instances (cooldown period: 24 hours observation)
- Verify no active connections in ALB access logs
- Kill CMEIntegration.jar process on asa-prd-file1 first

STEP 3 — Verify Pre-Destroy Gates
□ Confirm rds-use1-pzd-asa-totara-primary(-enc) is accepting connections
□ Confirm new ALB asa-prd-totara-alb is serving all traffic
□ Confirm admin VPN works WITHOUT asa-prd-openvpn-nat1
□ Confirm no Route53 records still point to old ALB

STEP 4 — Terraform Destroy Sequence
Generate terraform destroy commands in this exact order:
1. Application servers (asa-prd-application1-6)
2. Cron/scheduled task resources
3. Bastion host (if exists)
4. ALB (ASA-PROD-ALB) + target groups
5. EFS filesystem
6. ElastiCache cluster
7. RDS instance (asa-prddb-totara) — snapshot already taken
8. VPC + all networking (last)

STEP 5 — Manual Cleanup (Terraform won't catch these)
Generate AWS CLI commands to find and delete:
- Orphaned security groups (not attached to any resource)
- Unassociated Elastic IPs
- Stale Route53 records pointing to deleted resources
- Unused IAM roles with 'asa-prd' prefix
- Empty S3 buckets tagged with old stack
- CloudWatch log groups for decommissioned instances

STEP 6 — Verification
- AWS Cost Explorer: tag filter ASA-PRD-Stack1, confirm $0 going forward
- Config/Security Hub: no findings on deleted resource IDs
- CloudTrail: export deletion audit log for compliance

Output as: numbered runbook with exact AWS CLI commands,
expected outputs, and go/no-go checkpoints after each ste

____
BONUS — Fix the Active Issue (Tomcat/CMEIntegration)
Prompt:

On server asa-prd-file1, the process CMEIntegration.jar is still running:
root  1018785  0.6  3.7  /usr/bin/java -Xms256m -Xmx512m -jar CMEIntegration.jar

Generate:
1. Commands to check what this service connects to (DB, APIs, ports)
2. How to check Tomcat server.xml / context.xml for stale datasource connections
3. How to gracefully stop this service and verify it stays stopped
4. How to confirm the replacement service on new stack is handling this integration
5. Steps to add to the change control as a pre-condition gate
