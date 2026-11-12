# IAM — Identity and Access Management

Controls **who can call which AWS API**. Global (not per region).

## Root user
The sign-up email. Can do everything, including closing the account. → MFA on, **no access keys**, used only for account-level tasks (billing, first admin). Daily work = an IAM identity.

## Identities
| | What | Credentials |
|---|---|---|
| **User** | a person or a tool | password (console) and/or keys |
| **Group** | set of users sharing permissions (e.g. `admin`) | none — not an identity you log in as |
| **Role** | permissions something **borrows temporarily** (EC2 server, Lambda, CI job, another account) | temporary, auto-rotated (~1 h) |

**Why roles for servers:** keys in a file on a server live forever → leaked via hack, disk snapshot or a git commit = valid forever, from anywhere. A role's credentials come from AWS automatically and expire → nothing to leak or forget.
> "Roles mean no long-lived secrets on servers."

## Policies — JSON documents
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadDbSecret",
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:eu-central-1:<account-id>:secret:prod/postgres-*",
      "Condition": { "Bool": { "aws:SecureTransport": "true" } }
    }
  ]
}
```
| Field | Meaning |
|---|---|
| `Version` | policy language version — always `2012-10-17` |
| `Sid` | optional label (Statement ID), can be left out |
| `Effect` | `Allow` or `Deny` |
| `Action` | `service:APIOperation`, wildcards allowed (`s3:Get*`) |
| `Resource` | **ARN** = address of anything in AWS: `arn:aws:<service>:<region>:<account-id>:<resource>` |
| `Condition` | optional "only if…": IP range, MFA present, HTTPS, resource tag |
| `Principal` | *who* — only in resource-based and trust policies |

**Finding names:** never memorise. Console visual editor lists actions per service · docs "Service Authorization Reference" (all actions/resources/conditions per service) · AccessDenied errors name the missing action. ARNs: copy from the resource's console page, or reference them in Terraform (`aws_s3_bucket.logs.arn`).

## How AWS decides (evaluation)
1. Any **explicit Deny**? → denied. Nothing overrides it.
2. Otherwise any **Allow**? → allowed.
3. Otherwise → denied (**default deny**).

Permissions **add up** across user policies and all groups. *Not mentioning* an action ≠ Deny.
Example: user policy allows GET, group allows GET + PUT → **GET + PUT**. To block PUT for that user: explicit `Deny s3:PutObject`.
> "Explicit deny > allow > default deny."

**Least privilege** is a design habit, not an evaluation rule: grant only the actions + resources needed.

## Three kinds of policy
| Kind | Attached to | Answers |
|---|---|---|
| **Identity-based** | user / group / role | what can I do? |
| **Resource-based** | a resource, e.g. S3 bucket policy | who can touch me? |
| **Trust policy** | a role | who can **become** me? |

A role = a uniform: **trust policy** = who may wear it (e.g. `"Principal": {"Service": "ec2.amazonaws.com"}`, `"Action": "sts:AssumeRole"`), **permissions policy** = what the wearer may do.

### Common mix-ups
- ❌ "Policies are for users, roles are for machines." → Users **and** roles are both *identities*; policies attach to both. Role = identity, policy = permissions.
- ❌ "A role is how two EC2s talk to each other." → Typical case: a machine calling an **AWS service** (EC2 → S3). EC2-to-EC2 network traffic is security groups, not IAM.
- ❌ "Roles are only for machines." → Mostly, but humans assume them too (cross-account access, SSO).
- ❌ Forgetting the trust policy → the role exists with permissions but **nobody can assume it** (keys locked in the car). Users don't need one; they *are* the login.
- ❌ "One policy per user." → A user can have several policies directly **plus** everything from all its groups; they add up.
- ❌ "Custom = inline." → Custom policies are usually **customer managed** (standalone, reusable); inline is a separate type, embedded in one identity.

## Managed vs inline
| Type | What | Use |
|---|---|---|
| AWS managed | ready-made (`AdministratorAccess`, `ReadOnlyAccess`, `AmazonS3ReadOnlyAccess`…) | quick, often broad |
| Customer managed | written by you, reusable | the normal choice |
| Inline | embedded in one identity, dies with it | rare one-offs |

Account-wide read access: **`ReadOnlyAccess`** (incl. contents, e.g. S3 objects, logs) vs **`ViewOnlyAccess`** (what exists + configuration, not contents). Both are "job function" policies — filter the console list by "AWS managed - job function"; otherwise they drown among hundreds of `<Service>ReadOnlyAccess` policies (e.g. `AgentRegistryReadOnlyAccess` = one service only).

Creating policies: console (visual or JSON) · CLI (`aws iam create-policy --policy-document file://p.json`) · **Terraform** (`aws_iam_policy`) in real teams. Test with the **IAM Policy Simulator** and **Access Analyzer**.

## CLI sign-in with `aws login` (CLI ≥ 2.32)
- Browser sign-in with console credentials → **temporary** credentials (12 h, refreshed automatically, renewable up to 90 days), cached locally. No access keys on disk.
- `aws login --profile X` · `aws logout --profile X` · `aws sts get-caller-identity --profile X` (who am I?) · `aws configure list --profile X` (where creds come from; TYPE should be `login`).
- IAM users need **`SignInLocalDevelopmentAccess`** (or admin); root needs nothing.

### Pitfalls
- **Browser session reuse:** if the browser is already signed in (e.g. as root), `aws login` silently binds the profile to *that* identity. Always check `get-caller-identity`; sign out in the browser or use a private window with `--remote`.
- Console login works but CLI login fails with "invalid request" → missing `SignInLocalDevelopmentAccess`.
- `--remote` (paste a code) needs an interactive terminal.
- A closed AWS account's email can never be reused for a new account.

## Least privilege for automation and AI agents
Each tool gets **its own identity** with only what it needs — never a human admin's login.
| Identity | Permissions | Used by |
|---|---|---|
| admin user | `AdministratorAccess` | the human: console, CLI, Terraform |
| read-only user | `ViewOnlyAccess` / `ReadOnlyAccess` | AI assistant via the AWS MCP server |

- AWS MCP server picks its identity from `AWS_MCP_PROXY_PROFILES` in the MCP config.
- Verify with a **DryRun** write: `ec2:CreateSecurityGroup` with `DryRun=true` → `UnauthorizedOperation … no identity-based policy allows` = default deny working.
- Calls via AWS-managed MCP servers carry **`aws:ViaAWSMCPService = true`** → a policy can Deny writes only when they come through MCP.
- Plus a guard on the agent side (tool deny rules: no `aws` CLI, no credentials folder, no `terraform apply`), because tokens of all profiles sit on the same machine.
