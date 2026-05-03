You are a Senior AWS DevOps/Infrastructure Engineer executing a full 
decommission of legacy infrastructure ASA-PRD-Stack1.

Account: 458797919735
Region: us-east-1
New Stack (replacement): ec2-use1-prd-asa-totara-* / asa-prd-totara-alb

ALL 8 EC2 INSTANCES TO DECOMMISSION (explicit):
┌─────────────────────────────┬──────────────────────┬─────────┬──────────┐
│ Name                        │ Instance ID          │ Type    │ State    │
├─────────────────────────────┼──────────────────────┼─────────┼──────────┤
│ asa-prd-openvpn-nat1        │ i-0a99cbb011eafd855  │ t3.micro│ RUNNING  │
│ asa-prd-file1               │ i-02c50177d74c3a135  │ t3.large│ RUNNING  │
│ asa-prd-application1        │ i-01d7ae0109e7f49ef  │ m4.large│ STOPPED  │
│ asa-prd-application2        │ i-0085011c8c32a3b42  │ m4.large│ STOPPED  │
│ asa-prd-application3        │ i-07f5a02fc7e4d66d7  │ m4.large│ STOPPED  │
│ asa-prd-application4        │ i-041331619200c83af  │ m4.large│ STOPPED  │
│ asa-prd-application5        │ i-06cdb67d4971f152c  │ m4.large│ STOPPED  │
│ asa-prd-application6        │ i-07068bf443ebac8bd  │ m4.large│ STOPPED  │
└─────────────────────────────┴──────────────────────┴─────────┴──────────┘

OTHER RESOURCES:
- ALB:  ASA-PROD-ALB (created 2025-09-02) → replaced by asa-prd-totara-alb
- RDS:  asa-prddb-totara (db.m5.2xlarge, postgres)
        → replaced by rds-use1-pzd-asa-totara-primary(-enc)

ACTIVE RISK:
- CMEIntegration.jar still running on asa-prd-file1:
  PID 1018785 | root | /usr/bin/java -Xms256m -Xmx512m -jar CMEIntegration.jar

Execute ALL phases below in strict order. 
Do NOT proceed to next phase until current phase is 100% complete.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PHASE 0 — CRITICAL PRE-CHECK (Before anything)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Generate commands and checklist to verify:

0.1 CMEIntegration.jar investigation on asa-prd-file1:
    a) What DB/API endpoints does it connect to:
       ss -tulpn | grep 1018785
       cat /proc/1018785/net/tcp
       lsof -p 1018785 | grep -E 'TCP|ESTABLISHED'
    b) Check Tomcat config for stale datasource connections:
       find / -name "server.xml" -o -name "context.xml" 2>/dev/null
       grep -r "jdbc\|datasource\|url=" /opt/tomcat/conf/
       grep -r "asa-prddb-totara\|old-db-endpoint" /opt/tomcat/conf/
    c) Confirm replacement service handles this integration on new stack:
       curl -v https://[new-stack-endpoint]/cme-integration/health
    d) Graceful shutdown sequence:
       systemctl stop cme-integration || kill -15 1018785
       sleep 30 && ps aux | grep CMEIntegration  # confirm dead
       systemctl disable cme-integration
    e) Add as hard gate in Change Control: 
       "CMEIntegration.jar confirmed stopped AND replacement verified"

0.2 VPN continuity verification:
    - Confirm replacement VPN path active:
      ping -c 5 [new-vpn-endpoint]
      Test admin SSH access through new VPN before killing openvpn-nat1
    - Gate: "Admin VPN accessible WITHOUT asa-prd-openvpn-nat1"

0.3 RDS cutover confirmation:
    - Confirm new RDS is primary and accepting writes:
      psql -h rds-use1-pzd-asa-totara-primary -U [user] -c "SELECT version();"
      psql -h rds-use1-pzd-asa-totara-primary-enc -U [user] -c "\l"
    - Gate: "New RDS confirmed live, asa-prddb-totara read traffic = 0"

