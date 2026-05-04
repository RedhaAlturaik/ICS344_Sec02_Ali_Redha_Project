# ICS-344 DVSA Lesson 3 — Sensitive Information Disclosure

## Overview

This repository documents **Lesson #3: Sensitive Information Disclosure** from the ICS-344 DVSA course project. The lesson demonstrates how a valid authenticated user can access another user’s private order and receipt information by modifying the `order-id` parameter in an API request.

The vulnerability exists in the DVSA backend because the application verifies that a request is authenticated, but it does **not** properly verify that the requested order belongs to the authenticated user. As a result, sensitive information such as address, phone number, payment details, and internal user identifiers may be exposed.

> **Important:** This project is for educational use only in a controlled DVSA lab environment. DVSA is intentionally vulnerable and must not be deployed in production or used maliciously.

---

## Course Information

| Item | Details |
|---|---|
| Course | ICS-344 — Information Security |
| Section | 02 |
| Lesson | Lesson #3 |
| Vulnerability | Sensitive Information Disclosure |
| AWS Region | `us-east-1` / N. Virginia |
| DVSA Website | `http://dvsa-website-test-546079881937-us-east-1.s3-website.us-east-1.amazonaws.com/` |
| API Endpoint | `https://h3x14rchv6.execute-api.us-east-1.amazonaws.com/dvsa/order` |
| Date | 19 April 2026 |

---

## Vulnerability Summary

The vulnerability allows an authenticated user to access another user’s private order information by changing the `order-id` value in the API request.

### Affected Component

- **API Gateway**
- **AWS Lambda**
  - `DVSA-ORDER-GET`
- **DynamoDB**
- **CloudWatch Logs**

### Security Impact

If exploited, the vulnerability may disclose:

- Victim order details
- Address information
- Phone number
- Payment-related data
- Internal user identifiers
- Receipt-related information

This is a serious confidentiality issue because authentication is enforced, but authorization and ownership validation are missing.

---

## Root Cause

The backend trusts the supplied `order-id` without checking whether the authenticated user owns that order.

The application performs authentication by requiring a valid token, but it does not enforce authorization before returning sensitive order details.

In addition, the Lambda function contained an implementation bug in the `isAdmin` handling logic:

```python
is_admin = json.loads(event.get("isAdmin", "false")).lower()
```

Because `json.loads("false")` returns the Boolean value `False`, calling `.lower()` caused the function to crash:

```text
'bool' object has no attribute 'lower'
```

This bug initially blocked the exploit from completing cleanly. After correcting the type-handling issue, the missing authorization flaw became visible.

---

## Environment and Tools

The DVSA environment was deployed on AWS in the `us-east-1` region.

### AWS Services Used

- Amazon S3
- Amazon API Gateway
- AWS Lambda
- Amazon DynamoDB
- Amazon CloudWatch

### Tools Used

- Browser Developer Tools
- `curl`
- `jq`
- AWS Lambda Console
- CloudWatch Logs

### Test Users

| User | Role |
|---|---|
| User B | Attacker |
| User C | Victim |

Both users placed at least one order before testing.

---

## Reproduction Steps

### Step 1 — Confirm Authentication Exists

Send a request to the `/order` API endpoint without an authorization token:

```bash
curl -s "https://h3x14rchv6.execute-api.us-east-1.amazonaws.com/dvsa/order" \
-H "content-type: application/json" \
--data-raw '{"action":"get","order-id":"2d3f476f-b936-4901-b1dd-18946ff8ba25"}' | jq
```

Expected result:

```json
{
  "error": "Invalid token format"
}
```

This confirms that the endpoint requires authentication.

---

### Step 2 — Attempt Cross-User Access

Use a valid token from **User B** and request an order belonging to **User C**:

```bash
export TOKEN="<USER_B_TOKEN>"

BODY='{"action":"get","order-id":"2d3f476f-b936-4901-b1dd-18946ff8ba25"}'

curl -s "https://h3x14rchv6.execute-api.us-east-1.amazonaws.com/dvsa/order" \
-H "content-type: application/json" \
-H "authorization: $TOKEN" \
--data-raw "$BODY" | jq
```

Initial backend error:

```text
'bool' object has no attribute 'lower'
```

This showed that the request reached the backend but crashed because of an internal type-handling issue.

---

### Step 3 — Patch the Backend Type-Handling Bug

