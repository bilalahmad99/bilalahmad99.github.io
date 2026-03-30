---
layout: default
---

## SOC 2 and ISO 27001: Notes from personal experience

What SOC 2 and ISO 27001 actually mean for SREs, the technical controls you need to implement, and how to maintain compliance without losing your mind.

### SOC 2 vs ISO 27001: What's the Difference?

Both are security frameworks, but they serve different purposes and markets:

**SOC 2 (Service Organization Control 2)**
- **Origin**: American (AICPA)
- **Focus**: Service providers handling customer data
- **Scope**: Five Trust Service Criteria
- **Market**: Required by US enterprise customers
- **Audit**: External auditor issues report
- **Duration**: 6-12 months for Type II
- **Cost**: $30,000 - $100,000+

**ISO 27001**
- **Origin**: International (ISO/IEC)
- **Focus**: Information Security Management System (ISMS)
- **Scope**: 114 controls across 14 domains
- **Market**: Global, especially Europe and Asia
- **Audit**: Certification body issues certificate
- **Duration**: 3-year certificate with annual surveillance
- **Cost**: $50,000 - $150,000+

**Key Difference**: SOC 2 is a report showing you have controls. ISO 27001 is a certification proving you have a management system.

### Which One Do You Need?

**Get SOC 2 if:**
- Selling to US enterprise customers
- SaaS or cloud service provider
- Customers explicitly ask for SOC 2
- Want faster time to compliance (6-12 months)

**Get ISO 27001 if:**
- Selling internationally (especially EU, UK, Asia)
- Government or defense contracts
- Want a globally recognized certification
- Need to demonstrate GDPR compliance

**Get Both if:**
- Selling to global enterprise customers
- Want maximum market coverage
- Have the budget and resources
- Controls overlap significantly (60-70%)

### Control Mapping: SOC 2 to ISO 27001

Good news: most controls overlap. Implement once, satisfy both frameworks:

| SOC 2 Criteria | ISO 27001 Controls | Implementation |
|----------------|-------------------|----------------|
| Security - Access Control | A.9 Access Control | MFA, RBAC, least privilege |
| Security - Encryption | A.10 Cryptography | TLS, encryption at rest |
| Security - Logging | A.12.4 Logging and monitoring | Centralized logs, SIEM |
| Availability - Monitoring | A.12.1 Operational procedures | Prometheus, Datadog |
| Availability - Backup | A.12.3 Backup | Automated backups, DR testing |
| Security - Vulnerability Mgmt | A.12.6 Technical vulnerability | Scanning, patching |
| Security - Incident Response | A.16 Incident management | On-call, runbooks, post-mortems |
| Security - Change Management | A.12.1.2 Change management | GitOps, approval workflows |
| Confidentiality - Network Security | A.13 Communications security | Firewalls, network segmentation |
| Security - Physical Security | A.11 Physical security | Data center controls (cloud provider) |

### SOC 2 Type I vs Type II

**Type I**: Point-in-time assessment
- Evaluates if controls are properly designed
- Snapshot of your security posture
- Faster and cheaper
- Less valuable to customers

**Type II**: Period-of-time assessment
- Evaluates if controls operate effectively over 6-12 months
- Requires continuous evidence collection
- More rigorous and expensive
- What most enterprise customers require

### ISO 27001 Certification Process

**Stage 1 Audit**: Documentation review
- ISMS policies and procedures
- Risk assessment and treatment plan
- Statement of Applicability (SoA)
- Internal audit results

**Stage 2 Audit**: Implementation verification
- Control effectiveness testing
- Employee interviews
- Technical testing
- Evidence review

**Surveillance Audits**: Annual reviews
- Verify controls still operate
- Review changes to ISMS
- Check corrective actions
- Maintain certification

### Technical Controls SREs Must Implement

#### 1. Access Control and Authentication

**Requirements (Both Frameworks):**
- Multi-factor authentication (MFA) for all production access
- Role-based access control (RBAC)
- Principle of least privilege
- Regular access reviews and deprovisioning

**SOC 2 Focus**: Proving access is restricted and monitored
**ISO 27001 Focus**: Documented access control policy and procedures

**Implementation:**
```bash
# AWS IAM policy example - least privilege
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeSecurityGroups"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}

# Enforce MFA for AWS CLI access
aws iam create-virtual-mfa-device --virtual-mfa-device-name MyMFADevice

# Kubernetes RBAC - read-only access
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

**Evidence Collection:**
- **SOC 2**: Quarterly access review reports, MFA enrollment logs
- **ISO 27001**: Access control policy, access review procedures, training records

#### 2. Logging and Monitoring

**Requirements (Both Frameworks):**
- Centralized log aggregation
- Log retention (typically 1 year minimum)
- Real-time alerting for security events
- Audit trails for all administrative actions

**SOC 2 Focus**: Demonstrating logs are collected and retained
**ISO 27001 Focus**: Log review procedures and incident detection

**Implementation:**
```yaml
# CloudWatch Logs retention
aws logs put-retention-policy \
  --log-group-name /aws/lambda/production \
  --retention-in-days 365