0.4 Traffic verification on new ALB:
    - Confirm asa-prd-totara-alb serving 100% traffic:
      aws elbv2 describe-target-health \
        --target-group-arn [new-tg-arn] \
        --region us-east-1
    - Check old ALB has zero active connections:
      aws cloudwatch get-metric-statistics \
        --namespace AWS/ApplicationELB \
        --metric-name ActiveConnectionCount \
        --dimensions Name=LoadBalancer,Value=[ASA-PROD-ALB-suffix] \
        --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
        --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
        --period 300 --statistics Sum \
        --region us-east-1

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PHASE 1 — SI-386: DOCUMENT ALL RESOURCES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Generate AWS CLI commands to document EVERY resource below,
then output a Confluence-ready markdown inventory table and
a JSON file (asa-prd-stack1-inventory.json) for Terraform reference.

1.1 EC2 — Run for ALL 8 instances individually:

For each instance ID in:
(i-0a99cbb011eafd855, i-02c50177d74c3a135, i-01d7ae0109e7f49ef,
 i-0085011c8c32a3b42, i-07f5a02fc7e4d66d7, i-041331619200c83af,
 i-06cdb67d4971f152c, i-07068bf443ebac8bd)

Run:
# Full instance details
aws ec2 describe-instances \
  --instance-ids [INSTANCE_ID] \
  --region us-east-1 \
  --query 'Reservations[].Instances[].[
    InstanceId, InstanceType, State.Name,
    ImageId, KeyName, PrivateIpAddress, PublicIpAddress,
    SubnetId, VpcId, Placement.AvailabilityZone,
    IamInstanceProfile.Arn,
    SecurityGroups[*].[GroupId,GroupName]
  ]' --output table

# EBS volumes attached
aws ec2 describe-volumes \
  --filters Name=attachment.instance-id,Values=[INSTANCE_ID] \
  --region us-east-1 \
  --query 'Volumes[].[VolumeId,Size,VolumeType,
    Attachments[0].Device,SnapshotId,Encrypted]' \
  --output table

# Elastic IP
aws ec2 describe-addresses \
  --filters Name=instance-id,Values=[INSTANCE_ID] \
  --region us-east-1 \
  --query 'Addresses[].[PublicIp,AllocationId,AssociationId]' \
  --output table

# Security group rules
aws ec2 describe-security-groups \
  --filters Name=instance.group-id,Values=[SG_ID_FROM_ABOVE] \
  --region us-east-1 \
  --output json > sg-[INSTANCE_NAME].json

# Tags
aws ec2 describe-tags \
  --filters Name=resource-id,Values=[INSTANCE_ID] \
  --region us-east-1 --output table

1.2 ALB — ASA-PROD-ALB:

# Get ALB ARN and details
aws elbv2 describe-load-balancers \
  --names ASA-PROD-ALB \
  --region us-east-1 \
  --output json > alb-asa-prod-details.json

# Get all listeners
aws elbv2 describe-listeners \
  --load-balancer-arn [ALB_ARN] \
  --region us-east-1 --output json > alb-listeners.json

# Get all target groups
aws elbv2 describe-target-groups \
  --load-balancer-arn [ALB_ARN] \
  --region us-east-1 --output json > alb-target-groups.json

# Get target health for each TG
aws elbv2 describe-target-health \
  --target-group-arn [TG_ARN] \
  --region us-east-1 --output json

# Get SSL certificates
aws elbv2 describe-listener-certificates \
  --listener-arn [LISTENER_ARN] \
  --region us-east-1 --output json

# Route53 alias records pointing to old ALB DNS
aws route53 list-hosted-zones --output json | \
  jq -r '.HostedZones[].Id' | while read zone; do
    aws route53 list-resource-record-sets \
      --hosted-zone-id $zone \
      --query "ResourceRecordSets[?AliasTarget.DNSName!=null] |
               [?contains(AliasTarget.DNSName,'ASA-PROD-ALB')]" \
      --output json
  done

1.3 RDS — asa-prddb-totara:

# Full RDS details
aws rds describe-db-instances \
  --db-instance-identifier asa-prddb-totara \
  --region us-east-1 \
  --output json > rds-asa-prddb-totara-details.json

