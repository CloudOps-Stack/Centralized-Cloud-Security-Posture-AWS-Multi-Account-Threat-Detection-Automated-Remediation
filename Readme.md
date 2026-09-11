<img width="1883" height="810" alt="Screenshot 2026-09-11 123417" src="https://github.com/user-attachments/assets/560c34b5-0ed8-48c2-9f00-403009ad005e" /># Centralized Cloud Security Posture
### Security Hub · GuardDuty · Inspector · IAM Access Analyzer · AWS Organizations

A production-grade centralized security posture setup built on top of an existing AWS Landing Zone, enabling organization-wide threat detection, vulnerability management, and access analysis — all aggregated into a single Audit account using AWS delegated administrator patterns.

---

## What This Project Covers

- Amazon GuardDuty enabled org-wide with Audit account as delegated administrator
- AWS Security Hub enabled org-wide with FSBP, CIS, and NIST standards enforced
- Amazon Inspector enabled org-wide for continuous EC2, ECR, and Lambda vulnerability scanning
- IAM Access Analyzer enabled org-wide to detect unintended external resource access
- All findings aggregated into the Audit account — single pane of glass for the entire organization
<img width="876" height="1420" alt="_- visual selection" src="https://github.com/user-attachments/assets/4e507e00-a87e-4060-9ec5-4493c981c041" />

---

## Prerequisites

This project is a direct follow-on to the AWS Multi-Account Landing Zone. Before starting:

- [ ] Landing Zone is active — Control Tower status shows **Active**
- [ ] All 5 accounts are provisioned (Management, Audit, Log Archive, Dev, Prod)
- [ ] You are signed in via **IAM Identity Center SSO portal** — never use root for any of these steps
- [ ] SSO portal URL: `https://d-90667c19c8.awsapps.com/start/`
- [ ] You are operating in **us-east-1** throughout all steps — stay in the same region
- [ ] Audit account ID ready: `536882094857` — this becomes the delegated admin for all services

---

## Architecture Overview

```
Management Account (516027198635)
│
│  Delegates admin to Audit account for:
│  ├── GuardDuty
│  ├── Security Hub
│  ├── Amazon Inspector
│  └── IAM Access Analyzer (org zone of trust)
│
├── Security OU
│   ├── Audit Account (536882094857)        ← Central security hub
│   │   ├── GuardDuty aggregator            ← All threat findings flow here
│   │   ├── Security Hub aggregator         ← All compliance findings flow here
│   │   ├── Inspector aggregator            ← All vulnerability findings flow here
│   │   └── Access Analyzer (org-wide)      ← External access findings
│   │
│   └── Log Archive Account (251202803422)  ← CloudTrail + Config logs
│
└── Workload OU
    ├── Dev Account     ← Findings flow up to Audit account automatically
    └── Prod Account    ← Findings flow up to Audit account automatically
```
<img width="1536" height="1024" alt="ChatGPT Image Sep 11, 2026, 01_49_40 PM" src="https://github.com/user-attachments/assets/dcd4cd17-f69a-4bc7-97ed-58685dfaeb8d" />

---

## Step-by-Step Setup (AWS Console — 2025)

> All steps are performed signed in via the IAM Identity Center SSO portal. Never use root.
> Stay in **us-east-1** throughout the entire setup.

---

## Part 1 — Amazon GuardDuty

### What GuardDuty Does

Amazon GuardDuty is a continuous threat detection service. It analyzes CloudTrail logs, VPC Flow Logs, and DNS logs across all accounts to detect threats like:
- Compromised EC2 instances communicating with known malicious IPs
- Unusual API calls indicating credential theft or account compromise
- S3 bucket exfiltration attempts
- Cryptocurrency mining activity
- Privilege escalation attempts

With delegated admin setup, GuardDuty findings from every account in the organization flow automatically into the Audit account — no manual configuration needed per account.

---

### Step 1 — Enable GuardDuty in the Management Account

1. Sign in to **Management account** (`516027198635`) via SSO portal → `AWSAdministratorAccess` role
2. Make sure you are in **us-east-1**
3. In the search bar, type **GuardDuty** and click it
4. Click **"Get Started"**
5. Click **"Enable GuardDuty"**

