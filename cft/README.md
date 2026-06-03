# Lab 0: AWS Organization & Security Services Setup

This lab deploys a complete AWS Organization with security services (Security Hub, GuardDuty, CloudTrail, StackSets, SCPs) and IAM Identity Center for user access management.

## Prerequisites

- AWS Account with no existing Organization (fresh Workshop Studio account)
- AWS CLI configured with Administrator access
- Unique email address for the Audit Account
- Unique email address for the Workload Account

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     AWS Organization                             │
├─────────────────────────────────────────────────────────────────┤
│  Management Account (Root)                                       │
│  ├── Security Hub (enabled)                                      │
│  ├── GuardDuty (enabled)                                         │
│  ├── CloudTrail (Organization Trail)                             │
│  ├── StackSets (trusted access enabled)                          │
│  ├── SSM (trusted access enabled for Fleet Manager)              │
│  ├── AWS Config (trusted access enabled)                         │
│  ├── IAM Root Access Consolidation (trusted access enabled)      │
│  ├── License Manager (trusted access enabled)                    │
│  └── SCPs:                                                       │
│       ├── DenyDisableSecurityServices                            │
│       └── AllowOnlyUSRegions (us-east-1/2, us-west-1/2)         │
├─────────────────────────────────────────────────────────────────┤
│  Organizational Units                                            │
│  ├── Security OU                                                 │
│  │   └── Audit Account (Delegated Admin for SecurityHub/GuardDuty)│
│  └── Workloads OU                                                │
│       └── Workload Account (workshop workloads)                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Deployment Steps

### Step 1: Deploy Organization & Security Services

This creates the AWS Organization, OUs, Audit Account, Workload Account, and enables all security services.

```bash
aws cloudformation deploy \
  --template-file static/labs/lab0/org-security-services.yaml \
  --stack-name org-security-services \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    AuditAccountEmail=audit@your-domain.com \
    WorkloadAccountEmail=workload@your-domain.com
```

**Parameters:**
| Parameter | Description | Example |
|-----------|-------------|---------|
| `AuditAccountEmail` | Unique email for Audit Account | `workshop-audit@example.com` |
| `WorkloadAccountEmail` | Unique email for Workload Account | `workshop-workload@example.com` |

**Deployment Time:** ~15-20 minutes

**What it creates:**
- AWS Organization with ALL features enabled
- Security OU and Workloads OU
- Audit Account (delegated administrator) in Security OU
- Workload Account in Workloads OU
- Security Hub with Audit Account as delegated admin (organization-wide, all 4 US regions)
- GuardDuty with Audit Account as delegated admin (organization-wide, all 4 US regions, all protection plans enabled)
- Organization CloudTrail with S3 bucket and SNS topic
- AWS Service Access enabled for:
  - **StackSets** - Organization-wide CloudFormation deployments
  - **Systems Manager (SSM)** - Fleet Manager and instance management
  - **AWS Config** - Centralized compliance monitoring
  - **IAM Root Access Consolidation** - Centralized root credential management
  - **License Manager** - Organization-wide license sharing
- Service Control Policies (SCPs):
  - **DenyDisableSecurityServices**: Prevents disabling GuardDuty, Security Hub, and CloudTrail
  - **AllowOnlyUSRegions**: Restricts resource creation to us-east-1, us-east-2, us-west-1, us-west-2 only

### Step 2: Enable IAM Identity Center (Manual)

> ⚠️ **IMPORTANT:** IAM Identity Center cannot be enabled via CloudFormation API. This step MUST be done manually.

1. Go to **AWS Console** → Search for **"IAM Identity Center"**
2. Click **"Enable"** (if not already enabled)
3. Wait for the status to show **"Enabled"** (~2-3 minutes)
4. Note down:
   - **Instance ARN** (e.g., `arn:aws:sso:::instance/ssoins-xxxxxxxxx`)
   - **Identity Store ID** (e.g., `d-xxxxxxxxxx`)

### Step 3: Deploy Identity Center Configuration

After Identity Center is enabled, deploy the user/group configuration:

```bash
aws cloudformation deploy \
  --template-file static/labs/lab0/identity-center.yaml \
  --stack-name identity-center \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    AdminUserEmail=your-email@example.com \
    AdminUserFirstName=Workshop \
    AdminUserLastName=Admin \
    AdminUserDisplayName="Workshop Admin"
```

**Parameters:**
| Parameter | Description | Default |
|-----------|-------------|---------|
| `AdminUserEmail` | Email for the admin user | (required) |
| `AdminUserFirstName` | First name | `Workshop` |
| `AdminUserLastName` | Last name | `Admin` |
| `AdminUserDisplayName` | Display name | `Workshop Admin` |

**What it creates:**
- Admin user in Identity Store
- `WorkshopAdmins` group
- `AdministratorAccess` permission set (8-hour sessions)
- Group assigned to all member accounts (Audit Account, Workload Account)

