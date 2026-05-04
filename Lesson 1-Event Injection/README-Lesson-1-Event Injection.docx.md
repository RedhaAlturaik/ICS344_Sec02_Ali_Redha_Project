# DVSA Lesson 1 — Event Injection

## ICS-344 Information Security Project

This repository documents **Lesson 1: Event Injection** from the Damn Vulnerable Serverless Application (DVSA) project for the ICS-344 Information Security course at King Fahd University of Petroleum and Minerals (KFUPM).

The purpose of this lesson is to demonstrate how unsafe event handling and insecure deserialization in a serverless application can lead to **Remote Code Execution (RCE)** inside an AWS Lambda function.

---

## Project Information

| Item | Details |
|---|---|
| Course | ICS-344 — Information Security |
| Section | 02 |
| Institution | King Fahd University of Petroleum and Minerals (KFUPM) |
| Lesson | Lesson 1 — Event Injection |
| Application | OWASP Damn Vulnerable Serverless Application (DVSA) |
| AWS Region | `us-east-1` — N. Virginia |
| Main Vulnerable Component | `DVSA-ORDER-MANAGER` Lambda Function |
| Main AWS Services | API Gateway, Lambda, DynamoDB, CloudWatch, S3 |
| Report Date | 19 April 2026 |

---

## Team Members

| Name | Student ID |
|---|---|
| Ali Kamal Aloryd | 202322750 |
| Redha Abdulaziz Alturaik | 20232301 |

---

## Vulnerability Summary

The DVSA application contains an **Event Injection** vulnerability in the order-processing backend. The vulnerable Lambda function processes attacker-controlled request data using the `node-serialize` library.

Because `node-serialize` supports serialized JavaScript functions, a malicious payload can be injected into the API request. When the backend deserializes the request body, the injected function may execute inside the Lambda runtime.

This results in **Remote Code Execution**, where attacker-controlled JavaScript code runs in the backend serverless environment.

---

## Root Cause

The vulnerability occurs because the backend Lambda function uses unsafe deserialization on untrusted user input:

```javascript
const serialize = require('node-serialize');
var req = serialize.unserialize(event.body);
var headers = serialize.unserialize(event.headers);
```

The issue is that `serialize.unserialize()` may execute specially crafted serialized functions, such as payloads using the `_$$ND_FUNC$$_` marker.

The application should treat user input as data only. Instead, the vulnerable implementation allows input to be interpreted as executable code.

---

## Intended Application Workflow

Under normal behavior, the application should follow this workflow:

1. A user submits an order from the DVSA frontend hosted on Amazon S3.
2. The request is sent to Amazon API Gateway.
3. API Gateway forwards the request to the `DVSA-ORDER-MANAGER` Lambda function.
4. The Lambda function parses the request body safely.
5. The backend validates expected fields such as `action` and `cart-id`.
6. The backend verifies authorization before processing the request.
7. The function interacts with DynamoDB to store or retrieve order data.
8. A safe response is returned to the user.

The expected security rule is:

> User-controlled input must never be executed as code. It must be parsed as JSON, validated, and processed only as data.

---

## Environment and Tools Used

| Tool / Service | Purpose |
|---|---|
| AWS Console | Managing and observing DVSA resources |
| API Gateway | Locating the `/order` endpoint |
| AWS Lambda | Vulnerable backend execution environment |
| CloudWatch Logs | Verifying whether injected code executed |
| Kali Linux Terminal | Sending crafted HTTP requests |
| `curl` | Testing the API endpoint |

---

## Reproduction Steps

> **Important:** These steps are only for the controlled DVSA lab environment. Do not use them against any system that you do not own or have permission to test.

### Step 1 — Get the API Endpoint

1. Log in to the AWS Console.
2. Open **API Gateway**.
3. Select the DVSA API.
4. Open **Stages**.
5. Select the active stage.
6. Copy the **Invoke URL**.
7. Append `/order` to the end of the URL.

Example format:

```text
https://<api-id>.execute-api.us-east-1.amazonaws.com/<stage>/order
```

---

### Step 2 — Send the Malicious Payload

Open a Kali Linux terminal and run the following command after replacing the API URL:

```bash
curl -X POST "<YOUR_API_GATEWAY_ORDER_URL>" \
-H "Content-Type: application/json" \
-H "Authorization: Bearer dummy_token" \
-d '{"action":"_$$ND_FUNC$$_function(){console.error(\"PWNED\")}()","cart-id":""}'
```

The API may return:

```json
{"message":"Internal server error"}
```

This response does **not** mean the exploit failed. In this lesson, the malicious code may execute before the application crashes.

---

### Step 3 — Verify Execution in CloudWatch