> GuardDuty is now enabled in the Management account. This is required before you can delegate admin to the Audit account.

---

### Step 2 — Delegate GuardDuty Admin to the Audit Account

1. In the GuardDuty console, go to **Settings** in the left menu
2. Scroll down to **Delegated administrator**
3. In the account ID field, enter: `536882094857`
4. Click **"Delegate"**
5. A confirmation dialog appears — click **"Delegate"** again to confirm

> AWS automatically enables GuardDuty in the Audit account and registers it as the GuardDuty delegated administrator for the entire organization. This takes about 1–2 minutes.

---

### Step 3 — Configure GuardDuty Org-Wide from the Audit Account

1. Sign out of Management account
2. Sign in to **Audit account** (`536882094857`) via SSO portal → `AWSAdministratorAccess` role
3. Go to **GuardDuty** in the AWS Console
4. Go to **Accounts** in the left menu
5. You will see all organization accounts listed with their enrollment status
6. Click **"Enable all"** to enable GuardDuty across all accounts at once
7. Check **"Auto-enable for new accounts"** — this ensures any future account provisioned via Account Factory automatically gets GuardDuty enabled without manual steps

---

### Step 4 — Enable GuardDuty Protection Plans

Still in the Audit account GuardDuty console:

1. Go to **Protection plans** in the left menu
2. Enable the following for all accounts:

| Protection Plan | What It Detects |
|---|---|
| **S3 Protection** | Malicious API calls, unusual access patterns against S3 buckets |
| **EKS Audit Log Monitoring** | Suspicious activity in EKS clusters |
| **Malware Protection** | Malware on EC2 instances and ECS workloads |
| **RDS Protection** | Suspicious login attempts and brute force on RDS databases |
| **Lambda Network Activity** | Suspicious outbound network calls from Lambda functions |

3. For each protection plan, click **"Enable"** and select **"Enable for all accounts"**

---

### Step 5 — Verify GuardDuty is Active Across All Accounts

1. Go to **GuardDuty → Accounts** in the Audit account
2. All 5 accounts should show status **"Enabled"**
3. Go to **GuardDuty → Summary** — the findings dashboard appears
4. To test the pipeline is working:
   - Go to **Settings → Sample findings**
   - Click **"Generate sample findings"**
   - Wait 2–3 minutes
   - Go to **Findings** — sample findings from all severity levels appear
   - These are test findings only — they do not represent real threats
<img width="1883" height="810" alt="Screenshot 2026-09-11 123417" src="https://github.com/user-attachments/assets/3193c023-3ef0-486f-b56a-2dcb4a514f6a" />


---

## Part 2 — AWS Security Hub

### What Security Hub Does

AWS Security Hub is the central compliance and security findings aggregator. It:
- Runs continuous automated checks against your AWS resources using industry security standards
- Aggregates findings from GuardDuty, Inspector, Access Analyzer, and other AWS security services into one dashboard
- Scores your accounts against frameworks like CIS AWS Foundations Benchmark and NIST SP 800-53
- Shows you exactly which resources are failing which controls across every account in the organization

With delegated admin setup, Security Hub in the Audit account becomes the single dashboard showing the security posture of every account in the organization.

---

### Step 6 — Enable Security Hub in the Management Account

1. Sign in to **Management account** via SSO portal
2. In the search bar, type **Security Hub** and click it
3. Click **"Go to Security Hub"**
4. Click **"Enable Security Hub"**
5. On the enablement page, select the following security standards:
   - ✅ **AWS Foundational Security Best Practices (FSBP)** — 200+ controls across all core AWS services
   - ✅ **CIS AWS Foundations Benchmark v1.4** — industry-standard hardening benchmark
   - ✅ **NIST SP 800-53** — federal compliance framework
6. Click **"Enable Security Hub"**

---

### Step 7 — Delegate Security Hub Admin to the Audit Account

1. In Security Hub, go to **Settings** in the left menu
2. Click the **"General"** tab
3. Under **Delegated administrator**, enter: `536882094857`
4. Click **"Delegate"**
5. Confirm the delegation

> The Audit account is now the Security Hub delegated administrator. All findings from all accounts will aggregate here automatically.

---

### Step 8 — Configure Security Hub Org-Wide from the Audit Account

