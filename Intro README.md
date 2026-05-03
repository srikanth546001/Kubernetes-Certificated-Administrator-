You are an expert cloud architect and DevOps engineer.

I have an architecture decision document (ADR) and implementation details for an environment-based deployment system (Production, Staging, Development). 

Your task is to:

1. Understand the full architecture and extract:
   - Design decisions
   - Environment structures
   - Resource allocation strategy
   - CloudFormation structure
   - Migration plan
   - Validation requirements

2. Break everything down into:
   - Clear explanation (simple + advanced)
   - Step-by-step implementation plan
   - Terraform / CloudFormation actionable changes
   - Validation checklist
   - Automation scripts (where applicable)

3. Ensure NOTHING is missed from below content.

---

### 📌 ARCHITECTURE SUMMARY

ADR-002: Environment-Based Architecture for Synegen LMS

Decision:
- Replace tier-based architecture (A/B/C/D/F tiers) with environment-based:
  - Production
  - Staging
  - Development

- Scaling handled via instance sizing, NOT architecture tiers.

---

### 📌 ENVIRONMENT STRUCTURE

Production:
- Multi-AZ (2–3 AZs)
- Auto-scaling: 2–50 instances
- Instance types: t3.medium → t3.2xlarge
- Database: Aurora PostgreSQL (provisioned/serverless)
- Caching: Full ElastiCache
- Monitoring: Full CloudWatch (dashboards, alarms, logs)
- Backup: Automated + PITR
- CDN: Optional CloudFront
- Networking: SGs, NACLs, WAF

Cost: $400–$1500/month

---

Staging:
- 2 AZ deployment
- Auto-scaling: 1–10 instances
- Instance types: t3.small → t3.large
- Database: Aurora Serverless v2 or single instance
- Caching: Basic ElastiCache
- Monitoring: Standard CloudWatch
- Backup: 7-day retention
- CDN: None
- Networking: Simplified SGs

Cost: $200–$600/month

---

Development:
- Single AZ
- Auto-scaling: 1–2 instances
- Instance types: t3.micro → t3.medium
- Database: Aurora Serverless v2 (scale-to-zero)
- Caching: Optional / none
- Monitoring: Basic CloudWatch
- Backup: Manual only
- CDN: None
- Networking: Dev access SGs

Cost: $100–$300/month

---

### 📌 INSTANCE SIZING

Concurrent Users:
- 10–100 → t3.micro/small, db.t3.small
- 100–500 → t3.small/medium, db.t3.medium
- 500–1000 → t3.medium/large, db.t3.large
- 1000–5000 → t3.large/xlarge, db.r6g.large
- 5000+ → t3.xlarge/2xlarge, db.r6g.xlarge+

---

### 📌 INSTANCE TYPES

t3 (General Purpose):
- Burstable CPU
- 4GB RAM per vCPU
- Best cost/performance
- Suitable for web apps

r6g (Database):
- Memory optimized
- ARM (Graviton2)
- Better for PostgreSQL
- High throughput

---

### 📌 CLOUDFORMATION STRUCTURE

templates/
- vpc-baseline.yaml
- totara-application.yaml
- aspire-application.yaml
- 01-newvpc.yaml
- 02-securitygroups.yaml
- 03-rds.yaml

parameters/
- production/
  - client-app-prd.json
  - client-app-prd-vpc.json
- staging/
  - client-app-stg.json
  - client-app-stg-vpc.json
- development/
  - client-app-dev.json
  - client-app-dev-vpc.json

---

### 📌 CONFIGURATION MANAGEMENT

- Environment-based parameter files
- Client-specific overrides inherit from base
- Naming: remove tier codes → use env (prd/stg/dev)
- Tagging: environment-based tags
- Validation: environment-aware templates

---

### 📌 SUCCESS METRICS

- 50% reduction in parameter variations
- Deployment < 30 minutes
- 95% resource utilization
- >90% client satisfaction
- 60% reduction in maintenance effort

---

### 📌 MIGRATION STRATEGY

Phase 1 (Week 1):
- Update README
- Revise ADR
- Update runbooks
- Create sizing guide

Phase 2 (Week 2):
- Restructure parameters (tier → env)
- Create base configs
- Migrate clients
- Maintain backward compatibility

Phase 3 (Week 3):
- Remove tier code
- Update tagging
- Validate templates
- Update naming

Phase 4 (Week 4):
- Deploy test stacks
- Validate cost + performance
- Finalize docs

Phase 5:
- Gradual client migration
- No forced migration
- New clients use new system

---

### 📌 VALIDATION TASK (TASK-18)

Goal:
Validate AWS Organization Structure

Acceptance Criteria:
- All OUs exist:
  - Core
  - Clients
  - Suspended
  - Nested OUs
- OU hierarchy matches ADR-022
- No accounts in root (except management)
- OU IDs documented
- Screenshot saved

---

### 📌 WHAT YOU MUST DO

1. Explain the architecture in simple terms
2. Provide DevOps implementation steps
3. Give Terraform / CloudFormation changes
4. Provide validation scripts (CLI / automation)
5. Provide checklist for TASK-18
6. Suggest improvements / risks
7. Create production-ready documentation

Do NOT skip any section.
Keep output structured and actionable.