# Parameter group
aws rds describe-db-parameters \
  --db-parameter-group-name [PARAM_GROUP_FROM_ABOVE] \
  --region us-east-1 --output json > rds-parameter-group.json

# Existing snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier asa-prddb-totara \
  --region us-east-1 --output table

# Backup config
aws rds describe-db-instances \
  --db-instance-identifier asa-prddb-totara \
  --query 'DBInstances[].[BackupRetentionPeriod,
    PreferredBackupWindow,MultiAZ,StorageEncrypted]' \
  --region us-east-1 --output table

1.4 Network:

# VPC details
aws ec2 describe-vpcs \
  --filters Name=tag:Stack,Values=ASA-PRD-Stack1 \
  --region us-east-1 --output json > vpc-details.json

# Subnets
aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=[VPC_ID] \
  --region us-east-1 --output json > subnets.json

# Route tables
aws ec2 describe-route-tables \
  --filters Name=vpc-id,Values=[VPC_ID] \
  --region us-east-1 --output json > route-tables.json

# NAT Gateway
aws ec2 describe-nat-gateways \
  --filter Name=vpc-id,Values=[VPC_ID] \
  --region us-east-1 --output json

# Internet Gateway
aws ec2 describe-internet-gateways \
  --filters Name=attachment.vpc-id,Values=[VPC_ID] \
  --region us-east-1 --output json

OUTPUT REQUIRED — Confluence Inventory Table:
| Resource Type | Name | Old ID | Old Config | New Replacement | New ID | Snapshot ID | Status |
|---------------|------|--------|------------|-----------------|--------|-------------|--------|
| EC2 | asa-prd-openvpn-nat1 | i-0a99cbb011eafd855 | t3.micro | [new] | [new-id] | [ami-id] | RUNNING |
| EC2 | asa-prd-file1 | i-02c50177d74c3a135 | t3.large | [new] | [new-id] | [ami-id] | RUNNING |
| EC2 | asa-prd-application1 | i-01d7ae0109e7f49ef | m4.large | [new] | [new-id] | [ami-id] | STOPPED |
| EC2 | asa-prd-application2 | i-0085011c8c32a3b42 | m4.large | [new] | [new-id] | [ami-id] | STOPPED |
| EC2 | asa-prd-application3 | i-07f5a02fc7e4d66d7 | m4.large | [new] | [new-id] | [ami-id] | STOPPED |
| EC2 | asa-prd-application4 | i-041331619200c83af | m4.large | [new] | [new-id] | [ami-id] | STOPPED |
| EC2 | asa-prd-application5 | i-06cdb67d4971f152c | m4.large | [new] | [new-id] | [ami-id] | STOPPED |
| EC2 | asa-prd-application6 | i-07068bf443ebac8bd | m4.large | [new] | [new-id] | [ami-id] | STOPPED |
| ALB | ASA-PROD-ALB | [arn] | - | asa-prd-totara-alb | [arn] | N/A | ACTIVE |
| RDS | asa-prddb-totara | [arn] | db.m5.2xlarge/pg | rds-use1-pzd-asa-totara-primary | [arn] | [snap-id] | ACTIVE |
| VPC | ASA-PRD-VPC | [vpc-id] | [cidr] | Totara VPC | [new-vpc] | N/A | ACTIVE |

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PHASE 2 — SI-385: CHANGE CONTROL DOCUMENT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Generate a complete formal Change Control Request document:

CR TITLE: Decommission ASA-PRD-Stack1 Legacy Infrastructure
CR NUMBER: CR-[DATE]-ASA-DECOM
Account: 458797919735 | Region: us-east-1
Requested by: [Engineer Name]
Approved by: [Manager/CAB]

SECTION 1 — SCOPE OF CHANGE
Resources being deleted (all 10):
- 8x EC2 instances (listed above with IDs)
- 1x ALB: ASA-PROD-ALB
- 1x RDS: asa-prddb-totara (db.m5.2xlarge, postgres)
- Associated: EBS volumes, EIPs, SGs, IAM roles, Route53 records,
  NAT Gateway, Internet Gateway, VPC

