# crAPI Security Assessment — Black-Box API Testing

> A hands-on security assessment of OWASP crAPI, an intentionally vulnerable API training application. This project documents black-box testing focused on Broken Object Level Authorization (BOLA), IDOR, data exposure, and authentication protections.

## Disclaimer and scope

This assessment was performed against a deliberately vulnerable crAPI lab environment for educational purposes. It is not an assessment of a production service. All evidence published here should be sanitized: do not include real credentials, access tokens, session cookies, personal data, or unredacted responses.

**Testing approach:** black-box. The assessment was based on application behavior, HTTP requests and responses, and the tester's observations rather than source-code review.

**Focus areas:**
- Broken Object Level Authorization (BOLA) / Insecure Direct Object Reference (IDOR)
- Exposure of data through API endpoints
- Authentication weaknesses, including resistance to repeated login attempts

## Application under test

crAPI (Completely Ridiculous API) is an intentionally vulnerable, microservice-based application created by OWASP for learning and practicing API security. Its features model vehicle ownership, mechanics and service requests, user identity, and a community.

- Project: [OWASP crAPI](https://github.com/OWASP/crAPI)
- Official overview: [docs/overview.md](https://github.com/OWASP/crAPI/blob/develop/docs/overview.md)
- Official challenges: [docs/challenges.md](https://github.com/OWASP/crAPI/blob/develop/docs/challenges.md)
- Official OpenAPI specification: [crapi-openapi-spec.json](https://github.com/OWASP/crAPI/blob/develop/openapi-spec/crapi-openapi-spec.json)
- Setup guide: [docs/setup.md](https://github.com/OWASP/crAPI/blob/develop/docs/setup.md)

The OpenAPI specification is the best starting point for a broad inventory of documented API operations. It may not include every route discovered through black-box exploration, hidden endpoints, UI-only routes, or routes that differ between crAPI versions. The findings below therefore record the endpoints observed during this assessment, not a claim that this file is an exhaustive inventory of every route in every release.

## Executive summary

The assessment identified six observations for documentation and validation:

| ID | Endpoint / feature | Preliminary classification | Main security concern |
|---|---|---|---|
| CRAPI-01 | `/vehicle-service-dashboard?VIN=<VIN>` | BOLA / IDOR | Access to another user's vehicle-related data by changing the VIN |
| CRAPI-02 | `/api/v2/vehicle/<carId>/location` | BOLA / IDOR | Access to another vehicle's location by changing the vehicle identifier |
| CRAPI-03 | `/api/mechanic/service_request/<serviceId>` | BOLA / IDOR | Access to another user's service-request information by changing the service identifier |
| CRAPI-04 | `/contact-mechanic?VIN=<VIN>` | BOLA / IDOR-related workflow | Ability to open or submit a mechanic report associated with another user's vehicle |
| CRAPI-05 | `/community/api/v2/community/posts/` | Sensitive data exposure / authorization review | Response contains information belonging to other users |
| CRAPI-06 | Login page / authentication flow | Weak authentication protection / brute-force resistance | Repeated password guesses against an administrative account were used to identify valid credentials in the lab |

These classifications describe the tester's reported observations. The detailed evidence should specify the exact request method, relevant parameters, authentication state, expected result, observed result, and impact. Classification and severity should be finalized against the exact crAPI version and reproducible evidence.

## Findings

### CRAPI-01 — BOLA in vehicle service dashboard

**Endpoint:** `/vehicle-service-dashboard?VIN=<VIN>`  
**Reported behavior:** Changing the VIN parameter allowed the tester to view sensitive information associated with another user's vehicle.

**Security impact:** Depending on the returned fields, unauthorized access could expose vehicle details and information linking a vehicle to its owner.

**Recommended remediation:**
- Derive the authorized user from the authenticated server-side identity.
- Verify ownership or explicit access rights for the requested vehicle on every request.
- Do not rely on an unpredictable identifier or a client-supplied VIN as an authorization control.
- Return only the fields required by the current user and function.

**Evidence to include:** Sanitized request and response, account A versus account B comparison, and proof that account A cannot legitimately access account B's vehicle.

### CRAPI-02 — BOLA in vehicle location endpoint

**Endpoint:** `/api/v2/vehicle/<carId>/location`  
**Reported behavior:** Changing `carId` allowed the tester to retrieve location information for another vehicle.

**Security impact:** Vehicle location can be sensitive and may reveal movement patterns or physical whereabouts.

**Recommended remediation:**
- Enforce object-level authorization for each vehicle lookup.
- Scope the database query to both the requested vehicle and the authenticated user's permitted vehicles.
- Avoid returning precise location information to users without a valid business need.
- Add negative authorization tests for cross-account access.

**Evidence to include:** Sanitized requests using two test accounts and clearly synthetic vehicle identifiers/location values.

### CRAPI-03 — BOLA in mechanic service requests

**Endpoint:** `/api/mechanic/service_request/<serviceId>`  
**Reported behavior:** Changing `serviceId` allowed the tester to view service-request information belonging to another user.

**Security impact:** Service requests may contain customer details, vehicle information, issue descriptions, or other operational data.

**Recommended remediation:**
- Check authorization against the service request's owner and permitted mechanic/staff roles.
- Apply authorization in the service layer or data-access layer, not only in the UI.
- Return a generic not-found or forbidden response where appropriate, without leaking the existence of unauthorized objects.
- Test access across users and roles.

### CRAPI-04 — Unauthorized vehicle reference in the contact-mechanic workflow

**Endpoint / feature:** `/contact-mechanic?VIN=<VIN>`  
**Reported behavior:** Changing the VIN allowed the tester to open or create a mechanic report associated with another vehicle.

**Security impact:** Depending on the workflow, this could enable unauthorized actions on another user's vehicle record or create misleading service reports.

**Recommended remediation:**
- Verify that the authenticated user is allowed to submit a report for the specified vehicle.
- Bind report creation to the authorized vehicle record on the server.
- Validate ownership and workflow state before processing the request.
- Log and monitor suspicious cross-account attempts.

**Evidence to include:** Document whether the behavior reads an existing report, opens a form, or submits a new report. Those are distinct security impacts and should not be conflated.

### CRAPI-05 — Exposure of other users' community data

**Endpoint:** `/community/api/v2/community/posts/`  
**Reported behavior:** The response exposed data associated with other users.

**Security impact:** Depends on whether the exposed fields are intended to be public. Public posts are not inherently a vulnerability; private profile fields, internal identifiers, contact information, or non-public content may be.

**Recommended remediation:**
- Define which post fields are public and enforce a response schema/allowlist.
- Apply authorization checks to private posts and related resources.
- Avoid serializing internal user objects wholesale.
- Add tests that verify no non-public user attributes appear in public API responses.

**Evidence to include:** List the exact exposed field names, explain why each field should not be public, and redact all values.

### CRAPI-06 — Weak resistance to brute-force login attempts

**Feature:** Login page / authentication endpoint  
**Reported behavior:** After identifying the administrative account email, the tester repeatedly attempted passwords and discovered the account password in the intentionally vulnerable lab.

**Security impact:** If the login flow lacks effective controls, attackers may guess credentials and gain access to a privileged account.

**Recommended remediation:**
- Apply rate limits and progressive delays to repeated failed login attempts.
- Detect and alert on credential-stuffing and brute-force patterns.
- Use multi-factor authentication for privileged accounts.
- Prevent account enumeration and avoid exposing different error details for invalid users and passwords.
- Use strong, unique passwords and secure password storage.
- Test protections across source IPs, accounts, and sessions; avoid relying on a single easily bypassed counter.

**Evidence to include:** Number and rate of attempts, response behavior, whether throttling or lockout occurred, and the exact endpoint/method observed. Never publish the discovered password, a working wordlist containing it, or a usable administrative token.

## Methodology

1. Map the application features and observe normal user workflows.
2. Capture baseline requests with an account controlled by the tester.
3. Identify object references such as VINs, vehicle IDs, and service-request IDs.
4. Change one identifier at a time while keeping other request properties constant.
5. Compare behavior between two tester-controlled accounts and record whether server-side authorization is enforced.
6. Assess data returned by community endpoints and distinguish public fields from information that should be restricted.
7. Observe login defenses during authorized lab testing and record rate limiting, lockout, and response differences.
8. Save sanitized evidence and repeat tests to confirm reproducibility.

## How to document each finding

Use this template for each detailed report:

- **Title**
- **Affected endpoint and HTTP method**
- **Affected crAPI version / deployment**
- **Severity and rationale**
- **CWE / OWASP API Security Top 10 mapping**
- **Prerequisites and attacker role**
- **Expected behavior**
- **Observed behavior**
- **Reproduction steps using synthetic test accounts**
- **Sanitized request and response**
- **Impact**
- **Root cause hypothesis** (label as a hypothesis unless source code was reviewed)
- **Recommended remediation**
- **Retest result**

Do not publish secrets, real personal data, complete JWTs, session cookies, or sensitive response bodies. Use placeholders such as `<REDACTED_TOKEN>`, `<TEST_USER_A_VIN>`, and `<TEST_USER_B_VIN>`.

## Endpoint inventory and official references

The assessment endpoints listed above were reported during black-box testing. For the broader documented API surface, consult the upstream OpenAPI specification and the official crAPI documentation:

- [OpenAPI specification — all documented paths and methods](https://github.com/OWASP/crAPI/blob/develop/openapi-spec/crapi-openapi-spec.json)
- [OWASP crAPI challenges](https://github.com/OWASP/crAPI/blob/develop/docs/challenges.md)
- [crAPI architecture](https://github.com/OWASP/crAPI/blob/develop/docs/crAPI_architecture.md)
- [crAPI overview](https://github.com/OWASP/crAPI/blob/develop/docs/overview.md)

For a reproducible inventory, download the OpenAPI JSON from the exact branch/tag used by the lab and extract each `paths` entry and HTTP method. Record the commit hash or release tag, because endpoint paths can change across versions. Black-box discovery can also reveal routes that are absent from the published specification.

## Learning outcomes

This assessment reinforced several core API security principles:

- Authentication answers **who is making the request**; authorization answers **whether that identity may access this specific object**.
- Every endpoint that accepts an object identifier needs an object-level authorization decision.
- Unpredictable IDs are defense-in-depth, not a substitute for authorization.
- Data exposure must be evaluated field by field and against the intended privacy model.
- Authentication endpoints need layered controls against repeated guessing, credential stuffing, and account enumeration.

## References

- OWASP crAPI: https://github.com/OWASP/crAPI
- OWASP API Security Project: https://owasp.org/www-project-api-security/
- OWASP API Security Top 10 (2023): https://owasp.org/API-Security/editions/2023/en/0x00-header/

---

**Project status:** Initial assessment notes. Add sanitized evidence, exact methods, environment/version details, severity ratings, and retest results before presenting this as a final professional report.
