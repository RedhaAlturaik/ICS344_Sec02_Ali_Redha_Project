# Lesson 7: Over-Privileged Function

## ICS-344 Information Security Project

**Course:** ICS-344 - Information Security  
**Section:** 02  
**Students:**  
- Ali Kamal Aloryd - 202322750  
- Redha Abdulaziz Alturaik - 20232301  

**DVSA Website:** `http://dvsa-website-test-546079881937-us-east-1.s3-website.us-east-1.amazonaws.com/`  
**AWS Region:** `us-east-1 (N. Virginia)`  
**Date:** 19 April 2026  
**Lesson:** Lesson 7 - Over-Privileged Function  

---

## 1. Goal and Vulnerability Summary

This lesson evaluates whether the Lambda function responsible for sending receipt emails is operating with excessive IAM permissions.

An **Over-Privileged Function** occurs when a Lambda execution role is granted permissions beyond what the function actually needs. In serverless systems, this increases the blast radius because any compromised function inherits the permissions of its execution role.

The function analyzed in this lesson was:

```text
DVSA-SEND-RECEIPT-EMAIL
```

The main objective was to inspect the function's execution role, test its permissions using IAM Policy Simulator, and determine whether the function had unnecessary access to AWS services such as Amazon S3 or DynamoDB.

---

## 2. Why This Works / Root Cause

Over-privileged vulnerabilities happen when IAM roles use overly broad permissions such as:

```text
*
```

or when a Lambda function is granted access to services and resources unrelated to its intended task.

The security issue is based on violating the **Principle of Least Privilege**, which states that each function should only have the minimum permissions required to complete its job.

In this lesson, the role was analyzed to determine whether it allowed unnecessary access to S3 objects, DynamoDB tables, or other AWS resources.

---

## 3. Environment and Setup

The test was performed on the DVSA deployment in AWS.

### AWS Services Involved

- AWS Lambda
- IAM Roles
- IAM Policy Simulator
- Amazon S3
- Amazon DynamoDB
- CloudTrail
- CloudWatch
- API Gateway

### Lambda Function Under Review

```text
DVSA-SEND-RECEIPT-EMAIL
```

### IAM Role Under Review

```text
serverlessrepo-OWASP-DVSA-SendReceiptFunctionRole
```

---

## 4. Reproduction Steps

### Step 1: Access the Lambda Function

1. Log in to the AWS Management Console.
2. Open the **Lambda** service.
3. Search for:

```text
DVSA-SEND-RECEIPT-EMAIL
```

4. Open the function overview page.

### Step 2: Inspect the Execution Role

1. Go to the **Configuration** tab.
2. Select **Permissions** from the left panel.
3. Locate the execution role attached to the function.
4. Click the role name to open it in IAM.

### Step 3: Review IAM Policies

1. In the IAM role page, open the **Permissions** tab.
2. Review the attached policies.
3. Look for broad or dangerous permissions such as:

```text
Action: "*"
Resource: "*"
```

or wide service access such as:

```text
s3:*
dynamodb:*
```

### Step 4: Open IAM Policy Simulator

1. From the IAM role page, open the simulation option.
2. Confirm that the selected role is:

```text
serverlessrepo-OWASP-DVSA-SendReceiptFunctionRole
```

### Step 5: Test Amazon S3 Access

In the simulator:

1. Select **Amazon S3** as the service.
2. Add the following actions:

```text
s3:GetObject
s3:PutObject
```

3. Run the simulation.
4. Observe whether the actions are **Allowed** or **Denied**.

### Step 6: Test DynamoDB Access

In the simulator:

1. Select **Amazon DynamoDB** as the service.
2. Add the following actions:

```text
dynamodb:Scan
dynamodb:GetItem
dynamodb:PutItem
dynamodb:DeleteItem
```

3. Run the simulation.
4. Observe whether the actions are **Allowed** or **Denied**.

### Step 7: Enable CloudTrail Logging

1. Go to **CloudTrail**.
2. Open **Trails**.
3. Create a trail named:

```text
dvsa-policygen-trail
```

4. Confirm that the trail status shows **Logging**.

### Step 8: Trigger the Lambda Function