SECTION 2 — RISK MATRIX
Generate table:
| Risk | Severity | Probability | Mitigation | Owner |
|------|----------|-------------|------------|-------|
| CMEIntegration.jar still running | HIGH | CONFIRMED | Stop process + verify replacement | Sahana Hemraj |
| openvpn-nat1 serving admin VPN | HIGH | MEDIUM | Confirm new VPN before termination | [Owner] |
| RDS cutover not confirmed | CRITICAL | LOW | Verify new RDS primary accepting writes | [DBA] |
| Stale ALB connections | MEDIUM | LOW | 300s drain + verify zero connections | [Owner] |
| Snapshot failure | HIGH | LOW | Verify snapshot status = available | [Owner] |
| Terraform destroy wrong order | CRITICAL | LOW | Follow exact sequence, state backup | [Owner] |

SECTION 3 — ROLLBACK PLAN (per resource)
EC2 Rollback:
  - Restore AMI: aws ec2 run-instances --image-id [ami-id] ...
  - All 8 instances have AMIs taken in Phase 1
  - EBS snapshots available for data volumes

RDS Rollback:
  - aws rds restore-db-instance-from-db-snapshot \
      --db-instance-identifier asa-prddb-totara-restored \
      --db-snapshot-identifier asa-prddb-totara-final-[DATE]
  - Repoint application connection strings

ALB Rollback:
  - Re-register EC2 targets to ASA-PROD-ALB
  - aws route53 change-resource-record-sets (repoint alias)
  - Lower Route53 TTL to 60s BEFORE change window

DNS Rollback:
  - Pre-staged Route53 revert JSON ready before window starts

SECTION 4 — MAINTENANCE WINDOW
Window: Sunday [DATE] 02:00 UTC — 06:00 UTC (4 hour window)
Rollback deadline: 05:00 UTC (1 hour buffer)
Communication:
  - Notify 5 days prior via email + Slack
  - Status updates every 30 minutes during window
  - Incident bridge open for duration

SECTION 5 — ACCEPTANCE CRITERIA (from SI-164)
□ All ASA-PRD-Stack1 EC2 instances terminated (all 8 confirmed)
□ ASA-PROD-ALB deleted
□ asa-prddb-totara snapshot taken AND instance deleted
□ Legacy VPC + all dependencies removed
□ AWS monthly cost reduction verified in Cost Explorer
□ No orphaned SGs, EIPs, IAM roles remain
□ Confluence runbook updated with completion status
□ CloudTrail audit log exported

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PHASE 3 — SI-164: FULL DECOMMISSION RUNBOOK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Generate exact AWS CLI + Terraform commands for each step.
Each step must include: command, expected output, go/no-go gate.

STEP 1 — FINAL SNAPSHOTS (hard stop if any fail)

# RDS Final Snapshot
DATE=$(date +%Y%m%d-%H%M)
aws rds create-db-snapshot \
  --db-instance-identifier asa-prddb-totara \
  --db-snapshot-identifier asa-prddb-totara-final-${DATE} \
  --region us-east-1

# Wait for RDS snapshot completion
aws rds wait db-snapshot-completed \
  --db-snapshot-identifier asa-prddb-totara-final-${DATE} \
  --region us-east-1
echo "✅ RDS Snapshot complete: asa-prddb-totara-final-${DATE}"

# AMI for asa-prd-openvpn-nat1 (RUNNING - no-reboot)
aws ec2 create-image \
  --instance-id i-0a99cbb011eafd855 \
  --name "asa-prd-openvpn-nat1-final-${DATE}" \
  --no-reboot \
  --region us-east-1

# AMI for asa-prd-file1 (RUNNING - no-reboot)
aws ec2 create-image \
  --instance-id i-02c50177d74c3a135 \
  --name "asa-prd-file1-final-${DATE}" \
  --no-reboot \
  --region us-east-1