1. Sign in to **Audit account** via SSO portal
2. Go to **AWS Security Hub**
3. Go to **Settings → Accounts** in the left menu
4. Click **"Enable all accounts"**
5. Check **"Automatically enable new accounts"**
6. Go to **Settings → Configuration**
7. Select **"Central configuration"** — this allows you to manage Security Hub settings, standards, and controls for all member accounts directly from the Audit account without signing into each one

---

### Step 9 — Enable Security Standards Across All Accounts

Still in Audit account Security Hub:

1. Go to **Security standards** in the left menu
2. For each standard below, click **"Enable"** then select **"Apply to all accounts in organization"**:

| Standard | Purpose |
|---|---|
| **AWS Foundational Security Best Practices** | Checks 200+ controls across IAM, S3, EC2, RDS, CloudTrail, and more |
| **CIS AWS Foundations Benchmark v1.4** | Industry-standard hardening checks — most commonly required by enterprise clients |
| **NIST SP 800-53** | Required for government and regulated industry compliance |

---

### Step 10 — Verify Security Hub Findings Are Flowing

1. Go to **Security Hub → Summary** in the Audit account
2. You will see a findings count broken down by severity — CRITICAL, HIGH, MEDIUM, LOW, INFORMATIONAL
3. Go to **Security Hub → Findings**
4. Use the **Account ID** filter to confirm findings from Dev and Prod accounts are appearing in the Audit account
5. Go to **Security Hub → Security standards** — each standard shows a compliance score per account
6. Go to **Security Hub → Insights** — pre-built views showing:
   - Top failing controls across the organization
   - Accounts with the most findings
   - Most affected resource types

> First full findings scan takes approximately **30 minutes** to complete across all accounts.
<img width="1907" height="788" alt="Screenshot 2026-09-11 131013" src="https://github.com/user-attachments/assets/122d37df-24bb-48db-b34c-8e56d4d99537" />

---

## Part 3 — Amazon Inspector

### What Inspector Does

Amazon Inspector is a continuous vulnerability management service. Unlike GuardDuty which detects active threats, Inspector proactively scans your infrastructure for known vulnerabilities before they are exploited:
- **EC2 scanning** — scans operating system packages and software on EC2 instances for known CVEs using the SSM agent
- **ECR container scanning** — scans container images in ECR for OS and application layer vulnerabilities every time an image is pushed
- **Lambda scanning** — scans Lambda function code and dependencies for vulnerabilities in application packages

Inspector findings are scored using CVSS (Common Vulnerability Scoring System) and automatically flow into Security Hub, giving you a unified view of both compliance and vulnerability posture in one place.

---

### Step 11 — Enable Inspector in the Management Account

1. Sign in to **Management account** via SSO portal
2. In the search bar, type **Inspector** and click it — make sure you select **Amazon Inspector** (not Inspector Classic)
3. Click **"Get Started"**
4. Click **"Enable Inspector"**

> Inspector is now enabled in the Management account. Next we delegate admin to the Audit account.

---

### Step 12 — Delegate Inspector Admin to the Audit Account

1. In the Inspector console, go to **Settings** in the left menu
2. Click **"General settings"**
3. Under **Delegated administrator**, enter: `536882094857`
4. Click **"Delegate"**
5. Confirm the delegation

> The Audit account is now the Inspector delegated administrator for the organization.

---

### Step 13 — Configure Inspector Org-Wide from the Audit Account

1. Sign in to **Audit account** via SSO portal
2. Go to **Amazon Inspector**
3. Go to **Account management** in the left menu
4. You will see all organization accounts listed
5. Select all accounts → click **"Enable"**
6. Check **"Auto-enable for new member accounts"** — new accounts provisioned via Account Factory will automatically get Inspector enabled

---

### Step 14 — Enable Inspector Scan Types

Still in Audit account Inspector console:

1. Go to **Account management**
2. For each account, enable the following scan types:

| Scan Type | What It Scans |
|---|---|
| **EC2 scanning** | OS packages and software on EC2 instances for CVEs — requires SSM agent |
| **ECR container scanning** | Container images in ECR for OS and application layer vulnerabilities |
| **Lambda standard scanning** | Lambda function application package dependencies for known CVEs |

3. Click **"Enable all scan types"** for all accounts at once

