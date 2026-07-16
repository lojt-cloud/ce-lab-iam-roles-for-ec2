# IAM Roles for EC2 - Lab Documentation

**Student:** Balint Lojt
**Date:** 16/07/2026

## Why IAM Roles Are Better Than Access Keys

IAM roles provide temporary, automatically-rotating credentials instead of
permanent secret keys. When an EC2 instance has a role attached, it never
stores a static AccessKeyId/SecretAccessKey pair anywhere - the AWS CLI
retrieves short-lived credentials from the Instance Metadata Service on
demand, and those credentials expire within hours regardless of whether
anyone notices a problem.

This matters most in a compromise scenario. If an attacker got shell
access to an instance using a role, they could only access whatever
narrow permissions that role's policy grants, and only for a few hours at
most. A stolen static access key, by contrast, works indefinitely - often
with much broader permissions - until a human manually notices and
revokes it, which could take hours, days, or never happen at all. Roles
remove the single biggest risk of static keys: a long-lived secret sitting
somewhere it can be found and reused.

## Step-by-Step Process
1. Created an S3 bucket (`ce-bootcamp-m2-05-balint`) for the app to use

2. Wrote a custom IAM policy (`ec2-s3-access-policy`) granting only
   GetObject, PutObject, and ListBucket, scoped to that one bucket
   specifically (both the bucket ARN and the object ARN with /*)

3. Created an IAM role (`ec2-s3-cloudwatch-role`) with an EC2 trust
   relationship, attaching both the custom S3 policy and the AWS-managed
   CloudWatchAgentServerPolicy

4. Attached the role to a running EC2 instance via Actions → Security →
   Modify IAM role

5. Confirmed no static credentials existed on the instance beforehand

6. Tested authentication with `aws sts get-caller-identity`, confirming
   the CLI authenticated as `assumed-role/ec2-s3-cloudwatch-role`, with
   no `aws configure` ever run

7. Tested the actual granted permissions - listing, uploading, and
   reading back a file from the bucket, all successful

8. Tested least privilege by attempting to list a bucket never granted
   access, and attempting to delete the bucket itself - both correctly
   failed with AccessDenied

9. Retrieved the trust policy from my laptop (not the instance, since the
   role has no IAM permissions to inspect itself) to confirm the EC2
   trust relationship

10. Queried the Instance Metadata Service directly to see the actual
    temporary credentials being issued to the role, confirming the
    Expiration field and the auto-rotation mechanism

## Policy Explanation


**What was granted and why:**

The policy grants exactly three actions - `s3:GetObject`, `s3:PutObject`,
and `s3:ListBucket` - scoped only to the one bucket this instance actually
needs. GetObject and PutObject were required since the scenario states the
application needs to read and write objects. ListBucket was included so
the application (and I, when testing) can actually see what's in the
bucket - without it, `aws s3 ls` fails with AccessDenied even though
uploads/downloads still work, since ListBucket operates on the bucket
resource itself, not on individual objects.

Deliberately excluded: `s3:DeleteObject` and `s3:DeleteBucket`. The
scenario never required delete access, and removing it limits the
worst-case damage if the role were ever misused - the instance can add or
overwrite files, but can never permanently destroy them. This is least
privilege in practice: granting exactly what's needed, and no more, even
when a broader grant would technically still "work" for the stated task.

## Troubleshooting

**Confused a new instance with the required prerequisite instance:** 
I initially wasn't sure if I needed to reuse my existing EC2 instance from
Lab M2.01, or launch a fresh one specifically for this lab, and ended up
launching a new instance (`ec2-s3-instance`). This turned out to be fine
- the lab's prerequisite just requires *an* instance to exist, not
specifically the original one - but it did mean I had to set up SSH access
(key pair, security group, SSH config alias) for this instance separately
before continuing.

**Ran a command with the placeholder text still in it:** 
When fetching the custom policy JSON with the AWS CLI, I copied a command containing
`YOUR_ACCOUNT_ID` as a literal placeholder and ran it without substituting
my real account ID first. This produced an `AccessDenied` error that
initially looked like a real permissions problem, even though my IAM user
has admin access. The actual cause was that AWS was trying to look up a
policy on a literally nonexistent account ("YOUR_ACCOUNT_ID"), not a real
authorization issue. Fixed by re-running the command with my actual
account ID substituted in.

**Understanding IMDSv2's two-step handshake:** 
Initially expected to query the metadata endpoint directly with a single curl request, but Amazon
Linux 2023 enforces IMDSv2, which requires first requesting a short-lived
session token via a PUT request, then including that token as a header on
the actual metadata request. Both steps are required - a plain GET without
the token header returns a 401 Unauthorized.

## Reflection Questions


1. What on the instance enables the AWS CLI to authenticate? Trace the path from aws s3 ls to the credential.

When I run a command like aws s3 ls, the AWS CLI first checks for local static credentials (like a ~/.aws/credentials file) - I confirmed in Step 6 that none exist on this instance. With no static credentials found, the CLI automatically falls back to checking the Instance Metadata Service (IMDS) at 169.254.169.254, a special internal address only reachable from inside the instance itself, never from the public internet.
This all happens silently, behind the scenes, every time I run any aws command - I never see it happen directly. But in Step 10 I ran the same sequence manually to see it for myself:
First, the CLI requests a short-lived session token (a ticket to enter), since this instance uses the hardened IMDSv2:
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
Then, using that token, it reads the role's temporary credentials from the metadata service:
curl -sH "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-s3-cloudwatch-role
This returns the AccessKeyId, SecretAccessKey, Token, and an Expiration timestamp. The CLI uses these values to sign and send the actual aws s3 ls request. Before this credential set expires, AWS automatically issues a new one, so even if someone captured a set of these credentials, they would stop working once expired - unlike a permanent access key.
The way I think about it: requesting the token is like getting a movie ticket at the cinema - it lets me into the credentials for a limited time. With that ticket, I can ask for the specific seat (the actual AccessKeyId/SecretAccessKey/Token) to watch the movie. Since the ticket is temporary, before it expires, a new one is automatically issued so the movie (my session) never actually gets interrupted.


2. If an attacker obtains a shell on the instance, what credentials can they access, and how does this differ from a stolen access key on a laptop?

If an attacker got shell access to this instance, they could query IMDS themselves and retrieve the role's temporary credentials - but those credentials are scoped only to what the role's policy allows (in this case, read/write on one specific S3 bucket, nothing else). They couldn't delete anything, touch other buckets, or access other AWS services beyond what was explicitly granted. The credentials also expire within hours, so any specific set they captured and tried to reuse elsewhere (off the instance) becomes useless relatively quickly - though as long as they still have shell access, they could keep re-querying IMDS for fresh ones.
A stolen static access key from a laptop is far more dangerous on both dimensions. Depending on the permissions attached to that key's user, the damage could be much broader - potentially spanning the whole account rather than one bucket - including the ability to delete, modify, or use that access to explore and move deeper into connected systems. Critically, static access keys never expire on their own, so the compromise persists until a human notices and manually revokes it - which could take hours, days, or, in the worst case, never happen at all if nobody catches it.


3. Why does the trust policy matter? What would occur if it specified "Service": "lambda.amazonaws.com" instead?

The trust policy controls a different question than the permissions policy does - it decides which type of AWS service is even allowed to assume the role in the first place, separately from what that role can do once assumed. If the trust policy specified lambda.amazonaws.com instead of ec2.amazonaws.com, only Lambda functions would be permitted to assume this role - my EC2 instance would be locked out entirely, even though the S3 and CloudWatch permissions policy is still perfectly correctly configured.
In practice, this would mean every aws command on the instance would fail with something like "Unable to locate credentials," since EC2 was never granted permission to use the role at all. The permissions being correct wouldn't matter, since the instance can't even get in the door to use them in the first place.


4. The policy allows s3:PutObject but not s3:DeleteObject. Why might this distinction be significant in production?

This distinction limits the worst-case damage from a bug or a compromised credential. If something goes wrong - a bug in the application code, a misconfigured script, or even a compromised instance - PutObject alone can only overwrite or add files, not permanently remove them. If the S3 bucket has versioning enabled, an accidental or malicious overwrite can still be rolled back to a previous version, so the damage is recoverable.
DeleteObject is fundamentally more dangerous because deletion is permanent (or at minimum much harder to recover from, even with versioning, since a delete can also remove version history depending on configuration). Removing DeleteObject from the policy means that even in a worst-case scenario - a bug, a mistake, or a compromised role - the instance simply cannot cause that level of permanent, hard-to-reverse damage. This is exactly what least privilege is meant to achieve: scoping permissions not just to what's needed for the job, but specifically excluding the most destructive actions unless they're genuinely required.