# EBS Snapshots for ALL 8 instances data volumes
for INSTANCE_ID in \
  i-0a99cbb011eafd855 \
  i-02c50177d74c3a135 \
  i-01d7ae0109e7f49ef \
  i-0085011c8c32a3b42 \
  i-07f5a02fc7e4d66d7 \
  i-041331619200c83af \
  i-06cdb67d4971f152c \
  i-07068bf443ebac8bd; do
  VOL_IDS=$(aws ec2 describe-volumes \
    --filters Name=attachment.instance-id,Values=${INSTANCE_ID} \
    --query 'Volumes[].VolumeId' --output text --region us-east-1)
  for VOL_ID in $VOL_IDS; do
    aws ec2 create-snapshot \
      --volume-id ${VOL_ID} \
      --description "Final snapshot ${INSTANCE_ID} ${DATE}" \
      --region us-east-1
    echo "Snapshot created for ${VOL_ID} (${INSTANCE_ID})"
  done
done

# Check EFS - if attached to legacy stack
aws efs describe-file-systems \
  --query 'FileSystems[?Tags[?Key==`Stack` && Value==`ASA-PRD-Stack1`]]' \
  --region us-east-1 --output json

# ✅ GO/NO-GO GATE 1:
# All AMI states = available
# RDS snapshot status = available
# All EBS snapshots status = completed
# ABORT if any snapshot failed

STEP 2 — DRAIN AND DISABLE

# Stop CMEIntegration.jar FIRST on asa-prd-file1
# (SSH into asa-prd-file1: i-02c50177d74c3a135)
aws ssm send-command \
  --instance-ids i-02c50177d74c3a135 \
  --document-name AWS-RunShellScript \
  --parameters commands=["systemctl stop cme-integration",
    "kill -15 1018785",
    "sleep 30",
    "ps aux | grep CMEIntegration"] \
  --region us-east-1

# Deregister all targets from ASA-PROD-ALB
# Get all target group ARNs first
TG_ARNS=$(aws elbv2 describe-target-groups \
  --load-balancer-arn [ALB_ARN] \
  --query 'TargetGroups[].TargetGroupArn' \
  --output text --region us-east-1)

# For each TG, deregister targets
for TG_ARN in $TG_ARNS; do
  TARGETS=$(aws elbv2 describe-target-health \
    --target-group-arn ${TG_ARN} \
    --query 'TargetHealthDescriptions[].Target' \
    --output json --region us-east-1)
  aws elbv2 deregister-targets \
    --target-group-arn ${TG_ARN} \
    --targets ${TARGETS} \
    --region us-east-1
  echo "Targets deregistered from ${TG_ARN}"
done

# Wait 300 seconds for connection draining
echo "Waiting 300s for connection drain..."
sleep 300

# Stop ALL 8 instances
aws ec2 stop-instances \
  --instance-ids \
    i-0a99cbb011eafd855 \
    i-02c50177d74c3a135 \
    i-01d7ae0109e7f49ef \
    i-0085011c8c32a3b42 \
    i-07f5a02fc7e4d66d7 \
    i-041331619200c83af \
    i-06cdb67d4971f152c \
    i-07068bf443ebac8bd \
  --region us-east-1

# Wait for all to stop
aws ec2 wait instance-stopped \
  --instance-ids \
    i-0a99cbb011eafd855 \
    i-02c50177d74c3a135 \
    i-01d7ae0109e7f49ef \
    i-0085011c8c32a3b42 \
    i-07f5a02fc7e4d66d7 \
    i-041331619200c83af \
    i-06cdb67d4971f152c \
    i-07068bf443ebac8bd \
  --region us-east-1
echo "✅ All 8 instances stopped"

# ✅ GO/NO-GO GATE 2:
# ALB active connection count = 0
# All 8 instances state = stopped
# CMEIntegration.jar process not running
# HOLD 24 hours cooldown observation period

STEP 3 — PRE-DESTROY VERIFICATION GATES

# Gate A: New RDS accepting connections
aws rds describe-db-instances \
  --db-instance-identifier rds-use1-pzd-asa-totara-primary \
  --query 'DBInstances[].[DBInstanceStatus,Endpoint.Address]' \
  --region us-east-1