> EC2 scanning requires the **SSM Agent** to be installed and running on EC2 instances. AWS managed AMIs (Amazon Linux 2, Windows Server) have SSM Agent pre-installed.

---

### Step 15 — Review Inspector Findings

1. Go to **Inspector → Findings** in the Audit account
2. Findings are organized by:
   - **Severity** — CRITICAL, HIGH, MEDIUM, LOW
   - **Finding type** — Package vulnerability, Network reachability
   - **Resource type** — EC2, ECR, Lambda
   - **Account** — filter by account to see Dev vs Prod findings separately
3. Go to **Inspector → Dashboard** — shows:
   - Total active findings by severity across the organization
   - Most critical vulnerabilities ranked by CVSS score
   - Accounts with the most open findings
   - Top 5 vulnerable images in ECR
4. Go to **Inspector → Coverage** — shows which resources are being scanned and which are not covered
<img width="1906" height="810" alt="Screenshot 2026-09-11 130459" src="https://github.com/user-attachments/assets/4c12134b-563d-4552-b75b-a6c5bf0169e2" />

---

### Step 16 — Verify Inspector is Sending Findings to Security Hub

1. Go to **Security Hub → Findings** in the Audit account
2. Filter by **Product name** → **Inspector**
3. Inspector findings should appear here alongside GuardDuty findings
4. This confirms the full findings pipeline is working:
   - Inspector scans resources in Dev and Prod accounts
   - Findings flow to Inspector aggregator in Audit account
   - Inspector automatically sends findings to Security Hub
   - Security Hub shows unified view of all findings

---

## Part 4 — IAM Access Analyzer

### What Access Analyzer Does

IAM Access Analyzer continuously monitors resource policies across your organization and flags any resource that is accessible from **outside your AWS organization**. This catches unintended public or cross-account access on:
- S3 buckets
- IAM roles with external trust relationships
- KMS keys shared outside the org
- Lambda functions with resource-based policies allowing external invocation
- SQS queues with external access
- Secrets Manager secrets shared externally

With organization-level zone of trust, Access Analyzer treats your entire AWS organization as trusted — anything accessible from outside the org boundary is flagged as a finding.

---

### Step 17 — Enable Access Analyzer in the Management Account

1. Sign in to **Management account** via SSO portal
2. Go to **IAM** in the AWS Console
3. In the left menu, click **"Access Analyzer"**
4. Click **"Create analyzer"**
5. Fill in:
   - **Analyzer name** → `org-access-analyzer`
   - **Zone of trust** → select **Organization**
   - This means the entire AWS organization is treated as trusted — only access from outside the org boundary is flagged
6. Click **"Create analyzer"**

> The analyzer is created and immediately begins scanning all resource policies across all accounts in the organization. Initial scan takes a few minutes.

---

### Step 18 — Review Access Analyzer Findings

1. Go to **IAM → Access Analyzer → Findings** in the Management account
2. Each finding shows:
   - **Resource** — the specific S3 bucket, IAM role, KMS key, etc.
   - **External principal** — who has access from outside the org
   - **Access level** — read, write, list, etc.
   - **Account** — which account the resource lives in
3. For each finding, decide:
   - If the external access is **intentional** → click **"Archive"** to acknowledge it
   - If the external access is **unintended** → fix the resource policy to remove the external access, then the finding automatically resolves

---

### Step 19 — Enable Access Analyzer in the Audit Account

For account-level analysis in addition to org-level:

1. Sign in to **Audit account** via SSO portal
2. Go to **IAM → Access Analyzer**
3. Click **"Create analyzer"**
4. Fill in:
   - **Analyzer name** → `audit-account-analyzer`
   - **Zone of trust** → **Current account**
5. Click **"Create analyzer"**

> This analyzer flags any resource in the Audit account accessible from outside the Audit account itself — useful for catching overly permissive security tool configurations.

---

## Part 5 — Automated Remediation with Security Hub + Lambda

### What Automated Remediation Does

Instead of manually reviewing Security Hub findings and fixing them one by one, automated remediation uses EventBridge to detect CRITICAL findings the moment they appear and triggers a Lambda function to fix the issue automatically — no human intervention required. Examples:
- A public S3 bucket is detected → Lambda immediately blocks public access
- An overly permissive security group (0.0.0.0/0 on port 22 or 3389) is detected → Lambda revokes the rule
- An IAM access key is exposed → Lambda deactivates the key

