# ICS-344 DVSA Vulnerability Discovery and Remediation Project

## Course Information

| Item | Details |
|---|---|
| Course | ICS-344 — Information Security |
| Section | 02 |
| University | King Fahd University of Petroleum and Minerals (KFUPM) |
| Project | DVSA Vulnerability Discovery and Remediation |
| Application | OWASP Damn Vulnerable Serverless Application (DVSA) |
| AWS Region | `us-east-1` — N. Virginia |
| DVSA Website URL | `http://dvsa-website-test-546079881937-us-east-1.s3-website.us-east-1.amazonaws.com/` |
| Submission Date | 19 April 2026 / Bonus updates: 3 May 2026 |

---

## Team Members

| Student | Student ID | Main Contributions |
|---|---:|---|
| Ali Kamal Aloryd | 202322750 | Lesson 1, Lesson 3, Lesson 4, Lesson 7, Lesson 9, Extra Vulnerability #1, Extra Vulnerability #2 |
| Redha Abdulaziz Alturaik | 20232301 | Lesson 2, Lesson 5, Lesson 6, Lesson 8, Lesson 10, Extra Vulnerability #3, Extra Vulnerability #4 |

---

## Project Overview

This repository contains the documentation, evidence, reproduction steps, fixes, and verification results for the ICS-344 DVSA security project.

The goal of the project is to analyze the OWASP Damn Vulnerable Serverless Application (DVSA), identify security weaknesses, reproduce each vulnerability in a controlled AWS lab environment, apply suitable mitigations, and verify that the vulnerable behavior is removed after remediation.

The project focuses on AWS serverless security concepts such as:

- API Gateway request handling
- Lambda backend logic
- DynamoDB data access
- S3 bucket configuration
- IAM least privilege
- JWT authentication
- CloudWatch and CloudTrail evidence
- Secure error handling
- Input validation
- Dependency security
- Workflow and business logic protection

> DVSA is intentionally vulnerable and must only be deployed in a private, non-production AWS account for legal educational testing.

---

## Repository Structure

The repository is organized by vulnerability lesson and bonus finding.

```text
.
├── CORS Misconfiguration/
├── Extra Vulnerability #1/
├── Extra Vulnerability #2/
├── Lesson 07 Over-Privileged Function/
├── Lesson 1-Event Injection/
├── Lesson 10- Unhandled Exceptions/
├── Lesson 2- Broken Authentication/
├── Lesson 3- Sensitive Information Disclosure/
├── Lesson 4- Insecure Cloud Configuration/
├── Lesson 5- Broken Access Control/
├── Lesson 6- Denial of Service (DoS)/
├── Lesson 8- Logic Vulnerabilities/
├── Lesson-9/
└── Sensitive Data Exposure – JWT Token Stored in Browser Local Storage/
```

Each folder contains the related report files, screenshots, evidence, and README documentation for that vulnerability.

---

## Official Vulnerability Lessons

| Lesson | Vulnerability | Main Component(s) | Summary | Contributor |
|---:|---|---|---|---|
| 1 | Event Injection | API Gateway, `DVSA-ORDER-MANAGER`, CloudWatch | Demonstrates unsafe deserialization using `node-serialize`, allowing attacker-controlled input to execute inside Lambda. | Ali Aloryd |
| 2 | Broken Authentication | Cognito JWT, API Gateway, Lambda | Demonstrates JWT forgery where identity claims are modified because the backend decodes tokens without proper signature verification. | Redha Alturaik |
| 3 | Sensitive Information Disclosure | `DVSA-ORDER-GET`, DynamoDB | Demonstrates that a user can access another user’s order by modifying the `order-id` parameter due to missing ownership validation. | Ali Aloryd |
| 4 | Insecure Cloud Configuration | S3, Lambda, CloudWatch | Demonstrates insecure S3 configuration where attacker-controlled files can be uploaded into a trusted backend storage path. | Ali Aloryd |
| 5 | Broken Access Control | API Gateway, Lambda, DynamoDB | Demonstrates unauthorized update of order state through direct API requests, bypassing intended workflow restrictions. | Redha Alturaik |
| 6 | Denial of Service (DoS) | API Gateway, Lambda billing workflow | Demonstrates that repeated concurrent billing requests can overload the backend due to missing rate limiting and request throttling. | Redha Alturaik |
| 7 | Over-Privileged Function | Lambda execution role, IAM Policy Simulator, CloudTrail | Evaluates whether the receipt Lambda function has excessive permissions and verifies access behavior using IAM tools. | Ali Aloryd |
| 8 | Logic Vulnerabilities | Order update and billing workflow | Demonstrates workflow/race-condition risks where concurrent update and billing requests interact with the same order state. | Redha Alturaik |
| 9 | Vulnerable Dependencies | `node-serialize`, Lambda | Demonstrates that an insecure third-party dependency can enable remote code execution during deserialization. | Ali Aloryd |
| 10 | Unhandled Exceptions | `DVSA-ORDER-GET`, API Gateway, CloudWatch | Demonstrates that malformed requests expose internal stack traces, file paths, line numbers, and exception details. | Redha Alturaik |

---

## Bonus / Extra Vulnerabilities

