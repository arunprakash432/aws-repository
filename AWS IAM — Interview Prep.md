# AWS IAM — Interview Prep

Oct 13, 2026 · @varun

## Core notes

IAM (Identity and Access Management) controls **who** can do **what** on **which** AWS resources. It is global (not per region), free, and every API call to AWS is checked against it.

**Authentication vs authorization**

- Authentication = who are you? (password + MFA, access keys, SSO login, role session)
- Authorization = what are you allowed to do? (decided by policies)

### Identities

| Identity | What it is | Credentials | Use it for |
| --- | --- | --- | --- |
| Root user | Email used to create the account; full, unrestricted access | Password + MFA | Only a few root-only tasks (close account, change support plan, restore locked policies). Never daily use. |
| IAM user | A long-term identity inside one account | Console password and/or access keys (long-term) | Legacy cases, break-glass, tools that can't assume roles |
| IAM group | A collection of users; attach policies once | None (groups can't log in) | Giving the same permissions to many users |
| IAM role | An identity with permissions but no long-term credentials; assumed by someone/something trusted | Temporary credentials from STS | EC2/Lambda/EKS access to AWS, cross-account access, SSO, federation |

Key points:

- Groups cannot contain other groups. A user can be in up to 10 groups.
- A role has two policies: a **trust policy** (who can assume it) and a **permissions policy** (what it can do).
- Best practice today: humans log in through **IAM Identity Center** (SSO) or federation; workloads use **roles**. Avoid long-term access keys.

### Policies

A policy is a JSON document. Each statement has:

- **Effect** — Allow or Deny
- **Action** — API actions, e.g. `s3:GetObject`
- **Resource** — ARNs the actions apply to
- **Condition** (optional) — when it applies, e.g. source IP, MFA present, tags
- **Principal** — only in resource-based and trust policies (who the policy is about)

| Policy type | Attached to | Purpose |
| --- | --- | --- |
| AWS managed | Users, groups, roles | Pre-built by AWS (e.g. ReadOnlyAccess). Easy, but often too broad. |
| Customer managed | Users, groups, roles | Written by you, reusable, versioned (up to 5 versions). Preferred. |
| Inline | One user, group or role | Embedded 1:1; deleted with the identity. Use only for strict 1:1 cases. |
| Resource-based | A resource (S3 bucket, KMS key, SQS queue, Lambda, role trust policy) | Says who can access this resource; enables cross-account access. |
| Permissions boundary | User or role | Sets the **maximum** permissions an identity can get. Does not grant anything itself. |
| SCP / RCP (AWS Organizations) | OU or account | Guardrails: maximum permissions for every identity in member accounts (SCP) or for resources (RCP). Do not grant. |
| Session policy | Passed when assuming a role | Further limits that one session. |

### Policy evaluation logic (must know)

1. By default everything is **implicitly denied**.
2. An **explicit Deny** anywhere wins — always.
3. An **Allow** is needed from an identity-based or resource-based policy.
4. SCPs, permissions boundaries and session policies must **also** allow it (they act as filters).

Short form: *Explicit Deny > Explicit Allow > Implicit Deny.* Effective permissions = the intersection of all filters, minus any explicit deny.

Cross-account: the caller's identity policy must allow it **and** the resource policy (or role trust) in the other account must allow it.

### STS and temporary credentials

- AWS STS (Security Token Service) issues temporary credentials: access key, secret key, session token, with an expiry.
- `AssumeRole` — assume a role (same or cross-account). Default session 1 hour, max up to 12 hours (set on the role).
- `AssumeRoleWithWebIdentity` — for OIDC identities (GitHub Actions, EKS IRSA, Cognito).
- `AssumeRoleWithSAML` — for SAML IdPs (Azure AD/Entra ID, Okta).
- `GetSessionToken` — temporary creds for an IAM user, often to enforce MFA for CLI.
- **Instance profile** = the container that delivers a role to an EC2 instance; the app gets creds from the instance metadata service (use IMDSv2).

### Identity Center, federation, Organizations

- **IAM Identity Center** (formerly AWS SSO): one login for many accounts; users get **permission sets**, which become roles in each account.
- **Federation**: an external IdP (Entra ID, Okta, Google) authenticates users; AWS trusts it via SAML 2.0 or OIDC and hands out role credentials.
- **AWS Organizations**: groups accounts into OUs; SCPs set guardrails (e.g. deny leaving the org, deny regions you don't use).

### Security tools around IAM

| Tool | What it does |
| --- | --- |
| IAM Access Analyzer | Finds resources shared outside your account/org; finds unused access; validates policies; can generate a policy from CloudTrail activity |
| Credential report | Account-wide CSV: users, password/key age, MFA status, last used |
| Access Advisor (last accessed) | Shows which services an identity actually used → remove the rest |
| IAM Policy Simulator | Tests whether a policy allows an action without running it |
| CloudTrail | Logs every API call: who did what, when, from where |
| AWS Config | Rules to flag non-compliant IAM setup (e.g. root without MFA) |

### Best practices (say these in interviews)

1. Lock away the root user: MFA on, no access keys, use only for root-only tasks.
2. Humans → Identity Center/federation with temporary creds; workloads → roles.
3. Least privilege: start small, use Access Advisor/Access Analyzer to trim.
4. MFA for every human user; enforce with a condition (`aws:MultiFactorAuthPresent`).
5. No hard-coded keys in code or Git; if keys are unavoidable, rotate them and alert on age.
6. Use groups/permission sets, not per-user policies.
7. Use conditions (IP, tags, MFA, region, `aws:SecureTransport`) to tighten access.
8. Guardrails with SCPs and permissions boundaries in multi-account setups.
9. Monitor with CloudTrail, GuardDuty and Access Analyzer.

## Interview Q&A

### Basic

**1. What is IAM?** AWS's service to manage identities and permissions. It authenticates who is calling and authorizes what they can do. It's global and free.

**2. Is IAM regional or global?** Global. Users, groups, roles and policies apply across all regions.

**3. Difference between IAM user, group and role?** User = long-term identity with its own credentials. Group = a set of users sharing policies; can't log in. Role = no long-term credentials; assumed by a trusted entity to get temporary credentials.

**4. What is the root user and how do you protect it?** The account owner with full access. Enable MFA, delete its access keys, don't use it daily, and keep its email and recovery details secure. In AWS Organizations you can also centrally manage and remove root credentials of member accounts.

**5. What is an IAM policy? Explain its parts.** A JSON document with statements. Each has Effect (Allow/Deny), Action, Resource, optional Condition, and Principal in resource-based policies.

**6. Managed vs inline policy?** Managed is a standalone, reusable policy (AWS- or customer-managed). Inline is embedded in one identity and deleted with it. Prefer customer-managed.

**7. What is MFA and why use it?** A second factor (authenticator app, passkey/security key, hardware token) on top of a password. It stops account takeover from a leaked password.

**8. What is the principle of least privilege?** Give only the permissions needed for the task, nothing more. Start minimal and add as needed.

**9. What is an ARN?** Amazon Resource Name, the unique ID of a resource: `arn:aws:s3:::my-bucket/*`, `arn:aws:iam::123456789012:role/AppRole`.

**10. What are access keys?** Access key ID + secret access key for programmatic access (CLI/SDK). Long-term for IAM users, so avoid them; use roles or SSO instead.

### Intermediate

**11. Explain IAM policy evaluation logic.** Default is implicit deny. An explicit Deny anywhere overrides everything. Otherwise an Allow from an identity or resource policy is needed, and SCPs, boundaries and session policies must also allow it.

**12. Identity-based vs resource-based policy?** Identity-based is attached to a user/group/role ("what can I do"). Resource-based is attached to a resource like an S3 bucket and has a Principal ("who can access me"). Resource-based policies allow cross-account access without assuming a role.

**13. What is a trust policy?** The resource-based policy on a role saying who may assume it, e.g. `ec2.amazonaws.com` or another account ID.

**14. How does an EC2 instance get access to S3 without keys?** Attach an IAM role through an instance profile. The SDK fetches temporary creds from the instance metadata service (IMDSv2) and rotates them automatically.

**15. What is STS? Name its main APIs.** Security Token Service; issues temporary credentials. AssumeRole, AssumeRoleWithSAML, AssumeRoleWithWebIdentity, GetSessionToken, GetCallerIdentity.

**16. What does `aws sts get-caller-identity` do?** Shows the account, user/role ARN your current credentials belong to. First command to run when debugging access errors.

**17. What is a permissions boundary?** A managed policy set on a user/role that caps the maximum permissions. Effective = identity policy ∩ boundary. Used to let developers create roles safely without escalating privileges.

**18. What is an SCP?** Service Control Policy in AWS Organizations. Sets the maximum permissions for all identities in member accounts, including their root user. It grants nothing and does not apply to the management account.

**19. What is IAM Identity Center?** Central SSO for many accounts. Users from its own directory or an external IdP get permission sets, which become roles in target accounts. Gives short-lived credentials for console and CLI.

**20. What is IAM Access Analyzer?** It flags resources shared externally, unused roles/keys/permissions, checks policies for errors, and can generate least-privilege policies from CloudTrail logs.

**21. How do you rotate access keys without downtime?** Create a second key (max 2 per user), update the app, confirm the old key's last-used date stops moving, deactivate it, then delete it.

**22. What is a service-linked role?** A role predefined by an AWS service (e.g. for Auto Scaling, ELB) that the service creates and manages. You can view but not freely edit its permissions.

### Advanced

**23. How does cross-account access work?** Account B creates a role whose trust policy trusts Account A. Account A gives its user/role `sts:AssumeRole` on that role ARN. The caller assumes it and gets temporary creds in Account B. Add an `ExternalId` condition for third parties.

**24. What is the confused deputy problem?** A third party with access to many customers is tricked into acting on your account. Fix: `sts:ExternalId` in the trust policy for vendors, and `aws:SourceArn`/`aws:SourceAccount` conditions for AWS services.

**25. What is `iam:PassRole` and why is it dangerous?** It lets a user hand a role to a service (e.g. launch EC2 with a role). With unrestricted PassRole a user can pass an admin role and escalate privileges. Limit it to specific role ARNs and services with `iam:PassedToService`.

**26. How do pods in EKS get AWS permissions?** IRSA (IAM Roles for Service Accounts, via OIDC) or the newer EKS Pod Identity. Each Kubernetes service account maps to its own role, so pods don't share the node's role.

**27. How does GitHub Actions or Azure DevOps deploy to AWS without stored keys?** OIDC federation: add the CI provider as an OIDC identity provider in IAM, create a role that trusts it with conditions on repo/branch, and the pipeline calls AssumeRoleWithWebIdentity.

**28. What is ABAC?** Attribute-based access control: permissions based on tags, e.g. allow only if `aws:PrincipalTag/team` equals `aws:ResourceTag/team`. Scales better than writing a policy per project.

**29. An SCP allows `s3:*` but the user's identity policy doesn't mention S3. Can they use S3?** No. SCPs don't grant. The identity policy (or a resource policy) must also allow it.

**30. Can an explicit Allow override an SCP Deny?** No. Explicit deny anywhere, including SCPs and boundaries, always wins.

## Scenario questions

Answer pattern for every scenario: **identify the identity → find the right mechanism (role, policy, condition) → apply least privilege → say how you'd verify and monitor.**

**S1. A developer pushed AWS access keys to a public GitHub repo. What do you do?**

1. Deactivate the key immediately, then delete it.
2. Check CloudTrail for every call made with that key (look for new users, roles, EC2 instances in unusual regions, cost spikes).
3. Remove anything the attacker created; check GuardDuty findings.
4. Purge the secret from Git history; issue new credentials.
5. Prevent repeat: use roles/OIDC instead of keys, add secret scanning (GitHub push protection, git-secrets) in the pipeline.

**S2. An app on EC2 needs to read from one S3 bucket. How do you set up access?** Create a role trusted by `ec2.amazonaws.com` with a policy allowing `s3:GetObject` (and `s3:ListBucket` if needed) on that bucket only. Attach it via an instance profile. No keys on the server. Enforce IMDSv2.

**S3. A user gets AccessDenied even though their policy allows the action. How do you troubleshoot?**

1. `aws sts get-caller-identity` — confirm which identity is actually calling.
2. Read the error message; it often names the policy type that denied it (SCP, boundary, resource policy).
3. Check for an explicit Deny in identity, group, boundary, SCP, resource policy, or session policy.
4. Check the resource ARN and conditions (wrong region, missing MFA, IP, tags).
5. For S3/KMS, check the bucket policy and key policy too.
6. Use the Policy Simulator and CloudTrail to confirm.

**S4. Your company has 20 AWS accounts and 200 employees. How do you manage access?** AWS Organizations with OUs (prod, non-prod, security). IAM Identity Center connected to the corporate IdP (e.g. Entra ID). Permission sets per job role (Admin, Developer, ReadOnly), assigned to groups per account. SCPs for guardrails. No IAM users.

**S5. A vendor needs read-only access to your account to monitor costs.** Create a cross-account role trusting the vendor's account ID with an `sts:ExternalId` condition. Attach only the read permissions they need. Never share access keys. Review with Access Analyzer.

**S6. Developers must be able to create IAM roles for their Lambdas, but must not become admins.** Give them `iam:CreateRole`/`iam:AttachRolePolicy` only with a condition that the new role must have a specific **permissions boundary** attached. Restrict `iam:PassRole` to roles with a set name prefix and to `lambda.amazonaws.com`.

**S7. Interns should never be able to delete production resources, even by mistake.** Use an explicit Deny on `*:Delete*` / `ec2:TerminateInstances` for prod-tagged resources, or an SCP on the prod OU. Explicit deny wins over any allow they get later.

**S8. Enforce MFA for anyone using the console or CLI.** Attach a policy that denies all actions except MFA self-setup when `aws:MultiFactorAuthPresent` is false. For CLI, users call `sts get-session-token` with their MFA code. Better: move users to Identity Center where MFA is enforced at login.

**S9. Your Azure DevOps / GitHub Actions pipeline deploys to AWS with stored access keys. Improve it.** Replace the keys with OIDC: register the provider in IAM, create a deploy role trusting it, restrict the trust policy to your org/repo/branch, and grant only deployment permissions. Credentials become short-lived and nothing is stored.

**S10. Two pods in the same EKS cluster need different AWS permissions.** Use IRSA or EKS Pod Identity: a separate IAM role per Kubernetes service account. Don't widen the node role, because every pod on the node would inherit it.

**S11. Make sure nobody in the organization can disable CloudTrail or use unapproved regions.** SCP on the root/OU: Deny `cloudtrail:StopLogging` and `cloudtrail:DeleteTrail`; Deny all actions when `aws:RequestedRegion` is not in your allowed list (exclude global services like IAM).

**S12. Audit asks: which IAM users haven't logged in or used keys in 90 days?** Download the **credential report** (password last used, access key last used). Disable unused ones. Use Access Analyzer's unused-access findings and Access Advisor to trim permissions.

**S13. A team member left the company. What do you do?** Disable them in the IdP/Identity Center (or delete the IAM user's console password and keys), remove from groups, rotate any shared secrets they knew, and check CloudTrail for recent activity.

**S14. Someone in another account needs to put objects in your S3 bucket.** Two sides must allow it: your bucket policy allows their principal `s3:PutObject`, and their identity policy allows `s3:PutObject` on your bucket. Use bucket-owner-enforced object ownership so you own the uploaded objects.

## Policy JSON examples

Interviewers often ask you to write or read one of these. Know the shape by heart.

**1. Read-only access to one S3 bucket (identity policy)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-app-bucket"
    },
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    }
  ]
}
```

Note: `ListBucket` applies to the bucket ARN; `GetObject` applies to objects (`/*`). Mixing these up is a common mistake.

**2. Trust policy for an EC2 role**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**3. Cross-account trust with External ID**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:root" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": { "sts:ExternalId": "vendor-unique-id-123" }
      }
    }
  ]
}
```

`:root` here means "the whole account 111122223333", not its root user. That account must still grant its own users `sts:AssumeRole`.

**4. Deny everything without MFA**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "NotAction": ["iam:*MFADevice*", "iam:ListMFADevices", "iam:GetUser", "sts:GetSessionToken"],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" }
      }
    }
  ]
}
```

**5. SCP: block unapproved regions**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "NotAction": ["iam:*", "sts:*", "organizations:*", "support:*", "cloudfront:*", "route53:*"],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": { "aws:RequestedRegion": ["ap-south-1", "us-east-1"] }
      }
    }
  ]
}
```

**6. Trust policy for GitHub Actions OIDC (one repo, main branch)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**7. Tag-based access (ABAC)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:StartInstances", "ec2:StopInstances"],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": { "aws:ResourceTag/team": "${aws:PrincipalTag/team}" }
      }
    }
  ]
}
```

## Hands-on practice

Do these in a free-tier account before the interview, so you can say "I've done this" instead of "I've read this".

- [ ] Enable MFA on root; create an admin user/Identity Center user and stop using root
- [ ] Create a group `Developers`, attach a customer-managed policy, add a user
- [ ] Write the one-bucket S3 read-only policy and test it with the Policy Simulator
- [ ] Launch an EC2 instance with a role; from inside run `aws s3 ls` with no keys configured
- [ ] Create a role in a second account (or same account) and assume it from CLI
- [ ] Trigger an AccessDenied on purpose and trace it in CloudTrail
- [ ] Download the credential report and read it
- [ ] Turn on IAM Access Analyzer and review its findings
- [ ] Set up OIDC from GitHub Actions or Azure DevOps to AWS and deploy without stored keys

**CLI commands to know**

```bash
# Who am I?
aws sts get-caller-identity

# Users, groups, roles
aws iam create-user --user-name dev1
aws iam create-group --group-name Developers
aws iam add-user-to-group --user-name dev1 --group-name Developers
aws iam list-users
aws iam list-roles

# Policies
aws iam create-policy --policy-name S3ReadOne --policy-document file://policy.json
aws iam attach-group-policy --group-name Developers --policy-arn arn:aws:iam::123456789012:policy/S3ReadOne
aws iam list-attached-user-policies --user-name dev1

# Roles
aws iam create-role --role-name AppRole --assume-role-policy-document file://trust.json
aws sts assume-role --role-arn arn:aws:iam::123456789012:role/AppRole --role-session-name test

# Access keys
aws iam list-access-keys --user-name dev1
aws iam update-access-key --user-name dev1 --access-key-id AKIA... --status Inactive

# Audit
aws iam generate-credential-report
aws iam get-credential-report
```

**Terraform angle (DevOps interviews)** Be ready to describe creating a role with `aws_iam_role`, a policy with `aws_iam_policy` or the `aws_iam_policy_document` data source, and linking them with `aws_iam_role_policy_attachment`.

## Cheat sheet (revise 10 minutes before the interview)

| Remember | One line |
| --- | --- |
| Scope | IAM is global and free |
| Evaluation | Explicit Deny > Allow > Implicit Deny |
| Role = | Trust policy (who) + permissions policy (what) + temporary creds |
| Humans | Identity Center / federation, MFA, no long-term keys |
| Workloads | Roles: instance profile (EC2), execution role (Lambda), IRSA/Pod Identity (EKS), OIDC (CI/CD) |
| Guardrails | SCP (org-wide cap), permissions boundary (per identity cap) — neither grants |
| Cross-account | Both sides must allow; ExternalId for vendors |
| Escalation risks | `iam:PassRole`, `iam:CreatePolicyVersion`, `iam:AttachUserPolicy` on yourself |
| Debug AccessDenied | get-caller-identity → explicit denies → resource policy → conditions → CloudTrail |
| Audit tools | Credential report, Access Advisor, Access Analyzer, CloudTrail |
| Max keys per user | 2 (for rotation) |
| Max groups per user | 10; groups can't nest |

**Your own story:** link IAM to your internship work. Azure RBAC and managed identities (used with AKS) map to IAM policies/roles and IRSA/Pod Identity in AWS. Interviewers like candidates who can map concepts across clouds.
