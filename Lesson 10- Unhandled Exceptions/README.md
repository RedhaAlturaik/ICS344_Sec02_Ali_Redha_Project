# Lesson #10: Unhandled Exceptions

## ICS-344 Information Security — DVSA Project

This repository documents **Lesson #10: Unhandled Exceptions** from the Damn Vulnerable Serverless Application (DVSA) project for ICS-344 Information Security.

The lesson demonstrates how malformed or incomplete API requests can cause backend Lambda functions to crash and expose internal implementation details such as stack traces, file paths, line numbers, and exception messages.

---

## Vulnerability Overview

| Item | Description |
|---|---|
| Vulnerability | Unhandled Exceptions |
| Lesson | #10 |
| Affected Component | `DVSA-ORDER-GET` Lambda function |
| Entry Point | API Gateway `/order` endpoint |
| Main Impact | Internal error disclosure and backend crash behavior |
| AWS Region | `us-east-1` |
| Main AWS Services | API Gateway, Lambda, DynamoDB, CloudWatch |

---

## Goal

The goal of this lesson is to show that the DVSA backend does not safely handle malformed or incomplete requests.

Instead of validating input and returning a generic error message, the backend may expose sensitive implementation details such as:

- Internal file paths
- Exact source-code line numbers
- Python exception types
- Stack traces
- Backend logic details

This information can help an attacker understand the backend structure and plan further exploitation.

---

## Root Cause

The vulnerability occurs because the Lambda function directly accesses request fields without proper validation or exception handling.

Example vulnerable behavior:

```python
orderId = event["orderId"]
userId = event["user"]
```

If the required field is missing, Python raises a `KeyError`. Because the function does not safely catch and handle the error, internal exception details may be returned to the client.

The main causes are:

- Missing input validation
- Direct access to user-controlled fields
- Lack of `try/except` error handling
- Internal errors exposed to users
- Inconsistent error responses

---

## Environment and Setup

The test was performed in the DVSA AWS deployment.

### AWS Services Used

- **Amazon API Gateway** — public API entry point
- **AWS Lambda** — vulnerable backend function
- **Amazon DynamoDB** — backend data storage
- **Amazon CloudWatch** — log verification

### Target Lambda Function

```text
DVSA-ORDER-GET
```

### Tools Used

- `curl`
- Browser DevTools
- AWS Console
- CloudWatch Logs

---

## Safe Variables

Before testing, define the API endpoint and token using placeholders.

```bash
export API="https://<API_ID>.execute-api.us-east-1.amazonaws.com/dvsa/order"
export TOKEN="<YOUR_VALID_JWT_TOKEN>"
```

> Do not commit real JWTs, AWS credentials, session tokens, or private identifiers to GitHub.

---

## Reproduction Steps

### Step 1 — Missing Required Field

Send a request without the required `order-id` field.

```bash
curl -X POST "$API" -H "Content-Type: application/json" -H "Authorization: $TOKEN" -d '{"action":"get"}'
```

### Step 2 — Malformed JSON

Send malformed JSON to trigger parsing errors.

```bash
curl -X POST "$API" -H "Content-Type: application/json" -H "Authorization: $TOKEN" -d '{"action":"orders",'
```

### Step 3 — Invalid Data Type

Send an unexpected data type for the `action` field.

```bash
curl -X POST "$API" -H "Content-Type: application/json" -H "Authorization: $TOKEN" -d '{"action":12345}'
```

---

## Evidence and Proof

The vulnerability was confirmed when malformed or incomplete requests caused backend errors that exposed internal implementation details.

Observed evidence included:

- Internal file path such as `/var/task/get_order.py`
- Exact source-code line number
- Exception type
- Code-level error details
- JSON parsing error exposed directly to the client

### Evidence Summary

| Evidence | Description |
|---|---|
| Figure 1 | Full vulnerability output showing internal file path, line number, code snippet, and exception type |
| Figure 2 | JSON parsing error exposed directly |
| Figure 3 | Invalid input response showing inconsistent backend handling |

These results show that the backend crashes for invalid input and exposes details that should remain internal.

---

## Security Impact

Unhandled exceptions can create serious security risks because they reveal information that helps attackers understand the system.

Potential impact includes:

- Backend implementation disclosure
- Easier vulnerability discovery
- Exposure of internal file structure
- Information useful for chaining attacks
- Reduced reliability and availability

Even if the vulnerability does not directly modify data, it weakens the security posture of the application by exposing backend internals.

---

## Fix Strategy

The vulnerability should be mitigated by validating input before processing and returning safe error messages to users.

Recommended fixes:

- Validate all required fields before use
- Use safe access methods such as `event.get()`
- Wrap risky logic inside `try/except`
- Return generic client-safe error messages
- Log detailed errors only in CloudWatch
- Avoid exposing stack traces, file paths, or source-code details to users

---

## Code / Configuration Changes

### Before — Vulnerable Code

```python
orderId = event["orderId"]
userId = event["user"]
```

This code assumes that all required fields exist. If a field is missing, the Lambda function crashes.

---

### After — Fixed Code

```python
if "orderId" not in event:
    return {
        "status": "err",
        "msg": "Missing orderId"
    }

if "user" not in event:
    return {
        "status": "err",
        "msg": "Missing user"
    }
```

---

### Added Global Error Handling

```python
try:
    # main function logic here
    pass

except Exception as e:
    print("Internal error:", str(e))
    return {
        "status": "err",
        "msg": "Internal server error"
    }
```

The detailed error is logged internally, while the user receives only a safe generic response.

---

## Verification After Fix

After applying the fix, the same malformed requests were repeated.

Expected secure behavior:

- No backend crash
- No stack trace exposed
- No internal file path leaked
- No line number or source-code details returned
- User receives a safe error message only

Example safe response:

```json
{
  "status": "err",
  "msg": "Internal server error"
}
```

or:

```json
{
  "status": "err",
  "msg": "Missing orderId"
}
```

---

## Structured Operation and Security Analysis

### Table A — Intended vs Exploit Behavior

| Vulnerability | Intended Rule(s) | Artifacts Used to Infer Rule | Normal Behavior Evidence | Exploit Behavior Evidence |
|---|---|---|---|---|
| Unhandled Exceptions | Application must validate input and must not expose internal errors | API responses, CloudWatch logs, Lambda code | Valid request returns correct order data | Missing input causes crash and stack trace exposure |

---

### Table B — Deviation and Fix

| Vulnerability | Why This Is a Deviation | Deviation Class | Fix Applied | Post-Fix Verification | Optional Latency |
|---|---|---|---|---|---|
| Unhandled Exceptions | The application exposes internal logic because of missing validation and error handling | Security Misconfiguration | Added input validation and `try/except` handling in `get_order.py` | Malformed input returns safe error message only | ~120ms / ~125ms |

---

## Takeaway / Lessons Learned

This lesson shows that secure error handling is an important part of serverless security.

Even simple missing-field errors can expose internal backend details if they are not handled properly. In a serverless environment, Lambda functions should validate all input, handle exceptions safely, and log detailed technical errors only internally.

Key lessons:

- Always validate user input
- Never trust client-provided data
- Never expose stack traces to users
- Return safe and generic error messages
- Keep detailed diagnostics inside CloudWatch only

---

## Repository Safety Note

This README intentionally uses placeholders for sensitive values.

Do not upload:

- Real JWT tokens
- AWS access keys
- AWS secret keys
- Session tokens
- Private account identifiers
- Screenshots showing secrets

Use redacted screenshots and safe placeholders when submitting or publishing the repository.
