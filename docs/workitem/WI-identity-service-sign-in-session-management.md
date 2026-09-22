# Implementation Plan: Identity Service — Sign-In & JWT Session Management

## Source requirement

[REQ-staff-and-patient-authentication](../requirements/REQ-staff-and-patient-authentication.md) —
"Staff and Patient Authentication with Session Management." Scope: **sign-in
only** for accounts already provisioned elsewhere (Hospital Admin creates
staff accounts; patient accounts are generated at registration time and
delivered via WhatsApp). No sign-up/self-registration UI. Auth strategy:
JWT access token + refresh session.

## Approach

Implement the sign-in surface of the Identity Service (blueprint
Microservices Catalog #1: "Authentication, session/JWT, RBAC").

1. **`POST /auth/sign-in`**
   - Accepts `{ userId, password }`.
   - Looks up the account (staff or patient) by `userId`, verifies the
     password hash.
   - On success, issues:
     - a short-lived **JWT access token** with claims `{ sub, role,
       hospitalId (tenant, null for Super Admin), exp }`
     - a **refresh session**: an opaque refresh token persisted
       server-side (e.g. a `refresh_sessions` table/collection keyed by
       token hash, user id, expiry, revoked flag)
   - On failure, returns a generic 401 (do not distinguish "unknown user"
     vs "wrong password").

2. **`POST /auth/refresh`**
   - Accepts `{ refreshToken }`.
   - Validates the refresh session (exists, not expired, not revoked).
   - Issues a new access token; optionally rotates the refresh token
     (recommended, to limit replay window).

3. **`POST /auth/sign-out`**
   - Accepts `{ refreshToken }`.
   - Marks the corresponding refresh session as revoked.

4. **RBAC/tenant claims**
   - Access token payload must carry enough to let downstream services
     enforce the Roles & Permissions table from the blueprint and scope
     data by `hospitalId`. This work item is only responsible for
     populating those claims correctly at issuance — enforcing per-role
     permissions in each downstream service is out of scope here.

## Areas / files likely affected

- Identity Service: sign-in, refresh, sign-out handlers/controllers
- Password hashing/verification utility (if not already present)
- Refresh-session storage (new table/collection + migration)
- JWT signing/verification utility (shared secret or key pair, token TTL
  config)
- Any existing auth middleware in downstream services that will consume
  the JWT (verify signature, read `role`/`hospitalId` claims) — this work
  item issues the token; wiring every downstream service to consume it may
  warrant its own follow-up work items per service.

## Assumptions / risks (carried over from the requirement, not re-litigated here)

- Password reset / forgot-password is explicitly out of scope per the
  requirement doc; not implemented here.
- Patient credential generation and WhatsApp delivery are owned by the
  patient-registration flow, not this work item — this work item only
  consumes those credentials at sign-in time.
- Access token / refresh session TTLs are not numerically specified in the
  requirement's clarifications ("short-lived" / "longer-lived" only) —
  recommend defaulting to a conventional split (e.g. 15 min access token,
  7-30 day refresh session) and confirming with product before ship, since
  this materially affects UX (how often a user is forced to re-enter
  credentials) and security posture.