1. Open the DVSA web application.
2. Place a new order.
3. This should trigger the receipt email workflow and invoke the `DVSA-SEND-RECEIPT-EMAIL` function.

### Step 9: Attempt Policy Generation from CloudTrail

1. Go back to **IAM > Roles**.
2. Open the `SendReceiptFunctionRole`.
3. Locate **Generate policy based on CloudTrail events**.
4. Attempt to generate a policy.
5. Observe whether AWS generates a policy or returns no activity.

---

## 5. Evidence and Proof

The investigation included the following evidence:

- Lambda function page for `DVSA-SEND-RECEIPT-EMAIL`
- Permissions page showing the execution role
- IAM Policy Simulator page
- Amazon S3 simulation results
- DynamoDB simulation results
- CloudTrail trail creation
- DVSA order placement to trigger the workflow

### Observed Result

The IAM Policy Simulator showed that unauthorized S3 and DynamoDB actions were **Denied**.

This means that, in this specific deployment, the expected over-privileged behavior was not observed. The role appeared to enforce restricted access rather than broad access.

---

## 6. Fix Strategy / Probable Mitigation

Even though the exploit was not successful in this deployment, the correct mitigation for over-privileged Lambda roles is still to enforce least privilege.

A secure Lambda execution role should:

- Remove unnecessary full-access managed policies
- Avoid wildcard actions such as `s3:*` or `dynamodb:*`
- Avoid wildcard resources such as `Resource: "*"`
- Restrict S3 access to only the required bucket and object paths
- Restrict DynamoDB access to only the required table and actions
- Use CloudTrail and IAM Access Analyzer to review actual permissions used

---

## 7. Code / Configuration Changes

No direct fix was required because the simulator did not show exploitable over-privileged behavior.

However, the following examples show how an over-privileged policy should be corrected.

### Vulnerable Example

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

This grants broad access to all S3 actions and all S3 resources.

### Secure Example

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject"
  ],
  "Resource": "arn:aws:s3:::dvsa-receipts-bucket/*"
}
```

### DynamoDB Least-Privilege Example

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:GetItem"
  ],
  "Resource": "arn:aws:dynamodb:us-east-1:ACCOUNT_ID:table/DVSA-ORDERS"
}
```

This limits DynamoDB access to only the required action and table.

---

## 8. Verification After Fix

Verification was performed using the IAM Policy Simulator.

The tested unauthorized actions on S3 and DynamoDB remained denied, confirming that the function role did not allow unnecessary access in this deployment.

### Verification Result

```text
Unauthorized S3 actions: Denied
Unauthorized DynamoDB actions: Denied
```

This confirms that the role follows a restricted permission model for the tested actions.

---

## 9. Structured Operation and Security Analysis

### Table A: Intended vs Exploit Behavior

| Vulnerability | Intended Rule(s) | Artifacts Used to Infer Rule | Normal Behavior Evidence | Exploit Behavior Evidence |
|---|---|---|---|---|
| Over-Privileged Function | Lambda should access only the AWS resources required for its task | IAM policies, IAM Policy Simulator, CloudTrail, Lambda configuration | Simulator showed limited access | No exploit possible; unauthorized actions were denied |

### Table B: Deviation and Fix Analysis

| Vulnerability | Why This Is a Deviation | Deviation Class | Fix Applied Where | Post-Fix Verification |
|---|---|---|---|---|
| Over-Privileged Function | No deviation was observed in this deployment | N/A | Not required; least-privilege configuration recommended | IAM Policy Simulator showed denied results |

---

## 10. Takeaway / Lessons Learned

This lesson demonstrates the importance of validating IAM permissions in serverless applications.

Even when a vulnerability is expected in a lab environment, the actual deployment may behave differently because of AWS updates, changed policies, or deployment-specific configurations.

The main lesson is that IAM permissions should never be assumed. They must be tested using tools such as:

- IAM Policy Simulator
- CloudTrail
- IAM Access Analyzer
- CloudWatch Logs

The secure design principle is:

> Lambda functions should only have the minimum permissions required to perform their intended task.

Following least privilege reduces the blast radius if a Lambda function is compromised.

---

## Repository Notes

This README is part of the ICS-344 DVSA vulnerability documentation. It is intended for educational use only in a controlled, non-production AWS environment.