### Step 4: Send Password Email to User (Manual)

> ⚠️ **IMPORTANT:** The Identity Store API creates users but does NOT automatically send password setup emails.

1. Go to **IAM Identity Center** → **Users**
2. Select the created user (e.g., "Workshop Admin")
3. Click **"Reset password"** → **"Generate one-time password"**
4. **Copy the one-time password** and share it with the user
   - Or select **"Send email"** to email the password setup link

### Step 5: Add Management Account Access (Manual)

> ⚠️ **IMPORTANT:** AWS blocks SSO assignments to the management account via API.

1. Go to **IAM Identity Center** → **AWS Accounts**
2. Check the box next to **Management Account** (the account ID where you deployed the stack)
3. Click **"Assign users or groups"**
4. Select the **"Groups"** tab
5. Check **"WorkshopAdmins"**
6. Click **"Next"**
7. Select **"AdministratorAccess"** permission set
8. Click **"Submit"**

---

## Verification

After completing all steps, verify the deployment:

### Check Organization Structure
```bash
aws organizations describe-organization
aws organizations list-accounts
aws organizations list-organizational-units-for-parent \
  --parent-id $(aws organizations list-roots --query 'Roots[0].Id' --output text)
```

### Check Security Services
```bash
# Security Hub
aws securityhub describe-hub

# GuardDuty
aws guardduty list-detectors

# CloudTrail
aws cloudtrail describe-trails --trail-name-list OrganizationTrail

# StackSets
aws cloudformation describe-organizations-access
```

### Test SSO Access
1. Go to the SSO Portal URL: `https://<account-id>.awsapps.com/start`
2. Sign in with the admin user credentials
3. Verify you see all accounts:
   - Management Account
   - Audit Account
   - Workload Account

---

## Outputs

### org-security-services Stack Outputs
| Output | Description |
|--------|-------------|
| `OrganizationId` | AWS Organization ID |
| `OrganizationRootId` | Organization Root ID |
| `SecurityOUId` | Security OU ID |
| `WorkloadsOUId` | Workloads OU ID |
| `AuditAccountId` | Audit Account ID |
| `WorkloadAccountId` | Workload Account ID |
| `CloudTrailBucketName` | CloudTrail S3 bucket name |
| `CloudTrailArn` | Organization CloudTrail ARN |

### identity-center Stack Outputs
| Output | Description |
|--------|-------------|
| `IdentityCenterInstanceArn` | IAM Identity Center Instance ARN |
| `IdentityStoreId` | Identity Store ID |
| `AdminUserId` | Admin User ID |
| `AdminGroupId` | WorkshopAdmins Group ID |
| `SSOPortalURL` | SSO Portal URL |
| `AssignmentSummary` | Summary of account assignments |

---

## Troubleshooting

### "Organization management account is not allowed to perform the operation"
- **Cause:** AWS restricts certain SSO operations via API on the management account
- **Solution:** Perform the operation manually via AWS Console (Steps 2, 4, 5)

### Stack fails on CreateGroup or CreateUser
- **Cause:** Identity Center not fully enabled yet
- **Solution:** Wait 2-3 minutes after enabling Identity Center, then retry

### User doesn't receive password email
- **Cause:** The Identity Store API doesn't automatically send emails
- **Solution:** Manually send password via IAM Identity Center console (Step 4)

### "ConcurrentModificationException"
- **Cause:** Multiple operations on Organization happening simultaneously
- **Solution:** Wait and retry - the templates have built-in retry logic

---

## Cleanup

To delete all resources:

```bash
# Delete Identity Center stack first
aws cloudformation delete-stack --stack-name identity-center
aws cloudformation wait stack-delete-complete --stack-name identity-center

# Delete Org Security stack (note: Organization cannot be deleted via CloudFormation)
aws cloudformation delete-stack --stack-name org-security-services
aws cloudformation wait stack-delete-complete --stack-name org-security-services
```

> ⚠️ **Note:** The AWS Organization and member accounts will remain after stack deletion. To fully clean up, you must manually:
> 1. Remove member accounts from the Organization
> 2. Delete the Organization
> 3. Close member accounts (if desired)

---

## Alternative Templates

| Template | Use Case |
|----------|----------|
| `org-security-services.yaml` | Fresh deployment - creates new Organization |
| `org-security-services-existing-org.yaml` | Existing Organization - adds OUs and Audit Account |
| `org-security-services-continue.yaml` | Resume after partial deployment failure |

---

## Support

For issues with this workshop:
- Check CloudFormation stack events for error details
- Check Lambda CloudWatch logs: `/aws/lambda/EnableOrgServiceAccess-*`
- Check Lambda CloudWatch logs: `/aws/lambda/IdentityCenterSetup-*`
