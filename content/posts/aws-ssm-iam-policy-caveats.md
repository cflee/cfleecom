---
title: "Default AWS Systems Manager IAM policy may grant unexpected S3 permissions"
date: 2018-10-28
draft: false
tags:
- aws
---

If you use S3 buckets and the AWS Systems Manager agent with the suggested AWS-managed SSM IAM policy for EC2 instances, you should take a careful look at the effective S3 permissions on your SSM-managed instances.
Depending on how you're managing your S3 bucket/object permissions, your instances may have more access than expected.

I've been testing out AWS Systems Manager (SSM), ever since the new Session Manager features got announced a few weeks ago.
There's an Amazon SSM Agent (a Go project that's [open-source on GitHub](https://github.com/aws/amazon-ssm-agent/)) that needs to be running on EC2 or on-prem instances to let the SSM service manage them.

Part of the AWS Systems Manager setup docs tell you to use an IAM instance profile to link an IAM role to the EC2 instances.
That's certainly best practice, so you don't have to inject long-lived AWS access key credentials into the instances!
They even provide an AWS-managed policy for this, so you can automatically take advantage of new features that are rolled out, like how the Session Manager rollout added new `ssmmessages` endpoint and actions for communication with the agents.

## Problem
The devil is in the (policy) details. The [Create an Instance Profile for Systems Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-configuring-access-role.html) page suggests that you use the `AmazonEC2RoleforSSM` AWS-managed policy.
You should check out the current policy in full via [IAM console](https://console.aws.amazon.com/iam/home?region=us-east-1#/policies/arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforSSM$serviceLevelSummary?section=permissions) (AWS console link) or the IAM API (probably GetPolicy and then GetPolicyVersion on `arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforSSM`), but here is the one statement with S3 actions:

```json
{
    "Effect": "Allow",
    "Action": [
        "s3:GetBucketLocation",
        "s3:PutObject",
        "s3:GetObject",
        "s3:GetEncryptionConfiguration",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts",
        "s3:ListBucket",
        "s3:ListBucketMultipartUploads"
    ],
    "Resource": "*"
}
```

This makes sense as a generic option.
The SSM Agent that's running on your instance is expected to upload stuff directly to S3 buckets of your choice – whether as output of Run Command or session logs from Session Manager – and they can't predict what your bucket will be named, so it they need a wildcard resource for `s3:PutObject` and related actions in this policy that's made available to all AWS customers, so that things 'just work'.

IAM policy evaluation logic is pretty complex and situation-specific, but the general intuition is that it's deny by default unless there's an explicit allow, and an explicit deny in any policy overrides any explicit allows.

**However, this statement provides an explicit allow on `s3:GetObject` and `s3:PutObject` (and other listed actions) to all S3 buckets/objects that the instance can access.**
This is probably going to be all buckets in your account, at the very least.
Therefore, the instances with this role/profile can read and (over)write objects in all buckets that don't have any explicit deny statements in any other policies.

This might be an unexpected result, given the intuitive "deny by default" understanding, especially if you've been relying on explicit allow statements in inline or managed policy attached to users/groups/roles, instead of using explicit deny statements in bucket policy.

## Non-goals
We should pause and figure out what we're _not_ trying to address here:

Any SSM-managed instance, and any user/software on any instance that can access the instance metadata service to obtain credentials, can write or overwrite any arbitrary object in the S3 buckets/key-prefixes that you use to store SSM Agent run output or logs.

Integrity is probably more of a concern for Session Manager session logs, but they aren't going to be absolutely tamperproof because the logs are being created and uploaded from the instance anyway.
The risk of overwriting should be mitigated by bucket object versioning, entirely missing logs with shipped Linux audit logs, etc.
(If you're really paranoid and want to do off-host recording you'll probably want to use something like [Gravitational Teleport in recording proxy mode](https://gravitational.com/teleport/docs/architecture/#audit-log) instead, but there are some different security trade-offs there.)

## Solutions?
I don't think I know all the possible ways of dealing with this, but here are some ideas.

One obvious option would be to keep using their wildcard allow, and add a deny statement in bucket policy that applies to all principals except certain whitelisted ones.

This doesn't seem very fun.
A [2015 AWS Security Blog post](https://aws.amazon.com/blogs/security/how-to-create-a-policy-that-whitelists-access-to-sensitive-amazon-s3-buckets/) covered the `NotPrincipal` element that can be used, but it's a little annoying for assumed roles because [you're supposed to specify not just the role-name but also the specific role-session-name/assumed-role user](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notprincipal.html) which in this case is the EC2 instance ID, so the policy would have to exhaustively list all the role-instance combinations.
A [2016 AWS Security Blog post](https://aws.amazon.com/blogs/security/how-to-restrict-amazon-s3-bucket-access-to-a-specific-iam-role/) covers this problem and suggests a `aws:userId StringNotLike Condition`, where you _can_ match on the IAM role IDs with a wildcard for the role-session-name.

Apart from the downsides of having to maintain a central whitelist of roles on the bucket policy, IAM policy cannot specify IAM groups as principal, so the S3 bucket policy won't be able to grant rights to a group.
You'll need to either specify all the users alongside the roles, or specify some role and arrange for the users to assume that role to perform operations on the bucket.

A second option would be to make your own policy document based off the AWS-managed policy.
This way, you can limit the especially sensitive `s3:GetObject` action to the AWS-managed S3 buckets required for SSM Agent operations, and the `s3:PutObject` actions to your designated log-receiving S3 bucket(s).

This means that you'll need to keep up with any future SSM enhancements and adjust the policy accordingly.
But to even start, _how do you even determine what exactly are the required permissions for the SSM features you want to use_, if you don't want to blindly copy from the given policy?

The SSM docs currently don't document the permissions thoroughly enough to fully match up with all the actions in the policy.
There _is_ a list of [Minimum S3 Bucket Permissions for SSM Agent](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent-minimum-s3-permissions.html) which looks like it should be enough for us to restrict our new policy document for S3 actions.
However, the same can't be said for EC2/SSM/etc actions, other than [Session Manager Permissions](https://docs.aws.amazon.com/systems-manager/latest/userguide/getting-started-add-permissions-to-existing-profile.html) and an example policy for [Session Manager and Amazon S3 and CloudWatch Logging](https://docs.aws.amazon.com/systems-manager/latest/userguide/getting-started-create-iam-instance-profile.html#create-iam-instance-profile-ssn-logging).

## Conclusion
You should check very carefully if your SSM-managed instances have more permissions to S3 buckets than you thought they had, if you set up instances for SSM Agent using the `AmazonEC2RoleforSSM` AWS-managed policy.

These are almost definitely not the only two ways to deal with this issue, and which is the most appropriate approach will vary with how you're using IAM in your AWS accounts.
I think you might be able to use a permission boundary on the role to limit the AWS-managed policy, for one, but I haven't tested that.
YMMV!

While finishing up this post, I realised that someone has opened an PR [awsdocs/aws-systems-manager-user-guide#41](https://github.com/awsdocs/aws-systems-manager-user-guide/pull/41) just three days ago to propose adding a warning note to the SSM docs on the wildcard access to S3 buckets.
An employee responded and indicated that they would open an internal ticket to get the doc team to look at it in more detail.
Let's see if they incorporate this or similar content into their docs in future.

There's also an [AWS Developer Forums post from mid-Sep](https://forums.aws.amazon.com/thread.jspa?messageID=868804&#868804) asking about this issue, but the employee that replied essentially said that everything is needed for some SSM use case or another, and they (support?) might be able to help craft a scoped down policy.