1. Open **CloudWatch** in the AWS Console.
2. Go to **Logs**.
3. Open the log group:

```text
/aws/lambda/DVSA-ORDER-MANAGER
```

4. Open the latest log stream.
5. Search for the injected output:

```text
PWNED
```

If the `PWNED` log appears, the injected code executed successfully inside the Lambda function.

---

## Evidence Summary

The exploit was confirmed through CloudWatch logs. Although API Gateway returned an internal server error, the backend log showed that the injected payload executed before the Lambda function crashed.

Expected evidence includes:

| Evidence | Description |
|---|---|
| Invoke URL Screenshot | Shows the API Gateway endpoint used for testing |
| Terminal Screenshot | Shows the crafted `curl` request and API response |
| CloudWatch Screenshot | Shows the injected `PWNED` log output |

---

## Security Impact

This vulnerability is severe because it allows attacker-controlled code to run inside the Lambda environment.

Possible impacts include:

- Remote Code Execution inside AWS Lambda
- Access to temporary files under `/tmp`
- Exposure of environment variables
- Abuse of Lambda execution-role permissions
- Possible access to AWS services if the Lambda role is over-privileged
- Increased blast radius in the serverless environment

---

## Fix Strategy

The correct mitigation is to remove unsafe deserialization from the request path and replace it with safe JSON parsing and strict validation.

The backend should:

- Stop using `node-serialize` for user-controlled input
- Use `JSON.parse()` for request bodies
- Validate all expected request fields
- Allowlist valid `action` values
- Reject unexpected input
- Avoid executing user-controlled content
- Review and remove vulnerable dependencies

---

## Code Change

### Vulnerable Code

```javascript
const serialize = require('node-serialize');
var req = serialize.unserialize(event.body);
var headers = serialize.unserialize(event.headers);
```

### Secure Code

```javascript
const req = typeof event.body === "string" ? JSON.parse(event.body) : event.body;
const headers = event.headers || {};

const allowedActions = ["new", "get", "orders", "shipping", "billing"];

if (!allowedActions.includes(req.action)) {
  throw new Error("Invalid input");
}
```

This fix ensures that the request body is parsed as JSON data and that only expected actions are accepted.

---

## Verification After Fix

After applying the fix, the same malicious payload should be sent again.

Expected secure behavior:

- The injected function does not execute
- No `PWNED` message appears in CloudWatch logs
- The request is rejected or handled safely
- Normal valid order requests continue to work

This confirms that the application no longer treats attacker-controlled input as executable code.

---

## Structured Security Analysis

### Table A — Intended Rule and Evidence

| Vulnerability | Intended Rule(s) | Artifacts Used | Normal Behavior Evidence | Exploit Behavior Evidence |
|---|---|---|---|---|
| Event Injection | Input must not be executed as code; only validated JSON should be processed | API request, CloudWatch logs, Lambda execution | Request is processed without executing input | Injected payload executes and logs `PWNED` |

### Table B — Deviation and Fix

| Vulnerability | Why This Is a Deviation | Deviation Class | Fix Applied | Post-Fix Verification |
|---|---|---|---|---|
| Event Injection | The system executed attacker-controlled input instead of treating it as data | Intentional misuse / security-relevant abuse | Removed unsafe deserialization in `DVSA-ORDER-MANAGER` | No injected output appears in CloudWatch after the fix |

---

## Lessons Learned

This lesson shows the danger of unsafe deserialization in serverless applications. Even though serverless platforms manage infrastructure, vulnerable application code can still lead to severe compromise.

The main takeaway is that user input must always be treated as untrusted data. It should be safely parsed, validated, and never executed as code. Developers should also avoid vulnerable dependencies and apply secure design principles such as input validation, least privilege, and defense in depth.

---

## Safety and Usage Notice

DVSA is intentionally vulnerable and must only be deployed in a controlled, non-production AWS account for educational purposes.

This repository and its contents are intended only for the ICS-344 Information Security course project. The techniques shown here must not be used maliciously or against systems without explicit authorization.

---

## Repository Structure Suggestion

```text
.
├── README.md
├── report/
│   └── Lesson-1-Event-Injection.pdf
├── screenshots/
│   ├── figure-1-invoke-url.png
│   ├── figure-2-malicious-payload.png
│   └── figure-3-cloudwatch-log.png
└── fixes/
    └── order-manager-secure-snippet.js
```

---

## Status

| Item | Status |
|---|---|
| Vulnerability demonstrated | Completed |
| Evidence collected | Completed |
| Root cause explained | Completed |
| Fix strategy written | Completed |
| Secure code snippet included | Completed |
| Post-fix verification described | Completed |