# Kubernetes audit logging
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "rolebindings"]

# Prometheus alerting rules
groups:
- name: security_alerts
  rules:
  - alert: UnauthorizedAccessAttempt
    expr: rate(http_requests_total{status="401"}[5m]) > 10
    for: 5m
    annotations:
      summary: "High rate of 401 errors detected"
```

**Evidence Collection:**
- **SOC 2**: Log retention screenshots, alert configurations, sample logs
- **ISO 27001**: Logging policy, log review procedures, incident response logs

#### 3. Encryption

**Requirements (Both Frameworks):**
- Encryption at rest for all sensitive data
- Encryption in transit (TLS 1.2+)
- Key management and rotation
- Certificate management

**SOC 2 Focus**: Proving data is encrypted
**ISO 27001 Focus**: Cryptographic controls policy and key management procedures

**Implementation:**
```bash
# AWS KMS key creation and rotation
aws kms create-key \
  --description "Production data encryption key" \
  --key-policy file://key-policy.json

aws kms enable-key-rotation --key-id <key-id>

# RDS encryption at rest
aws rds create-db-instance \
  --db-instance-identifier prod-db \
  --storage-encrypted \
  --kms-key-id <kms-key-id>

# Enforce TLS in nginx
server {
    listen 443 ssl http2;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
}
```

**Evidence Collection:**
- **SOC 2**: Encryption configuration screenshots, key rotation logs
- **ISO 27001**: Cryptographic policy, key management procedures, certificate inventory

#### 4. Network Security

**Requirements (Both Frameworks):**
- Network segmentation
- Firewall rules and security groups
- VPN or bastion hosts for production access
- DDoS protection

**SOC 2 Focus**: Network controls are in place and monitored
**ISO 27001 Focus**: Network security policy and architecture documentation

**Implementation:**
```bash
# AWS Security Group - restrictive ingress
aws ec2 create-security-group \
  --group-name production-app \
  --description "Production application security group" \
  --vpc-id vpc-xxxxx

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 443 \
  --source-group sg-alb-xxxxx

# Kubernetes Network Policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Evidence Collection:**
- **SOC 2**: Network diagrams, security group configs, VPN logs
- **ISO 27001**: Network security policy, architecture documentation, change logs

#### 5. Vulnerability Management

**Requirements (Both Frameworks):**
- Regular vulnerability scanning
- Patch management process
- Penetration testing (annual minimum)
- Remediation tracking

**SOC 2 Focus**: Vulnerabilities are identified and remediated
**ISO 27001 Focus**: Vulnerability management process and risk treatment

**Implementation:**
```bash
# Container image scanning with Trivy
trivy image --severity HIGH,CRITICAL nginx:latest

# Automated patching with AWS Systems Manager
aws ssm create-patch-baseline \
  --name production-baseline \
  --operating-system AMAZON_LINUX_2 \
  --approval-rules 'PatchRules=[{PatchFilterGroup={PatchFilters=[{Key=CLASSIFICATION,Values=[Security]}]},ApproveAfterDays=7}]'
```

**Evidence Collection:**
- **SOC 2**: Scan reports, patch logs, penetration test reports
- **ISO 27001**: Vulnerability management procedure, risk register, remediation tracking

#### 6. Backup and Disaster Recovery

**Requirements (Both Frameworks):**
- Regular automated backups
- Backup testing and restoration procedures
- Documented RTO/RPO
- Offsite backup storage

**SOC 2 Focus**: Backups work and are tested
**ISO 27001 Focus**: Business continuity and disaster recovery plans

**Implementation:**
```bash
# AWS Backup plan
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "production-daily",
  "Rules": [{
    "RuleName": "daily-backup",
    "TargetBackupVaultName": "production-vault",
    "ScheduleExpression": "cron(0 2 * * ? *)",
    "StartWindowMinutes": 60,
    "CompletionWindowMinutes": 120,
    "Lifecycle": {
      "DeleteAfterDays": 30,
      "MoveToColdStorageAfterDays": 7
    }
  }]
}'

# Velero for Kubernetes backup
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 720h0m0s
```

**Evidence Collection:**
- **SOC 2**: Backup logs, restoration test reports
- **ISO 27001**: BC/DR plan, RTO/RPO documentation, test results

#### 7. Change Management

**Requirements (Both Frameworks):**
- Documented change approval process
- Change tracking and audit trail
- Rollback procedures
- Separation of duties