The EventBridge rule runs in the **Audit account** and listens for Security Hub findings across the entire organization. Lambda executes the remediation in the affected member account using a cross-account IAM role.

---

### Step 20 — Create the Cross-Account Remediation IAM Role

This role must exist in every member account (Dev, Prod) so the Lambda function in the Audit account can assume it and perform remediation.

1. Sign in to **Dev account** via SSO portal → `AWSAdministratorAccess`
2. Go to **IAM → Roles → Create role**
3. Fill in:
   - **Trusted entity type** → **AWS account**
   - **Account ID** → `536882094857` (Audit account — the Lambda will assume this role from here)
4. Click **Next**
5. Attach the following permissions policies:
   - `AmazonS3FullAccess` — to block public access on S3 buckets
   - `AmazonEC2FullAccess` — to revoke security group rules
   - `IAMFullAccess` — to deactivate IAM access keys

> For production use, replace these broad policies with a custom least-privilege policy scoped to only the remediation actions needed.

6. Click **Next**
7. **Role name** → `SecurityHubRemediationRole`
8. Click **Create role**
9. Copy the **Role ARN** — you will need it in the Lambda function

**Repeat this step in the Prod account** — create the same `SecurityHubRemediationRole` with the same trust policy pointing to `536882094857`.

---

### Step 21 — Create the Remediation Lambda Function

1. Sign in to **Audit account** (`536882094857`) via SSO portal
2. Go to **AWS Lambda → Functions → Create function**
3. Fill in:
   - **Function name** → `SecurityHubAutoRemediation`
   - **Runtime** → **Python 3.12**
   - **Architecture** → x86_64
4. Click **Create function**
5. In the **Code** tab, replace the default code with the following:

```python
import boto3

def lambda_handler(event, context):
    finding = event["detail"]["findings"][0]
    account_id = finding["AwsAccountId"]
    resources = finding["Resources"]

    role_arn = f"arn:aws:iam::{account_id}:role/SecurityHubRemediationRole"
    sts = boto3.client("sts")
    creds = sts.assume_role(RoleArn=role_arn, RoleSessionName="RemediationSession")["Credentials"]

    session = boto3.Session(
        aws_access_key_id=creds["AccessKeyId"],
        aws_secret_access_key=creds["SecretAccessKey"],
        aws_session_token=creds["SessionToken"],
    )

    for resource in resources:
        resource_type = resource["Type"]
        resource_id = resource["Id"]

        # Remediate public S3 bucket
        if resource_type == "AwsS3Bucket":
            bucket_name = resource_id.split(":::")[-1]
            s3 = session.client("s3")
            s3.put_public_access_block(
                Bucket=bucket_name,
                PublicAccessBlockConfiguration={
                    "BlockPublicAcls": True,
                    "IgnorePublicAcls": True,
                    "BlockPublicPolicy": True,
                    "RestrictPublicBuckets": True,
                },
            )
            print(f"Blocked public access on S3 bucket: {bucket_name}")

        # Remediate overly permissive security group (SSH/RDP open to 0.0.0.0/0)
        elif resource_type == "AwsEc2SecurityGroup":
            sg_id = resource_id.split("/")[-1]
            ec2 = session.client("ec2", region_name="us-east-1")
            sg = ec2.describe_security_groups(GroupIds=[sg_id])["SecurityGroups"][0]
            for rule in sg["IpPermissions"]:
                for ip_range in rule.get("IpRanges", []):
                    if ip_range["CidrIp"] == "0.0.0.0/0" and rule.get("FromPort") in [22, 3389]:
                        ec2.revoke_security_group_ingress(GroupId=sg_id, IpPermissions=[rule])
                        print(f"Revoked open ingress rule on security group: {sg_id}")

    return {"statusCode": 200, "body": "Remediation complete"}
```

6. Click **Deploy**

---

### Step 22 — Attach IAM Permissions to the Lambda Execution Role

The Lambda function needs permission to call STS AssumeRole to switch into member accounts.