# Gate B: New ALB healthy targets
aws elbv2 describe-target-health \
  --target-group-arn [NEW_TG_ARN] \
  --region us-east-1 \
  --query 'TargetHealthDescriptions[].TargetHealth.State'

# Gate C: No Route53 records pointing to old ALB
aws route53 list-resource-record-sets \
  --hosted-zone-id [ZONE_ID] \
  --query "ResourceRecordSets[?contains(AliasTarget.DNSName || '',
           'ASA-PROD-ALB')]" \
  --output json

# Gate D: Old ALB zero connections (15 min window)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=[ALB_SUFFIX] \
  --start-time $(date -u -d '15 min ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 900 --statistics Sum --region us-east-1

# ✅ ALL 4 GATES MUST PASS BEFORE STEP 4
# ABORT and rollback if any gate fails

STEP 4 — TERRAFORM DESTROY (exact sequence)

# CRITICAL: Backup Terraform state first
aws s3 cp s3://[tfstate-bucket]/asa-prd-stack1/terraform.tfstate \
  s3://[tfstate-bucket]/asa-prd-stack1/terraform.tfstate.backup-${DATE}

# Destroy in this EXACT order (dependencies matter):
# 1. Application servers (all 6)
terraform destroy -target=aws_instance.asa_prd_application1 -auto-approve
terraform destroy -target=aws_instance.asa_prd_application2 -auto-approve
terraform destroy -target=aws_instance.asa_prd_application3 -auto-approve
terraform destroy -target=aws_instance.asa_prd_application4 -auto-approve
terraform destroy -target=aws_instance.asa_prd_application5 -auto-approve
terraform destroy -target=aws_instance.asa_prd_application6 -auto-approve

# Verify each terminated before next:
aws ec2 describe-instances \
  --instance-ids i-01d7ae0109e7f49ef i-0085011c8c32a3b42 \
    i-07f5a02fc7e4d66d7 i-041331619200c83af \
    i-06cdb67d4971f152c i-07068bf443ebac8bd \
  --query 'Reservations[].Instances[].State.Name' \
  --region us-east-1

# 2. File server
terraform destroy -target=aws_instance.asa_prd_file1 -auto-approve

# 3. OpenVPN/NAT (LAST of EC2 - admin VPN must be confirmed via new path)
terraform destroy -target=aws_instance.asa_prd_openvpn_nat1 -auto-approve

# 4. ALB + target groups + listeners
terraform destroy -target=aws_alb.asa_prod_alb -auto-approve
terraform destroy -target=aws_alb_target_group.asa_prod -auto-approve
terraform destroy -target=aws_alb_listener.asa_prod -auto-approve

# 5. EFS
terraform destroy -target=aws_efs_file_system.asa_prd -auto-approve

# 6. ElastiCache
terraform destroy -target=aws_elasticache_cluster.asa_prd -auto-approve

# 7. RDS (snapshot already confirmed in Step 1)
terraform destroy -target=aws_db_instance.asa_prddb_totara \
  -var="skip_final_snapshot=true" -auto-approve

# 8. VPC + all networking (must be last)
terraform destroy -target=aws_vpc.asa_prd -auto-approve

# ✅ GO/NO-GO GATE 4:
# All resources show terminated/deleted state
# No Terraform errors
# Terraform state shows 0 resources

STEP 5 — MANUAL CLEANUP

# Find orphaned security groups (no ENI attached)
aws ec2 describe-security-groups \
  --filters Name=vpc-id,Values=[VPC_ID] \
  --query 'SecurityGroups[].GroupId' \
  --output text --region us-east-1 | tr '\t' '\n' | while read SG; do
  ENI=$(aws ec2 describe-network-interfaces \
    --filters Name=group-id,Values=${SG} \
    --query 'NetworkInterfaces[].NetworkInterfaceId' \
    --output text --region us-east-1)
  [ -z "$ENI" ] && echo "ORPHANED SG: ${SG}" && \
    aws ec2 delete-security-group --group-id ${SG} --region us-east-1
done

# Release unassociated Elastic IPs
aws ec2 describe-addresses \
  --filters Name=domain,Values=vpc \
  --query 'Addresses[?AssociationId==null].[AllocationId,PublicIp]' \
  --output text --region us-east-1 | while read ALLOC_ID IP; do
  echo "Releasing EIP: ${IP} (${ALLOC_ID})"
  aws ec2 release-address --allocation-id ${ALLOC_ID} --region us-east-1
done

# Delete stale Route53 records
aws route53 list-resource-record-sets \
  --hosted-zone-id [ZONE_ID] \
  --query "ResourceRecordSets[?contains(Name,'asa-prd')]" \
  --output json

# Delete unused IAM roles with asa-prd prefix
aws iam list-roles \
  --query 'Roles[?contains(RoleName,`asa-prd`)].RoleName' \
  --output text | tr '\t' '\n' | while read ROLE; do
  LAST_USED=$(aws iam get-role --role-name ${ROLE} \
    --query 'Role.RoleLastUsed.LastUsedDate' --output text)
  echo "Role: ${ROLE} | Last used: ${LAST_USED}"
  # Manually confirm before deleting each role
done

# CloudWatch log groups cleanup
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/ec2/asa-prd" \
  --region us-east-1 \
  --query 'logGroups[].logGroupName' \
  --output text | tr '\t' '\n' | while read LG; do
  aws logs delete-log-group --log-group-name ${LG} --region us-east-1
  echo "Deleted log group: ${LG}"
done

# S3 buckets tagged with old stack
aws s3api list-buckets \
  --query 'Buckets[].Name' --output text | tr '\t' '\n' | while read BUCKET; do
  TAGS=$(aws s3api get-bucket-tagging \
    --bucket ${BUCKET} 2>/dev/null | \
    jq -r '.TagSet[] | select(.Key=="Stack" and .Value=="ASA-PRD-Stack1")')
  [ -n "$TAGS" ] && echo "Stack bucket found: ${BUCKET}"
done

STEP 6 — FINAL VERIFICATION

# Cost Explorer - confirm $0 for old stack tag
aws ce get-cost-and-usage \
  --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --filter '{"Tags":{"Key":"Stack","Values":["ASA-PRD-Stack1"]}}' \
  --metrics BlendedCost \
  --region us-east-1

# CloudTrail - export deletion audit log
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=TerminateInstances \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%S) \
  --region us-east-1 --output json > cloudtrail-decom-audit-${DATE}.json

aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteDBInstance \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%S) \
  --region us-east-1 --output json >> cloudtrail-decom-audit-${DATE}.json

# Final resource check - confirm nothing left
aws ec2 describe-instances \
  --filters Name=tag:Stack,Values=ASA-PRD-Stack1 \
  --query 'Reservations[].Instances[?State.Name!=`terminated`]' \
  --region us-east-1

# Expected output: [] (empty array)
# If not empty: STOP and investigate

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FINAL ACCEPTANCE CRITERIA SIGN-OFF
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

□ All 8 EC2 instances terminated:
  □ i-0a99cbb011eafd855 (asa-prd-openvpn-nat1)
  □ i-02c50177d74c3a135 (asa-prd-file1)
  □ i-01d7ae0109e7f49ef (asa-prd-application1)
  □ i-0085011c8c32a3b42 (asa-prd-application2)
  □ i-07f5a02fc7e4d66d7 (asa-prd-application3)
  □ i-041331619200c83af (asa-prd-application4)
  □ i-06cdb67d4971f152c (asa-prd-application5)
  □ i-07068bf443ebac8bd (asa-prd-application6)
□ ASA-PROD-ALB deleted
□ asa-prddb-totara snapshot confirmed + instance deleted
□ Legacy VPC deleted
□ Zero orphaned SGs, EIPs, IAM roles
□ CloudTrail audit exported
□ Cost Explorer shows $0 for ASA-PRD-Stack1 tag
□ Confluence runbook updated with completion + snapshot IDs