**SOC 2 Focus**: Changes are approved and tracked
**ISO 27001 Focus**: Change management procedure and risk assessment

**Implementation:**
```yaml
# GitOps with ArgoCD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-app
spec:
  project: production
  source:
    repoURL: https://github.com/company/infrastructure
    targetRevision: main
    path: k8s/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
```

**Evidence Collection:**
- **SOC 2**: Pull request history, deployment logs
- **ISO 27001**: Change management procedure, change records, risk assessments

#### 8. Incident Response

**Requirements (Both Frameworks):**
- Documented incident response plan
- On-call rotation and escalation procedures
- Incident tracking and post-mortems
- Security incident procedures

**SOC 2 Focus**: Incidents are detected, tracked, and resolved
**ISO 27001 Focus**: Incident management process and continuous improvement

**Implementation:**
```yaml
# Incident response runbook template
## Incident: [TITLE]
**Severity**: P1 / P2 / P3 / P4
**Detected**: YYYY-MM-DD HH:MM UTC
**Resolved**: YYYY-MM-DD HH:MM UTC

### Timeline
- HH:MM - Incident detected
- HH:MM - On-call paged
- HH:MM - Root cause identified
- HH:MM - Fix deployed
- HH:MM - Resolved

### Root Cause
[Technical explanation]

### Impact
- Users affected: X
- Services impacted: [list]

### Action Items
- [ ] Implement monitoring
- [ ] Update runbook
```

**Evidence Collection:**
- **SOC 2**: Incident tickets, post-mortems, on-call schedules
- **ISO 27001**: Incident response procedure, incident register, lessons learned

### ISO 27001 Specific Requirements

#### Risk Assessment and Treatment

ISO 27001 requires a formal risk assessment process:

**Risk Assessment Process:**
1. **Identify Assets**: List all information assets (databases, servers, applications)
2. **Identify Threats**: What could go wrong (data breach, outage, ransomware)
3. **Identify Vulnerabilities**: Weaknesses that could be exploited
4. **Assess Impact**: What happens if the threat materializes
5. **Assess Likelihood**: How likely is this to happen
6. **Calculate Risk**: Impact × Likelihood = Risk Score

**Risk Treatment Options:**
- **Mitigate**: Implement controls to reduce risk
- **Accept**: Document why risk is acceptable
- **Transfer**: Insurance or third-party responsibility
- **Avoid**: Stop the risky activity

**Statement of Applicability (SoA):**
Document which of the 114 Annex A controls apply to your organization and why.

```markdown
# Example SoA Entry

## A.9.2.1 User registration and de-registration

**Applicable**: Yes

**Justification**: We manage user access to production systems and must ensure proper provisioning and deprovisioning.

**Implementation**: 
- Automated user provisioning via Okta
- Quarterly access reviews
- Automated deprovisioning on termination
- RBAC with least privilege

**Evidence**:
- Access review reports
- Provisioning/deprovisioning logs
- RBAC policy documentation
```

#### Internal Audits

ISO 27001 requires regular internal audits:

**Audit Schedule:**
- Full ISMS audit: Annually
- Critical controls: Quarterly
- High-risk areas: As needed

**Audit Process:**
1. Plan audit scope and schedule
2. Review documentation
3. Interview control owners
4. Test control effectiveness
5. Document findings
6. Track corrective actions

#### Management Review

ISO 27001 requires management to review the ISMS:

**Review Frequency**: At least annually

**Review Topics:**
- Internal audit results
- Incident trends
- Risk assessment updates
- Control effectiveness
- Improvement opportunities
- Resource needs

### Continuous Compliance: Automation is Key

Manual compliance is unsustainable. Automate evidence collection:

```python
# Automated compliance evidence collector
import boto3
import json
from datetime import datetime

def collect_iam_evidence():
    """Collect IAM access review evidence"""
    iam = boto3.client('iam')
    users = iam.list_users()['Users']
    
    evidence = []
    for user in users:
        mfa_devices = iam.list_mfa_devices(UserName=user['UserName'])
        
        evidence.append({
            'UserName': user['UserName'],
            'MFAEnabled': len(mfa_devices['MFADevices']) > 0,
            'CreatedDate': user['CreateDate'].isoformat()
        })
    
    return evidence

def collect_encryption_evidence():
    """Collect encryption configuration evidence"""
    s3 = boto3.client('s3')
    
    evidence = {'s3_buckets': []}
    buckets = s3.list_buckets()['Buckets']
    
    for bucket in buckets:
        try:
            encryption = s3.get_bucket_encryption(Bucket=bucket['Name'])
            evidence['s3_buckets'].append({
                'Bucket': bucket['Name'],
                'Encrypted': True
            })
        except:
            evidence['s3_buckets'].append({
                'Bucket': bucket['Name'],
                'Encrypted': False
            })
    
    return evidence

def generate_compliance_report():
    """Generate monthly compliance report"""
    report = {
        'report_date': datetime.now().isoformat(),
        'iam_evidence': collect_iam_evidence(),
        'encryption_evidence': collect_encryption_evidence()
    }
    
    # Save to S3 for auditor access
    s3 = boto3.client('s3')
    s3.put_object(
        Bucket='compliance-evidence',
        Key=f'reports/{datetime.now().strftime("%Y-%m")}/report.json',
        Body=json.dumps(report, indent=2)
    )
    
    return report
```