Inside the Lambda function `DVSA-ORDER-GET`, the vulnerable line was changed.

Before:

```python
is_admin = json.loads(event.get("isAdmin", "false")).lower()
```

After:

```python
is_admin = str(event.get("isAdmin", "false")).lower()
```

After applying this patch, the Lambda function was deployed again.

---

### Step 4 — Re-run the Exploit

Run the same request again using:

- User B’s valid token
- User C’s `order-id`

After the temporary execution patch, the API returned the victim’s full order details. This confirmed that the system was vulnerable because it allowed access to another user’s sensitive information without verifying ownership.

---

## Evidence and Proof

The vulnerability was confirmed through two observations:

1. A request without an authorization token was rejected with `Invalid token format`, proving that authentication exists.
2. A request using User B’s valid token with User C’s `order-id` returned the victim’s private order details.

This proves that the weakness is not missing authentication. The real weakness is missing authorization and ownership validation.

---

## Fix Strategy

The correct mitigation is to enforce ownership validation before returning order or receipt information.

The backend must verify that the authenticated user owns the requested order. Only administrators should be allowed to bypass this check.

### Required Security Rule

```text
A user can only access an order if the order belongs to that user, unless the user is an authorized administrator.
```

---

## Code / Configuration Changes

### Temporary Execution Patch

This patch fixed the backend crash so the vulnerability could be tested properly.

Before:

```python
is_admin = json.loads(event.get("isAdmin", "false")).lower()
```

After:

```python
is_admin = str(event.get("isAdmin", "false")).lower()
```

### Real Security Fix

Ownership validation should be added before returning sensitive order data:

```python
if order["userId"] != userId:
    return {
        "statusCode": 403,
        "body": "Unauthorized access"
    }
```

This prevents users from accessing orders that do not belong to them.

---

## Verification After Fix

After applying ownership validation, repeat the exploit using:

- User B’s valid token
- User C’s `order-id`

Expected result:

```json
{
  "statusCode": 403,
  "body": "Unauthorized access"
}
```

A legitimate request where the user accesses their own order should still work normally.

---

## Structured Operation and Security Analysis

### Table A — Intended Rule and Evidence

| Vulnerability | Intended Rule(s) | Artifacts Used to Infer Rule | Normal Behavior Evidence | Exploit Behavior Evidence |
|---|---|---|---|---|
| Sensitive Information Disclosure | Users must only access their own orders and receipts. Authentication alone is not sufficient; ownership validation must be enforced. | Browser workflow, API requests, JWT token, Lambda code, terminal output, Lambda logs | Request without token returned `Invalid token format`, proving authentication exists | Valid token from User B + User C `order-id` returned victim’s full order details |

### Table B — Deviation and Fix

| Vulnerability | Why This Is a Deviation | Deviation Class | Fix Applied | Post-Fix Verification |
|---|---|---|---|---|
| Sensitive Information Disclosure | The backend trusted `order-id` without verifying ownership, allowing attacker-controlled input to access another user’s sensitive data | Intentional misuse / security-relevant abuse | Lambda function `DVSA-ORDER-GET` — added ownership validation and fixed `isAdmin` type handling | Unauthorized cross-user requests should return `403`; legitimate same-user requests should still work |

---

## Lessons Learned

This lesson shows that authentication alone is not enough to protect sensitive data. A backend must verify both:

1. **Who the user is**
2. **Whether the user is allowed to access the requested resource**

In serverless applications, sensitive data can be exposed when API Gateway and Lambda pass user-controlled identifiers to backend storage without ownership checks.

The main takeaway is:

> Never trust user-controlled identifiers without validating ownership.

Every request for sensitive data must be checked against the authenticated identity, and backend errors should be handled safely to avoid leaking implementation details.

---

## Repository Contents

Suggested repository structure:

```text
.
├── README.md
├── screenshots/
│   ├── invalid-token-format.png
│   ├── cross-user-request.png
│   ├── victim-order-output.png
│   └── post-fix-403.png
├── commands/
│   └── lesson-3-curl-commands.txt
└── fixes/
    └── DVSA-ORDER-GET-fix.py
```

---

## Disclaimer

This work was completed as part of the ICS-344 Information Security course project using the intentionally vulnerable DVSA application. All testing was performed in a controlled educational environment. The techniques shown here must not be used against systems without explicit authorization.
