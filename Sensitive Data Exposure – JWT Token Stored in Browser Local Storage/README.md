# Extra Vulnerability #4: Sensitive Data Exposure — JWT Token Stored in Browser Local Storage

## ICS-344 Information Security — DVSA Bonus Finding

This repository documents **Extra Vulnerability #4: Sensitive Data Exposure — JWT Token Stored in Browser Local Storage** discovered in the Damn Vulnerable Serverless Application (DVSA) project for ICS-344 Information Security.

The issue demonstrates that authentication tokens can be reused outside the original browser session, allowing protected API access if the token is exposed.

---

## Vulnerability Overview

| Item | Description |
|---|---|
| Vulnerability | Sensitive Data Exposure |
| Specific Issue | JWT token stored in browser local storage / reusable token |
| Type | Bonus / Extra Finding |
| Affected Area | Frontend authentication storage and API authorization |
| Entry Point | API Gateway `/order` endpoint |
| Main Impact | Token reuse, session hijacking, unauthorized API access |
| AWS Region | `us-east-1` |
| Main AWS Services | API Gateway, Amazon Cognito, Lambda |

---

## Goal

The goal of this finding is to show that a valid JWT token is enough to access protected DVSA API endpoints even when reused outside the original browser session.

If an attacker obtains the token, they can manually send API requests from another terminal or environment and retrieve protected user data without logging in through the browser.

---

## Root Cause

The weakness exists because the application relies only on the presence of a valid JWT token for authentication.

The token is:

- Not bound to a specific browser session
- Not bound to a specific client device
- Reusable from another terminal or environment
- Potentially exposed if stored in JavaScript-accessible storage such as `localStorage`

A vulnerable frontend pattern is:

```javascript
localStorage.setItem("token", jwt);
```

Since `localStorage` is accessible through JavaScript, any successful XSS or malicious browser script could read the token and reuse it.

---

## Environment and Setup

### AWS Region

```text
us-east-1 (N. Virginia)
```

### Application

```text
DVSA — Damn Vulnerable Serverless Application
```

### Authentication

```text
AWS Cognito JWT token
```

### Tools Used

- Kali Linux terminal
- `curl`
- `jq`
- Browser / DevTools

### Safe Environment Variables

```bash
export API="https://<API_ID>.execute-api.us-east-1.amazonaws.com/dvsa/order"
export TOKEN="<YOUR_VALID_JWT_TOKEN>"
```

> Do not commit real JWT tokens, AWS credentials, or screenshots showing secrets to GitHub.

---

## Reproduction Steps

### Step 1 — Authenticate Normally

Log in to DVSA normally and obtain a valid JWT token from the browser session.

The token should be treated as sensitive data.

---

### Step 2 — Use the Token to Access Protected API

Send an API request using the valid token.

```bash
curl -s "$API" -H "content-type: application/json" -H "authorization: $TOKEN" --data-raw '{"action":"orders"}' | jq
```

Expected result:

- The request succeeds
- The API returns the authenticated user’s orders

---

### Step 3 — Remove the Token

Remove the token from the current terminal session.

```bash
unset TOKEN
```

Then send the request again.

```bash
curl -s "$API" -H "content-type: application/json" -H "authorization: $TOKEN" --data-raw '{"action":"orders"}' | jq
```

Expected result:

- The request fails
- The API returns an authentication-related error

---

### Step 4 — Reuse the Same Token in a Separate Session

Open a new terminal session and manually paste the same previously captured token.

```bash
export TOKEN="<PASTED_STOLEN_OR_REUSED_JWT_TOKEN>"
```

Then send the request again.

```bash
curl -s "$API" -H "content-type: application/json" -H "authorization: $TOKEN" --data-raw '{"action":"orders"}' | jq
```

Expected vulnerable result:

- Access is granted again
- The API returns protected user order data
- The token works outside the original browser session

---

## Evidence and Proof

The vulnerability was confirmed through three observations.

| Evidence | Description |
|---|---|
| Figure 1 | Normal authenticated access returns user orders |
| Figure 2 | Removing the token causes the request to fail |
| Figure 3 | Reusing the same token in a separate session successfully retrieves user data |

These results show that the token alone is sufficient to access protected resources.

---

## Security Impact

This weakness can lead to session hijacking if the token is exposed.

Potential impact includes:

- Unauthorized access to user data
- Reuse of stolen JWT tokens outside the original browser
- Account/session impersonation
- Increased damage if combined with XSS
- Weak client-side session protection

This is especially important because tokens stored in `localStorage` can be read by JavaScript. If an attacker can execute JavaScript in the victim’s browser, the token can be stolen and reused.

---

## Fix Strategy

The main mitigation is to prevent authentication tokens from being exposed to JavaScript and reduce the risk of token reuse.

Recommended mitigations:

- Avoid storing JWTs in `localStorage`
- Use `HttpOnly` cookies for token storage
- Enable the `Secure` flag so cookies are only sent over HTTPS
- Use `SameSite=Strict` or `SameSite=Lax`
- Use short-lived access tokens
- Use refresh-token rotation
- Revoke tokens on logout where possible
- Add server-side checks for sensitive operations

---

## Code / Configuration Changes

### Before — Vulnerable Storage Pattern

```javascript
localStorage.setItem("token", jwt);
```

This stores the token in a JavaScript-accessible location.

---

### After — Safer Token Storage Pattern

```http
Set-Cookie: token=<jwt>; HttpOnly; Secure; SameSite=Strict
```

This prevents client-side JavaScript from directly reading the token.

---

## Verification After Fix

After applying secure token handling:

1. The token should no longer be accessible through JavaScript.
2. Browser scripts should not be able to read the token directly.
3. Removing or invalidating the session should prevent API access.
4. Manually reused tokens should be limited by expiration, revocation, and server-side checks.

Expected secure behavior:

```json
{
  "status": "err",
  "msg": "invalid or expired token"
}
```

or another authentication failure response.

---

## Structured Operation and Security Analysis

### Table A — Intended vs Exploit Behavior

| Vulnerability | Intended Rule(s) | Artifacts Used to Infer Rule | Normal Behavior Evidence | Exploit Behavior Evidence |
|---|---|---|---|---|
| Insecure Storage of JWT Tokens in Browser Local Storage | Authentication tokens must not be accessible or reusable outside a secure client session | Terminal output, API responses, `curl` requests | Valid token allows access to user orders through authenticated requests | Reused token in a separate session successfully retrieves user data |

---

### Table B — Deviation and Fix

| Vulnerability | Why This Is a Deviation | Deviation Class | Fix Applied | Post-Fix Verification | Optional Latency |
|---|---|---|---|---|---|
| Insecure Storage of JWT Tokens in Browser Local Storage | The token can be reused outside its original session, violating secure session-binding principles | Security misuse | Store token in `HttpOnly`, `Secure`, `SameSite` cookies instead of JavaScript-accessible storage | Access fails when token is removed or invalidated; exposed tokens are no longer readable by JavaScript | N/A |

---

## Takeaway / Lessons Learned

This finding shows that authentication tokens must be protected as sensitive data.

If a JWT token is exposed and reusable, an attacker may access protected APIs without knowing the user’s password. Secure token storage is therefore critical in serverless web applications.

Key lessons:

- Do not store sensitive tokens in `localStorage`
- Use `HttpOnly` and `Secure` cookies where possible
- Treat JWTs like passwords or session credentials
- Use short token lifetimes and token rotation
- Protect against XSS because JavaScript-accessible tokens are high risk
- Authentication is only secure if token handling is also secure

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