| Extra Finding | Vulnerability | Main Component(s) | Summary | Contributor |
|---:|---|---|---|---|
| Extra #1 | Second-Order SQL Injection in Cart Total Calculation | `DVSA-GET-CART-TOTAL`, Lambda, CloudWatch | Malicious item data is stored during order creation and later reused unsafely in a backend SQL query during billing. | Ali Aloryd |
| Extra #2 | Additional Undocumented Vulnerability | DVSA backend workflow | Additional bonus vulnerability documented separately in its folder. | Ali Aloryd |
| Extra #3 | CORS Misconfiguration | API Gateway, `DVSA-ORDER-MANAGER` | API responses allowed `Access-Control-Allow-Origin: *`, allowing any origin to access responses. | Redha Alturaik |
| Extra #4 | Sensitive Data Exposure — JWT Token Stored in Browser Local Storage | Browser storage, JWT, API Gateway | Demonstrates that a JWT token can be reused outside the original browser session if exposed. | Redha Alturaik |

---

## High-Level Architecture

DVSA follows a serverless AWS architecture.

```text
User Browser
    │
    ▼
S3 Static Website Frontend
    │
    ▼
API Gateway
    │
    ▼
AWS Lambda Functions
    │
    ├── DynamoDB
    ├── S3 Buckets
    ├── SQS
    ├── SES
    └── CloudWatch Logs
```

Security testing focused on the request lifecycle from the frontend/API layer to Lambda functions and backend AWS services.

---

## Tools Used

The following tools were used during testing and documentation:

- AWS Management Console
- API Gateway Console
- Lambda Console
- IAM Policy Simulator
- CloudTrail
- CloudWatch Logs
- S3 Console
- Browser Developer Tools
- Kali Linux terminal
- `curl`
- `jq`
- `python3`
- AWS CLI

---

## General Testing Workflow

Most vulnerability lessons followed this workflow:

1. Identify the vulnerable component
2. Understand the intended application logic
3. Send normal requests to confirm baseline behavior
4. Send malicious or abnormal requests
5. Capture proof from terminal output, browser DevTools, AWS Console, or CloudWatch
6. Identify the root cause
7. Apply a code or configuration fix
8. Repeat the test after remediation
9. Confirm that malicious behavior is blocked
10. Confirm that legitimate behavior still works

---

## Security Controls Applied

Across the project, the following mitigation strategies were used or recommended:

- Replace unsafe deserialization with safe JSON parsing
- Verify JWT signatures and required claims
- Enforce ownership validation for sensitive data
- Restrict S3 permissions and enable Block Public Access
- Add server-side authorization checks
- Apply rate limiting and request throttling
- Enforce least privilege in IAM roles
- Add workflow state validation and conditional updates
- Remove or replace vulnerable dependencies
- Add structured exception handling
- Restrict CORS to trusted origins
- Store tokens securely using `HttpOnly`, `Secure`, and `SameSite` cookies
- Validate stored data before reuse
- Use parameterized database queries

---

## Evidence Types

The project includes evidence such as:

- API responses
- Terminal outputs
- `curl` command results
- Browser DevTools screenshots
- CloudWatch log entries
- IAM Policy Simulator results
- S3 permission screenshots
- Lambda code screenshots
- Post-fix verification screenshots

Each lesson folder contains the evidence relevant to that vulnerability.

---

## Important Security Notice

This repository is for educational use only.

The following sensitive values must not be committed:

- Real AWS access keys
- AWS secret keys
- AWS session tokens
- Valid JWT tokens
- Passwords
- Private account identifiers
- Unredacted secrets in screenshots

All reusable commands in the README files should use placeholders such as:

```bash
export API="https://<API_ID>.execute-api.us-east-1.amazonaws.com/<STAGE>/order"
export TOKEN="<YOUR_VALID_JWT_TOKEN>"
```

---

## How to Navigate This Repository

To review a specific vulnerability:

1. Open the folder for the lesson or extra vulnerability.
2. Read the folder-specific `README.md`.
3. Review the reproduction steps.
4. Check the screenshots and evidence files.
5. Review the fix strategy and code/configuration changes.
6. Review the post-fix verification.

Recommended review order:

```text
Lesson 1  → Event Injection
Lesson 2  → Broken Authentication
Lesson 3  → Sensitive Information Disclosure
Lesson 4  → Insecure Cloud Configuration
Lesson 5  → Broken Access Control
Lesson 6  → Denial of Service
Lesson 7  → Over-Privileged Function
Lesson 8  → Logic Vulnerabilities
Lesson 9  → Vulnerable Dependencies
Lesson 10 → Unhandled Exceptions
Bonus Findings → Extra Vulnerabilities
```

---

## Summary of Lessons Learned

This project showed that serverless security is not only about writing secure Lambda code. It also depends on secure configuration, strong identity validation, least privilege, safe dependency management, and careful workflow design.

Key takeaways:

- Authentication alone is not enough; authorization and ownership checks are required.
- Serverless functions must treat all input as untrusted, including data read from backend storage.
- Cloud resources such as S3 buckets and IAM roles are part of the application attack surface.
- Vulnerable dependencies can compromise otherwise simple backend logic.
- Error handling must protect internal implementation details.
- Rate limiting is necessary to protect availability.
- Secure design must be enforced at every trust boundary.

---

## Academic Integrity and Usage

This work was completed for the ICS-344 Information Security course project at KFUPM.

The project is intended for learning, reporting, and controlled security testing only. It must not be used against real systems, production AWS accounts, or unauthorized targets.