### Infrastructure as Code for Compliance

Use IaC to enforce compliance controls:

```hcl
# Terraform module for compliant S3 bucket
module "compliant_s3_bucket" {
  source = "./modules/s3-compliant"
  
  bucket_name = "production-data"
  
  # Encryption required (SOC 2 + ISO 27001)
  enable_encryption = true
  kms_key_id       = aws_kms_key.production.id
  
  # Versioning required (SOC 2 + ISO 27001)
  enable_versioning = true
  
  # Logging required (SOC 2 + ISO 27001)
  enable_logging = true
  log_bucket     = "audit-logs"
  
  # Block public access (SOC 2 + ISO 27001)
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
  
  tags = {
    Compliance = "SOC2-ISO27001"
    Environment = "Production"
  }
}
```

### The Audit Process: What to Expect

#### SOC 2 Audit Timeline

**Pre-Audit (2-3 months):**
- Gap assessment
- Evidence collection
- Documentation
- Mock audit

**Audit (2-4 weeks):**
- Kickoff meeting
- Control testing
- Interviews
- Observation
- Remediation

**Post-Audit:**
- Draft report review
- Final report issued
- Continuous monitoring

#### ISO 27001 Audit Timeline

**Pre-Certification (6-12 months):**
- ISMS implementation
- Risk assessment
- Internal audits
- Management review

**Stage 1 Audit (1-2 days):**
- Documentation review
- ISMS scope verification
- Readiness assessment

**Stage 2 Audit (3-5 days):**
- Control testing
- Site visits
- Employee interviews
- Technical verification

**Post-Certification:**
- Certificate issued (3 years)
- Annual surveillance audits
- Recertification every 3 years

### Common Audit Findings and How to Avoid Them

**Finding**: Insufficient access reviews
**Fix**: Automate quarterly reviews with evidence collection
**Applies to**: Both SOC 2 and ISO 27001

**Finding**: Incomplete logging
**Fix**: Centralize all logs with proper retention
**Applies to**: Both SOC 2 and ISO 27001

**Finding**: Missing MFA on production accounts
**Fix**: Enforce MFA via IAM policies
**Applies to**: Both SOC 2 and ISO 27001

**Finding**: No risk assessment (ISO 27001 specific)
**Fix**: Conduct formal risk assessment with documented methodology

**Finding**: Missing internal audits (ISO 27001 specific)
**Fix**: Schedule and conduct internal audits quarterly

**Finding**: No management review (ISO 27001 specific)
**Fix**: Hold annual management review meetings with documentation

### Dual Compliance Strategy

If pursuing both, optimize your approach:

**Phase 1: Foundation (Months 1-3)**
- Implement overlapping controls first
- Set up logging and monitoring
- Establish access controls
- Deploy encryption

**Phase 2: SOC 2 Prep (Months 4-6)**
- Focus on evidence collection
- Document control operation
- Prepare for Type II observation period

**Phase 3: ISO 27001 Prep (Months 7-12)**
- Conduct risk assessment
- Create Statement of Applicability
- Perform internal audits
- Hold management review

**Phase 4: Audits (Months 13-15)**
- SOC 2 Type II audit
- ISO 27001 Stage 1 and 2 audits
- Address findings

**Phase 5: Maintenance (Ongoing)**
- Continuous evidence collection
- Annual surveillance audits
- Regular internal audits
- Management reviews

### Lessons Learned

**1. Start with Overlapping Controls**
Implement controls that satisfy both frameworks first. You'll get 60-70% coverage for both.

**2. Automate Everything**
Manual compliance doesn't scale. Invest in automation and IaC from day one.

**3. Use Compliance Platforms**
Tools like Vanta and Drata support both SOC 2 and ISO 27001, saving hundreds of hours.

**4. Document as You Go**
Don't wait until audit prep. Document procedures when you create them.

**5. Get Executive Buy-In**
Both frameworks require significant investment. Make sure leadership understands cost and timeline.

**6. Treat It as Security Improvement**
These frameworks force you to implement controls that prevent outages and breaches.

[back](../)
