# Lab M2.05 - IAM Roles for EC2

**Repository:** [https://github.com/cloud-engineering-bootcamp/ce-lab-iam-roles-for-ec2](https://github.com/cloud-engineering-bootcamp/ce-lab-iam-roles-for-ec2)

**Activity Type:** Individual  
**Estimated Time:** 45-60 minutes

## Learning Objectives

- [ ] Create an IAM role with an EC2 trust relationship
- [ ] Write a least-privilege policy scoped to specific resources
- [ ] Assign an IAM role to a running EC2 instance
- [ ] Test AWS API access using temporary role credentials
- [ ] Verify that no long-lived credentials exist on the instance

## Prerequisites

- [ ] Completed Lab M2.01 (EC2 instance running, Amazon Linux 2023)
- [ ] SSH access to your instance
- [ ] IAM permissions to create roles and policies

---

## Introduction

Long-lived AWS credentials should not be stored on a server. This excludes `aws configure`, access keys in environment variables, and keys embedded in application code.

The recommended approach is to attach an **IAM role** to the instance. AWS then delivers short-lived, automatically rotating credentials to the instance through the instance metadata service, and the AWS CLI and SDKs retrieve them without additional configuration. If the instance is compromised, the exposed credentials expire within hours and are scoped to the permissions defined in the role, rather than being a permanent key with broad access.

## Scenario

Your application needs to read and write objects in one specific S3 bucket and send logs to CloudWatch, and requires no other permissions. Your security team does not permit solutions that use access keys, nor policies that grant `s3:*` on `*`.

---

## Your Task

**What you'll create:**
- An S3 bucket for the app to use
- A custom IAM policy scoped to that one bucket
- An IAM role trusted by EC2, with that policy plus CloudWatch access
- The role attached to your running instance

**Time limit:** 45-60 minutes

---

## Step-by-Step Instructions

### Step 1: Create an S3 Bucket

The role needs something to be scoped to. Bucket names are **globally unique** across all of AWS, so include your name.

```bash
# Run from your laptop, where the AWS CLI is already configured
aws s3 mb s3://ce-bootcamp-m2-05-YOURNAME   # mb = "make bucket"
aws s3 ls                                   # list all buckets to confirm it exists
```

**Write down your bucket name:** ce-bootcamp-m2-05-balint

**Expected outcome:** The bucket appears in `aws s3 ls`.

---

### Step 2: Review the Policy Before Applying It

Review the following policy, paying particular attention to the `Resource` block, which is a common source of errors.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::ce-bootcamp-m2-05-balint/*",
      "arn:aws:s3:::ce-bootcamp-m2-05-balint"
    ]
  }]
}
```

**Why the policy includes two ARNs, one with `/*` and one without:**

```
arn:aws:s3:::YOUR-BUCKET-NAME       <- the bucket   -> s3:ListBucket
arn:aws:s3:::YOUR-BUCKET-NAME/*     <- the objects  -> s3:GetObject, s3:PutObject
```

`ListBucket` is an operation performed on the bucket. `GetObject` and `PutObject` are performed on objects. These are different resource types and require different ARNs.

If the bare bucket ARN is omitted, uploads succeed but `aws s3 ls` fails with `AccessDenied`.

---

### Step 3: Create the Custom Policy

1. **IAM Console** → **Policies** → **Create policy**

2. Select the **JSON** tab and paste the policy from Step 2

3. **Replace both occurrences** of `YOUR-BUCKET-NAME` with your real bucket name

4. **Next** → **Policy name:** `ec2-s3-access-policy`

5. **Create policy**

**Expected outcome:** The policy appears in your Customer managed policies list.

---

### Step 4: Create the IAM Role

1. **IAM Console** → **Roles** → **Create role**

2. **Trusted entity type:** AWS service

3. **Use case:** **EC2** → **Next**

   > This designates the role as an EC2 role. AWS generates a **trust policy**
   > stating that the EC2 service is permitted to assume this role. This file is
   > required for your submission; see Step 9.

4. **Add permissions** - tick both:
   - `CloudWatchAgentServerPolicy` (AWS managed)
   - `ec2-s3-access-policy` (the one you just made)

5. **Next** → **Role name:** `ec2-s3-cloudwatch-role`

6. **Create role**

**Expected outcome:** The role exists with exactly two policies attached.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/01-role-creation.png
```

**Capture**

The `ec2-s3-cloudwatch-role` summary page in the IAM console, showing the role was created with an EC2 (AWS service) trusted entity.

**Purpose**

Verifies the IAM role was created with the correct EC2 trust relationship.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/02-policy-attachment.png
```

**Capture**

The role's **Permissions** tab, showing both attached policies:

- `CloudWatchAgentServerPolicy`
- `ec2-s3-access-policy`

**Purpose**

Verifies exactly the two required policies are attached to the role - no more, no less.

---

### Step 5: Attach the Role to Your Instance

1. **EC2 Console** → **Instances** → select your instance

2. **Actions** → **Security** → **Modify IAM role**

3. Select `ec2-s3-cloudwatch-role` → **Update IAM role**

**Expected outcome:** The instance's Details tab shows the IAM role. No reboot or reconnect is needed - it takes effect within seconds.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/03-ec2-with-role.png
```

**Capture**

The EC2 instance **Details** tab, with the **IAM Role** field showing `ec2-s3-cloudwatch-role`.

**Purpose**

Confirms the role is attached to the running instance, which is what supplies the temporary credentials used in the tests that follow.

---

### Step 6: Confirm No Credentials Exist on the Instance

Before testing, confirm the instance has no stored credentials:

```bash
ls -la ~/.aws/ 2>/dev/null || echo "No ~/.aws directory."
```

> **If `~/.aws/credentials` exists, delete it.** Static credentials take
> precedence over the instance role and will override it. With static credentials
> present, the tests would pass without exercising the role.
> ```bash
> rm -f ~/.aws/credentials
> ```

---

### Step 7: Test the Role

On the instance, with **no `aws configure` run**:

```bash
# Who am I?
aws sts get-caller-identity
```

**Expected output** - note `assumed-role`, not `user`:
```json
{
    "UserId": "AROA...:i-0abc123",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/ec2-s3-cloudwatch-role/i-0abc123"
}
```

#### 📸 Screenshot Required

**Filename**

```
screenshots/04-assumed-role-identity.png
```

**Capture**

The terminal on the instance showing `aws sts get-caller-identity` and its output, where the `Arn` contains `assumed-role/ec2-s3-cloudwatch-role/...`.

**Purpose**

Proves the instance is authenticating as the IAM role, not an IAM user, with no `aws configure`.

---

```bash
# Can I list my bucket?
aws s3 ls s3://YOUR-BUCKET-NAME/

# Can I write to it?
echo "written by an IAM role, no keys involved" > test.txt
aws s3 cp test.txt s3://YOUR-BUCKET-NAME/

# Can I read it back?
aws s3 cp s3://YOUR-BUCKET-NAME/test.txt -
```

**Expected outcome:** All succeed, and you never typed a credential.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/05-s3-upload-success.png
```

**Capture**

The terminal showing the successful `aws s3 cp test.txt s3://YOUR-BUCKET-NAME/` upload (optionally with the read-back that prints the file contents).

**Purpose**

Demonstrates the role can perform the S3 actions it was granted, using role-supplied credentials.

---

### Step 8: Test Least Privilege

A policy that permits the required actions must also deny everything else. Verify that unauthorized actions are denied.

```bash
# A bucket that was not granted access
aws s3 ls s3://some-other-bucket-name/

# An action that was not allowed (the policy grants Get/Put/List, not Delete)
aws s3 rb s3://YOUR-BUCKET-NAME --force
```

**Expected outcome:** Both fail with `AccessDenied`. This is the correct result and demonstrates least privilege. Capture a screenshot as evidence.

---

### 📸 Screenshot Required

**Filename**

```
screenshots/06-access-denied-proof.png
```

**Capture**

The terminal showing an `AccessDenied` error for an action the policy does not grant - either listing a bucket you were not given access to, or the `s3 rb` delete attempt.

**Purpose**

Proves the role denies everything outside its granted permissions - the definition of least privilege.

---

### Step 9: Capture the Trust Policy for Submission

The console generated the trust policy invisibly when you chose "EC2". Retrieve it.

> **Run this from your laptop, not the instance.** The instance role grants only
> S3 and CloudWatch actions and has no `iam:*` permissions, so running it on the
> instance returns `AccessDenied`. This is expected and confirms least privilege:
> the role cannot inspect IAM, including itself. Your laptop's own credentials
> have the required access.

```bash
# On your laptop. get-role returns the full role; --query extracts only the trust policy document
aws iam get-role --role-name ec2-s3-cloudwatch-role \
  --query 'Role.AssumeRolePolicyDocument'
```

Save the output as `trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

This statement permits the EC2 service to assume the role. It is what allows the instance to use the role while preventing other principals from doing so.

---

### Step 10: Locate the Source of the Credentials

The following demonstrates where the credentials originate.

> **Run this on the instance.** It queries the **Instance Metadata Service (IMDS)**,
> a special address (`169.254.169.254`) reachable only from within the instance
> itself. IMDS exposes data about the instance, including the temporary
> credentials delivered by the attached IAM role.
>
> Amazon Linux 2023 uses **IMDSv2**, the hardened version of this service. Instead
> of allowing a plain request, IMDSv2 requires you to first obtain a short-lived
> session token (`PUT /latest/api/token`) and then send that token with each
> metadata request. This two-step handshake blocks a common attack where a
> tricked application is made to fetch credentials from the metadata endpoint.

```bash
# Step 1: request a short-lived session token (required by IMDSv2)
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")

# Step 2: use the token to read the role's temporary credentials from the metadata service
curl -sH "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-s3-cloudwatch-role
```

**Expected outcome:** The response contains `AccessKeyId`, `SecretAccessKey`, `Token`, and an `Expiration` timestamp several hours in the future. AWS rotates these credentials automatically, and the CLI retrieves them from this endpoint on each call. The `Expiration` field is the defining characteristic of role-based credentials.

---

## 📤 What to Submit

**Submission Type:** GitHub Repository

Create a **public** GitHub repository named `ce-lab-iam-roles-ec2` containing:

1. **IAM Policy Files:**
   - `s3-cloudwatch-policy.json` (your custom IAM policy)
   - `trust-policy.json` (EC2 assume role policy)

2. **Test Output:**
   - `aws-test-commands.txt` (commands you ran and their output)
   - `s3-test-output.txt` (S3 upload test results)
   - `access-denied-test.txt` (proof least privilege works)

3. **Screenshots** (in `screenshots/` folder):
   - IAM role creation in console
   - Policy attachment
   - EC2 instance with role attached
   - `aws sts get-caller-identity` showing assumed role
   - Successful S3 file upload
   - `AccessDenied` on a bucket you did not grant

4. **Documentation** (`README.md`):
   - Why IAM roles are better than access keys
   - Step-by-step process
   - Policy explanation (what you granted and why)
   - Why the policy needs two ARNs
   - Any troubleshooting steps

**Structure:**
```
ce-lab-iam-roles-ec2/
├── README.md
├── policies/
│   ├── s3-cloudwatch-policy.json
│   └── trust-policy.json
├── test-output/
│   ├── aws-test-commands.txt
│   ├── s3-test-output.txt
│   └── access-denied-test.txt
└── screenshots/
    ├── 01-role-creation.png
    ├── 02-policy-attachment.png
    ├── 03-ec2-with-role.png
    ├── 04-assumed-role-identity.png
    ├── 05-s3-upload-success.png
    └── 06-access-denied-proof.png
```

> ⚠️ **This repository is public.** Your outputs contain your 12-digit AWS
> **account ID**. That is not a credential, but it is account-identifying.
> Redact it as `123456789012` in committed files if you prefer.
>
> If you ever paste a real `SecretAccessKey` or `Token` from Step 10 into a
> public repo, **delete the repo and rotate immediately** - those are live
> credentials until they expire.

---

## Bonus Challenges

**+5 points each:**

- [ ] **Add a `Condition`** to the policy so it only allows uploads with server-side encryption (`s3:x-amz-server-side-encryption`)
- [ ] **Scope it tighter:** restrict access to a single *prefix* (`/uploads/*`) rather than the whole bucket
- [ ] **Write to CloudWatch Logs** using the `CloudWatchAgentServerPolicy` you attached but never used
- [ ] **Demonstrate the ARN requirement:** remove the bare-bucket ARN, re-test, and document which command begins failing and why

---

## Common Issues & Solutions

### Issue 1: `aws s3 ls` says AccessDenied, but uploads work

**Cause:** Your policy has only the `/*` object ARN. `s3:ListBucket` acts on the **bucket**, which needs the bare ARN.

**Solution:** Add `"arn:aws:s3:::YOUR-BUCKET-NAME"` (no `/*`) to the Resource list.

---

### Issue 2: `get-caller-identity` shows an IAM **user**, not an assumed-role

**Cause:** Static credentials on the instance are overriding the role. Someone ran `aws configure`.

**Solution:**
```bash
rm -f ~/.aws/credentials
aws sts get-caller-identity     # should now show assumed-role/...
```

---

### Issue 3: "Unable to locate credentials"

**Causes:**
1. The role is not actually attached to this instance
2. You attached it to a *different* instance

**Solution:** EC2 Console → Instance → **Details** tab → check **IAM Role** is populated. Then re-test; it applies within seconds.

---

### Issue 4: Metadata call returns `401 Unauthorized`

**Cause:** Amazon Linux 2023 requires **IMDSv2** - you must fetch a token first.

**Solution:** Use the two-step `TOKEN=$(curl -X PUT ...)` form from Step 10.

---

## Learning Reflections

Answer these in your README:

1. What on the instance enables the AWS CLI to authenticate? Trace the path from `aws s3 ls` to the credential.
2. If an attacker obtains a shell on the instance, what credentials can they access, and how does this differ from a stolen access key on a laptop?
3. Why does the trust policy matter? What would occur if it specified `"Service": "lambda.amazonaws.com"` instead?
4. The policy allows `s3:PutObject` but not `s3:DeleteObject`. Why might this distinction be significant in production?

---

## Cleanup

- [ ] Empty and delete the S3 bucket: `aws s3 rb s3://YOUR-BUCKET-NAME --force`
- [ ] Instance **stopped** (not terminated - later labs reuse it)
- [ ] You may leave the IAM role; it costs nothing

---

## Screenshot Checklist

Before submitting, verify that the following screenshots are present in the `screenshots/` folder:

- [ ] `screenshots/01-role-creation.png`
- [ ] `screenshots/02-policy-attachment.png`
- [ ] `screenshots/03-ec2-with-role.png`
- [ ] `screenshots/04-assumed-role-identity.png`
- [ ] `screenshots/05-s3-upload-success.png`
- [ ] `screenshots/06-access-denied-proof.png`

---

## Submission Checklist

- [ ] GitHub repository created and **public**
- [ ] S3 bucket created and scoped in the policy
- [ ] Policy uses **both** ARN forms (bucket and objects)
- [ ] Role created with EC2 trust relationship
- [ ] Role attached to running instance
- [ ] `aws sts get-caller-identity` shows `assumed-role/`
- [ ] `~/.aws/credentials` does **not** exist on the instance
- [ ] `AccessDenied` captured for a bucket you did not grant
- [ ] Account ID redacted (if you chose to)
- [ ] Repository URL submitted

---

## Grading: 100 points

| Criteria | Points |
|----------|--------|
| **IAM role creation** (correct EC2 trust relationship) | 25 |
| **Policy configuration** (least privilege, both ARNs correct) | 30 |
| **Testing and verification** (incl. proving AccessDenied) | 25 |
| **Documentation and security analysis** | 20 |
| **Total** | **100** |