1. Go to **Lambda → SecurityHubAutoRemediation → Configuration → Permissions**
2. Click the **execution role name** — this opens IAM in a new tab
3. Click **Add permissions → Attach policies**
4. Attach **`AWSSecurityHubReadOnlyAccess`**
5. Click **Add permissions → Create inline policy**
6. Switch to **JSON** and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::*:role/SecurityHubRemediationRole"
    }
  ]
}
```

7. **Policy name** → `AllowAssumeRemediationRole`
8. Click **Create policy**

---

### Step 23 — Create the EventBridge Rule to Trigger Lambda

1. Go to **Amazon EventBridge → Rules → Create rule**
2. Fill in:
   - **Name** → `SecurityHubCriticalFindingRemediation`
   - **Event bus** → **default**
   - **Rule type** → **Rule with an event pattern**
3. Click **Next**
4. Under **Event pattern**, select:
   - **Event source** → **AWS services**
   - **AWS service** → **Security Hub**
   - **Event type** → **Security Hub Findings - Imported**
5. Click **Edit pattern** and replace with the following to filter only CRITICAL findings:

```json
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "Severity": {
        "Label": ["CRITICAL"]
      },
      "Workflow": {
        "Status": ["NEW"]
      },
      "RecordState": ["ACTIVE"]
    }
  }
}
```

6. Click **Next**
7. Under **Target**:
   - **Target type** → **AWS service**
   - **Select a target** → **Lambda function**
   - **Function** → `SecurityHubAutoRemediation`
8. Click **Next → Next → Create rule**

> The EventBridge rule is now live. Every time a CRITICAL Security Hub finding is imported from any account in the organization, EventBridge triggers the Lambda function which assumes the `SecurityHubRemediationRole` in the affected account and performs the remediation automatically.

---

### Step 24 — Test the Automated Remediation Pipeline

1. Sign in to **Dev account** via SSO portal
2. Go to **Amazon S3 → Create bucket**
3. Create a test bucket with **Block all public access turned OFF** — this will trigger a Security Hub finding
4. Wait approximately **5–10 minutes** for Security Hub to detect the misconfiguration
5. Go to **Audit account → Security Hub → Findings** — a CRITICAL finding for the public S3 bucket appears
6. Within 1–2 minutes of the finding appearing, go back to **Dev account → S3 bucket → Permissions**
7. Confirm that **Block all public access** is now **ON** — Lambda remediated it automatically
8. Go to **Audit account → Lambda → SecurityHubAutoRemediation → Monitor → Logs** — CloudWatch logs show the remediation execution

---

## Part 6 — AWS Config Conformance Packs

### What Conformance Packs Do

AWS Config Conformance Packs are pre-built collections of AWS Config rules that map to specific compliance frameworks. Instead of manually creating individual Config rules for each compliance requirement, you deploy a single conformance pack and AWS creates all the rules automatically across every account in the organization.

This is particularly valuable for regulated industry clients:
- **PCI-DSS** — required for any workload handling payment card data
- **HIPAA** — required for healthcare workloads handling protected health information
- **SOC 2** — required for SaaS companies undergoing SOC 2 Type II audits

Conformance packs are deployed from the **Management account** using AWS Config's organization-level deployment, which pushes the rules to every account in the organization simultaneously.

---

### Step 25 — Enable AWS Config in the Management Account

AWS Config must be active in the Management account before deploying organization conformance packs.

1. Sign in to **Management account** via SSO portal
2. Go to **AWS Config** in the AWS Console
3. If Config is not yet enabled, click **"Get started"**
4. Fill in:
   - **Recording strategy** → **Record all current and future resource types**
   - **AWS Config role** → **Create AWS Config service-linked role** (default)
   - **S3 bucket** → **Create a new bucket** — AWS names it automatically
5. Click **Confirm**

> AWS Config is now recording resource configuration changes in the Management account.

---

### Step 26 — Deploy the PCI-DSS Conformance Pack Org-Wide

1. Go to **AWS Config → Conformance packs** in the Management account
2. Click **"Deploy conformance pack"**
3. Fill in:
   - **Conformance pack name** → `PCI-DSS-Compliance`
   - **Template** → select **"Use sample template"**
   - **Sample template** → select **`Operational Best Practices for PCI DSS 3.2.1`** from the dropdown
4. Under **Delivery location**:
   - **S3 bucket** → select the Config S3 bucket created in Step 25 or use the Control Tower managed bucket
5. Click **Next → Deploy conformance pack**

> AWS Config deploys the PCI-DSS conformance pack to every account in the organization. This creates approximately 140 Config rules covering encryption, access control, logging, and network security requirements from PCI-DSS 3.2.1.

---

### Step 27 — Deploy the HIPAA Conformance Pack Org-Wide

1. Go to **AWS Config → Conformance packs**
2. Click **"Deploy conformance pack"**
3. Fill in:
   - **Conformance pack name** → `HIPAA-Compliance`
   - **Template** → **"Use sample template"**
   - **Sample template** → **`Operational Best Practices for HIPAA Security`**
4. Click **Next → Deploy conformance pack**

> The HIPAA conformance pack deploys rules covering encryption at rest, encryption in transit, access logging, MFA enforcement, and backup requirements — all mapped to HIPAA Security Rule safeguards.

---

### Step 28 — Deploy the SOC 2 Conformance Pack Org-Wide

1. Go to **AWS Config → Conformance packs**
2. Click **"Deploy conformance pack"**
3. Fill in:
   - **Conformance pack name** → `SOC2-Compliance`
   - **Template** → **"Use sample template"**
   - **Sample template** → **`Operational Best Practices for SOC 2`**
4. Click **Next → Deploy conformance pack**

> The SOC 2 conformance pack deploys rules covering availability, confidentiality, processing integrity, and security trust service criteria — the four pillars auditors check during a SOC 2 Type II audit.

---

### Step 29 — Review Conformance Pack Compliance Results

1. Go to **AWS Config → Conformance packs** in the Management account
2. Click on **PCI-DSS-Compliance**
3. You will see:
   - **Compliance score** — percentage of rules currently passing across all accounts
   - **Rules** tab — each individual Config rule with COMPLIANT / NON_COMPLIANT status
   - **Resources** tab — specific resources that are failing each rule
4. Click on any NON_COMPLIANT rule to see:
   - Which accounts have failing resources
   - Which specific resources are non-compliant
   - The remediation action needed
5. Repeat for **HIPAA-Compliance** and **SOC2-Compliance** packs

> Conformance pack results also flow into the **Audit account AWS Config aggregator** — you can view compliance scores for all three frameworks across all accounts from the Audit account in one place.

---

### Conformance Pack Summary

| Pack | Template Name | Rules Deployed | Target Clients |
|---|---|---|---|
| PCI-DSS | Operational Best Practices for PCI DSS 3.2.1 | ~140 rules | Fintech, e-commerce, payment processors |
| HIPAA | Operational Best Practices for HIPAA Security | ~90 rules | Healthcare, health-tech, insurance |
| SOC 2 | Operational Best Practices for SOC 2 | ~100 rules | SaaS companies, B2B platforms |

---

## Part 7 — Verification Checklist

### Step 30 — Full End-to-End Verification

Sign in to the **Audit account** via SSO portal and verify each service:

#### GuardDuty
1. Go to **GuardDuty → Accounts** — all 5 accounts show **Enabled**
2. Go to **GuardDuty → Summary** — findings dashboard is populated
3. Go to **GuardDuty → Protection plans** — all protection plans show **Enabled**
4. Confirm **Auto-enable for new accounts** is checked

#### Security Hub
1. Go to **Security Hub → Summary** — findings from all accounts are visible
2. Go to **Security Hub → Security standards** — all 3 standards show compliance scores
3. Go to **Security Hub → Findings** — filter by account ID to confirm Dev and Prod findings are flowing in
4. Go to **Security Hub → Settings → Accounts** — all accounts show **Enabled**

#### Amazon Inspector
1. Go to **Inspector → Dashboard** — findings from all accounts are visible
2. Go to **Inspector → Coverage** — EC2, ECR, and Lambda resources show as covered
3. Go to **Inspector → Account management** — all accounts show **Enabled**
4. Go to **Security Hub → Findings** → filter by **Inspector** — Inspector findings appear in Security Hub

#### IAM Access Analyzer
1. Go to **Management account → IAM → Access Analyzer**
2. Analyzer `org-access-analyzer` shows status **Active**
3. Findings tab shows any externally accessible resources
4. All findings reviewed and either archived or remediated

#### Automated Remediation
1. Go to **Audit account → EventBridge → Rules** — rule `SecurityHubCriticalFindingRemediation` shows **Enabled**
2. Go to **Lambda → SecurityHubAutoRemediation → Monitor** — invocation count shows activity after any CRITICAL finding
3. Go to **CloudWatch → Log groups → /aws/lambda/SecurityHubAutoRemediation** — logs show successful remediation executions
4. Confirm `SecurityHubRemediationRole` exists in both Dev and Prod accounts

#### Config Conformance Packs
1. Go to **Management account → AWS Config → Conformance packs**
2. All 3 packs show status **COMPLIANT** or **NON_COMPLIANT** (not Pending) — meaning rules have been deployed and evaluated
3. Click each pack — compliance score is visible with per-rule and per-resource breakdown
4. Go to **Audit account → AWS Config → Aggregator** — conformance pack results from all accounts are visible here

---

## Account Summary

| Account | Role in This Project |
|---|---|
| Management (`516027198635`) | Delegates GuardDuty, Security Hub, Inspector admin to Audit account. Hosts org-level Access Analyzer |
| Audit (`536882094857`) | Central aggregator for all findings — GuardDuty, Security Hub, Inspector all report here |
| Log Archive (`251202803422`) | Unchanged — continues storing CloudTrail and Config logs |
| Development | Member account — findings automatically flow to Audit account |
| Production | Member account — findings automatically flow to Audit account |

---

## Services Summary

| Service | What It Does | Where Findings Go |
|---|---|---|
| Amazon GuardDuty | Threat detection — active attacks, compromised credentials, malicious activity | GuardDuty aggregator in Audit account |
| AWS Security Hub | Compliance checks — resource misconfigurations against FSBP, CIS, NIST standards | Security Hub aggregator in Audit account |
| Amazon Inspector | Vulnerability management — CVEs in EC2, ECR images, Lambda packages | Inspector aggregator in Audit account + Security Hub |
| IAM Access Analyzer | Access analysis — unintended external access to resources across the org | Management account + IAM in each account |
| EventBridge + Lambda | Automated remediation — CRITICAL Security Hub findings trigger Lambda to auto-fix misconfigurations | CloudWatch Logs in Audit account |
| AWS Config Conformance Packs | Compliance framework enforcement — PCI-DSS, HIPAA, SOC 2 rules deployed org-wide | Config aggregator in Audit account |

---

## Security Posture Achieved

- **Threat detection** — GuardDuty monitors all accounts 24/7 for active threats and anomalous behavior
- **Compliance visibility** — Security Hub scores every account against CIS, FSBP, and NIST from a single dashboard
- **Vulnerability management** — Inspector continuously scans EC2, containers, and Lambda for known CVEs before they are exploited
- **Access governance** — Access Analyzer flags any resource accidentally exposed outside the organization boundary
- **Automated remediation** — CRITICAL Security Hub findings are auto-remediated by Lambda within minutes — no manual intervention required
- **Regulatory compliance** — PCI-DSS, HIPAA, and SOC 2 conformance packs enforce framework-specific Config rules across every account simultaneously
- **Zero manual per-account setup** — delegated admin pattern means new accounts provisioned via Account Factory automatically inherit all services
- **Single pane of glass** — all findings from all accounts across all services are visible in the Audit account

---

## Notes

- GuardDuty, Security Hub, and Inspector all have per-account costs — review AWS pricing before enabling across many accounts
- Inspector EC2 scanning requires SSM Agent on EC2 instances — AWS managed AMIs have it pre-installed
- Security Hub findings take up to 30 minutes to populate after initial enablement
- Access Analyzer findings for archived resources do not reappear unless the resource policy changes again
- All services support **us-east-1** and all other AWS regions — if you operate in multiple regions, repeat the delegated admin setup per region
- The Lambda remediation function handles S3 and security group findings by default — extend the handler with additional `elif` blocks for other resource types as needed
- Conformance pack deployment to all organization accounts can take **15–30 minutes** — Config rules are evaluated on a schedule, not instantly
- For regulated clients, export conformance pack compliance reports from AWS Config as evidence for PCI-DSS, HIPAA, or SOC 2 auditors

---

