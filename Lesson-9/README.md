# Lesson 9: Vulnerable Dependencies

## ICS-344: Information Security

**Course:** ICS-344 - Information Security  
**Section:** 02  
**Students:**  
- Ali Kamal Aloryd - 202322750  
- Redha Abdulaziz Alturaik - 20232301  

**DVSA Website:**  
`http://dvsa-website-test-546079881937-us-east-1.s3-website.us-east-1.amazonaws.com/`

**AWS Region:**  
`us-east-1 (N. Virginia)`

**Date:**  
19 April 2026

---

## Vulnerability Overview

This lesson demonstrates a **Vulnerable Dependencies** issue in the Damn Vulnerable Serverless Application (DVSA). The backend uses an outdated and insecure third-party Node.js package called `node-serialize`.

The vulnerable dependency is used inside the `DVSA-ORDER-MANAGER` Lambda function to deserialize incoming request data. Because `node-serialize` can deserialize and execute JavaScript functions, attacker-controlled input can be executed inside the Lambda runtime.

This creates a serious **Remote Code Execution (RCE)** risk.

---

## Affected Component

| Component | Details |
|---|---|
| Application | OWASP DVSA |
| Vulnerability | Vulnerable Dependencies |
| Lesson | Lesson #9 |
| AWS Service | AWS Lambda |
| Lambda Function | `DVSA-ORDER-MANAGER` |
| Vulnerable Package | `node-serialize` |
| Impact | Remote Code Execution through unsafe deserialization |

---

## Root Cause

The vulnerability exists because the application relies on an unsafe third-party dependency:

```js
const serialize = require('node-serialize');
```

The vulnerable code then uses:

```js
var req = serialize.unserialize(event.body);
```

Unlike safe JSON parsing, `node-serialize` can interpret specially crafted serialized functions. When the attacker sends a payload containing the `_$$ND_FUNC$$_` marker, the library can execute the function during deserialization.

The root cause is a combination of:

- unsafe deserialization
- use of an insecure third-party dependency
- lack of dependency security review
- lack of strict input validation
- treating user-controlled input as executable content

---

## Intended Secure Workflow

In a secure serverless application:

1. API Gateway receives a request from the frontend or client.
2. API Gateway forwards the request to the Lambda function.
3. The Lambda function parses the request body safely using `JSON.parse()`.
4. Input fields are validated against an allowlist.
5. User input is treated only as data.
6. No dependency should execute user-controlled input.
7. Vulnerable or outdated packages should be removed or replaced.

The expected behavior is that request data should never be interpreted as code.

---

## Environment and Tools

The testing environment included:

- AWS Console
- AWS Lambda Console
- Amazon CloudWatch Logs
- API Gateway endpoint
- Kali Linux terminal
- `curl`

---

## Reproduction Steps

> Replace `YOUR_API/order` with the real API Gateway `/order` endpoint.

### Step 1: Identify the Vulnerable Dependency

1. Open AWS Console.
2. Go to **Lambda**.
3. Open the function:

```text
DVSA-ORDER-MANAGER
```

4. Open the source code.
5. Locate the following vulnerable dependency:

```js
const serialize = require('node-serialize');
```

---

### Step 2: Locate Unsafe Deserialization

Inside the Lambda code, locate:

```js
var req = serialize.unserialize(event.body);
```

This confirms that attacker-controlled request data is being passed into an unsafe deserialization function.

---

### Step 3: Send the Malicious Payload

Run the following request:

```bash
curl -X POST "YOUR_API/order" \
-H "Content-Type: application/json" \
-d '{"action":"_$$ND_FUNC$$_function(){console.error(\"PWNED\")}()","cart-id":""}'
```

---

### Step 4: Verify Execution in CloudWatch

1. Open AWS Console.
2. Go to **CloudWatch**.
3. Open **Log Groups**.
4. Open:

```text
/aws/lambda/DVSA-ORDER-MANAGER
```

5. Open the latest log stream.
6. Search for:

```text
PWNED
```

If the message appears in the Lambda logs, the dependency allowed attacker-controlled code to execute.

---

## Evidence and Proof

The vulnerability was confirmed using three main pieces of evidence:

### Evidence 1: Vulnerable Dependency in Lambda Code

The Lambda function includes:

