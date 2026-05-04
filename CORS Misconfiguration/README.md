# Extra Vulnerability #3: CORS Misconfiguration

## ICS-344 Information Security — DVSA Bonus Finding

This repository documents **Extra Vulnerability #3: CORS Misconfiguration** discovered in the Damn Vulnerable Serverless Application (DVSA) project for ICS-344 Information Security.

The issue occurs because the backend API returns a wildcard CORS header, allowing any origin to access API responses.

---

## Vulnerability Overview

| Item | Description |
|---|---|
| Vulnerability | CORS Misconfiguration |
| Type | Bonus / Extra Finding |
| Affected Component | `DVSA-ORDER-MANAGER` Lambda function |
| Entry Point | API Gateway `/order` endpoint |
| Main Impact | Untrusted origins can access API responses |
| AWS Region | `us-east-1` |
| Main AWS Services | API Gateway, Lambda |

---

## Goal

The goal of this finding is to demonstrate that the DVSA API allows requests from any origin because it returns:

```http
Access-Control-Allow-Origin: *
```

This means a malicious website could make browser-based requests to the API and read responses if the victim is authenticated or if a valid token is used.

The fix is to restrict CORS access to only the trusted DVSA frontend domain.

---

## Root Cause

The vulnerability exists because the Lambda response includes the following permissive CORS header:

```javascript
headers: {
  "Access-Control-Allow-Origin": "*"
}
```

Since DVSA uses **Lambda proxy integration**, the CORS headers are controlled by the Lambda response rather than only by API Gateway settings.

This means fixing API Gateway CORS alone may not be enough. The response headers must be corrected inside the Lambda function.

---

## Environment and Setup

### AWS Region

```text
us-east-1 (N. Virginia)
```

### Target API Endpoint

```bash
export API="https://<API_ID>.execute-api.us-east-1.amazonaws.com/dvsa/order"
```

### Target Lambda Function

```text
DVSA-ORDER-MANAGER
```

### Tools Used

- Kali Linux terminal
- `curl`
- AWS Lambda Console
- API response headers

### Safe Token Placeholder

```bash
export TOKEN="<YOUR_VALID_JWT_TOKEN>"
```

> Do not commit real JWTs, AWS credentials, or session tokens to GitHub.

---

## Reproduction Steps

### Step 1 — Send a Normal API Request

```bash
curl -i "$API" -H "authorization: $TOKEN" --data-raw '{"action":"orders"}'
```

The response should include normal API output and response headers.

---

### Step 2 — Send Request from a Malicious Origin

```bash
curl -i "$API" -H "Origin: http://evil.com" -H "authorization: $TOKEN" --data-raw '{"action":"orders"}'
```

---

### Step 3 — Observe the Vulnerable CORS Header

In the response headers, the API returns:

```http
Access-Control-Allow-Origin: *
```

This confirms that the API allows any origin, including `http://evil.com`.

---

## Evidence and Proof

The vulnerability was confirmed using response headers and Lambda code inspection.

| Evidence | Description |
|---|---|
| Figure 1 | API response showing `Access-Control-Allow-Origin: *` |
| Figure 2 | Lambda code showing wildcard origin |
| Figure 3 | Updated Lambda code with restricted origin |
| Figure 4 | API response after fix showing restricted trusted origin |

The evidence confirms that the API initially allowed all origins and was later secured by restricting the allowed origin.

---

## Security Impact

A wildcard CORS policy can expose sensitive API responses to untrusted websites.

Potential impact includes:

- Cross-origin access from malicious domains
- Browser-based data theft
- Increased risk if tokens are exposed or stored unsafely
- Weaker frontend/backend trust boundary
- Larger attack surface for authenticated API abuse

Although CORS is enforced by browsers and is not a replacement for authentication, a permissive policy can make attacks easier when combined with weak token handling, XSS, or exposed credentials.

---

## Fix Strategy

The fix is to restrict CORS to only the trusted DVSA frontend domain.

Because this application uses Lambda proxy integration, the fix should be applied inside the Lambda response headers.

---

## Code / Configuration Changes

### Before — Vulnerable Configuration

```javascript
headers: {
  "Access-Control-Allow-Origin": "*"
}
```

This allows any domain to access API responses.

---

### After — Fixed Configuration

```javascript
const ALLOWED_ORIGIN = "http://dvsa-website-test-546079881937-us-east-1.s3-website.us-east-1.amazonaws.com";

headers: {
  "Access-Control-Allow-Origin": ALLOWED_ORIGIN
}
```

This restricts cross-origin access to the trusted DVSA frontend only.

---

## Verification After Fix

After applying the fix, re-run the malicious-origin request:

```bash
curl -i "$API" -H "Origin: http://evil.com" -H "authorization: $TOKEN" --data-raw '{"action":"orders"}'
```

Expected result:

```http
access-control-allow-origin: http://dvsa-website-test-546079881937-us-east-1.s3-website.us-east-1.amazonaws.com
```

The wildcard is no longer returned.

This confirms that:

- `Access-Control-Allow-Origin: *` was removed
- The API no longer trusts every origin
- Only the trusted DVSA frontend origin is allowed

---

## Structured Operation and Security Analysis

### Table A — Intended vs Exploit Behavior

| Vulnerability | Intended Rule(s) | Artifacts Used to Infer Rule | Normal Behavior Evidence | Exploit Behavior Evidence |
|---|---|---|---|---|
| CORS Misconfiguration | API should only allow trusted origins | `curl` output, response headers, Lambda code | Requests return data normally to the trusted frontend | Malicious origin is allowed and receives response because wildcard CORS is enabled |

---

### Table B — Deviation and Fix

| Vulnerability | Why This Is a Deviation | Deviation Class | Fix Applied | Post-Fix Verification | Optional Latency |
|---|---|---|---|---|---|
| CORS Misconfiguration | API allows any origin using `*`, violating the trusted-origin rule | Security misuse / misconfiguration | Restricted allowed origin in Lambda response headers | Malicious origin no longer receives wildcard access | N/A |

---

## Takeaway / Lessons Learned

This finding shows that CORS configuration is part of application security, especially in serverless applications using Lambda proxy integration.

A secure API should not use wildcard origins when sensitive or authenticated data may be returned. Instead, the backend should allow only trusted frontend domains.

Key lessons:

- Do not use `Access-Control-Allow-Origin: *` for sensitive APIs
- Restrict CORS to trusted frontend domains
- Understand whether CORS is controlled by API Gateway or Lambda
- Authentication is still required, but CORS must not weaken the browser security boundary
- Serverless response headers must be reviewed as part of security testing

---

## Repository Safety Note

This README intentionally uses placeholders for sensitive values.

Do not upload:

- Real JWT tokens
- AWS access keys
- AWS secret keys
- Session tokens
- Private account identifiers
- Screenshots exposing secrets