```js
const serialize = require('node-serialize');
```

This confirms that the application depends on the insecure `node-serialize` package.

### Evidence 2: Unsafe Deserialization

The Lambda function uses:

```js
var req = serialize.unserialize(event.body);
```

This shows that attacker-controlled request data is passed directly into the unsafe dependency.

### Evidence 3: CloudWatch Execution Proof

After sending the crafted payload, CloudWatch logs showed the injected output:

```text
PWNED
```

Although the API may return:

```json
{"message":"Internal server error"}
```

the exploit is still successful because the malicious code executes before the application crashes.

---

## Security Impact

This vulnerability is critical because it allows attacker-controlled input to execute inside the Lambda function.

Possible impacts include:

- remote code execution
- access to Lambda environment variables
- abuse of the Lambda execution role
- reading or modifying backend data depending on IAM permissions
- increased blast radius if the function has excessive permissions
- service disruption or backend crashes

In serverless applications, vulnerable dependencies are especially dangerous because Lambda functions often have access to cloud resources through IAM roles.

---

## Fix Strategy

The correct fix is to remove the vulnerable dependency and replace unsafe deserialization with safe JSON parsing.

Recommended mitigations:

- remove `node-serialize`
- use `JSON.parse()` for request bodies
- validate all request fields using an allowlist
- reject unexpected actions or malformed data
- regularly update dependencies
- scan dependencies during development and deployment
- avoid libraries that deserialize executable functions

---

## Code / Config Changes

### Before: Vulnerable Code

```js
const serialize = require('node-serialize');

var req = serialize.unserialize(event.body);
```

### After: Secure Code

```js
const req = typeof event.body === "string"
  ? JSON.parse(event.body)
  : event.body;
```

### Additional Recommended Validation

```js
const allowedActions = ["new", "get", "orders", "shipping", "billing", "update"];

if (!allowedActions.includes(req.action)) {
  return callback(null, {
    statusCode: 400,
    headers: { "Access-Control-Allow-Origin": "*" },
    body: JSON.stringify({
      status: "err",
      msg: "Invalid action"
    })
  });
}
```

This ensures the request is parsed safely and only expected actions are accepted.

---

## Verification After Fix

After removing `node-serialize`, the same malicious payload was tested again:

```bash
curl -X POST "YOUR_API/order" \
-H "Content-Type: application/json" \
-d '{"action":"_$$ND_FUNC$$_function(){console.error(\"PWNED\")}()","cart-id":""}'
```

Expected result after the fix:

- the payload is treated as a normal string
- no JavaScript function is executed
- no `PWNED` message appears in CloudWatch logs
- the backend rejects the request as invalid input

This confirms that removing the vulnerable dependency eliminates the execution path.

---

## Structured Operation and Security Analysis

### Table A: Intended Logic vs Exploit Behavior

| Vulnerability | Intended Rule(s) | Artifacts Used to Infer Rule | Normal Behavior Evidence | Exploit Behavior Evidence |
|---|---|---|---|---|
| Vulnerable Dependencies | Dependencies must not execute user input | Lambda code, dependency usage, CloudWatch logs | Safe parsing should treat input as data | `node-serialize` executed attacker-controlled input and logged `PWNED` |

### Table B: Deviation and Fix

| Vulnerability | Why This Is a Deviation | Deviation Class | Fix Applied | Post-Fix Verification |
|---|---|---|---|---|
| Vulnerable Dependencies | An insecure dependency executed user-controlled input instead of treating it as data | Misconfiguration / security-relevant misuse | Removed `node-serialize` from `DVSA-ORDER-MANAGER` and replaced unsafe deserialization with `JSON.parse()` | Same payload no longer executes and no `PWNED` appears in CloudWatch |

---

## Lessons Learned

This lesson shows that application security depends not only on custom code, but also on third-party packages. A vulnerable dependency can introduce severe security flaws even when the main application logic appears simple.

In DVSA, the `node-serialize` package created a direct path from user input to backend code execution. In serverless environments, this is especially dangerous because compromised Lambda code may inherit cloud permissions through its IAM execution role.

The main takeaway is:

> Never use unsafe deserialization libraries on attacker-controlled input.

Secure serverless applications should use safe parsing, strict validation, dependency scanning, and continuous package maintenance.
