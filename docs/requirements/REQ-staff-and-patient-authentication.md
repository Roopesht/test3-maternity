# Staff and Patient Authentication with Session Management

## Summary

Provide sign-in and session management for all Maternal Care Platform users
— hospital staff (Super Admin, Hospital Admin, USG Staff, Lady Doctor,
Entry/Registration Staff, LR Staff, Billing Reception, RMO, Incident
Manager) and Patients — via the Identity Service. This requirement covers
**sign-in only**: account creation/provisioning is out of scope here and
remains the responsibility of Hospital Admin (for staff) and the platform
(for patients), per the Microservices Catalog note that the Identity
Service "does not create users."

## Background

- Per the blueprint's Microservices Catalog, the Identity Service is
  responsible for "Authentication, session/JWT, RBAC."
- Staff accounts are provisioned by the Hospital Admin as part of hospital
  onboarding/staff management — never self-signed-up.
- Patients follow the "Patient-side chain": a patient is registered into
  the system by staff (USG Staff or Lady Doctor) *before* the patient ever
  logs in themselves ("Patient Registration, without patient login").
- The blueprint flagged "Sign up vs Sign in" as an open decision with no
  elaboration; this requirement resolves that ambiguity by scoping strictly
  to sign-in for already-provisioned accounts (see Clarifications below).

## Clarifications (resolved via QnA)

1. **How do patients get credentials?**
   Once a patient is registered by staff, the system generates a user ID
   and password for that patient and sends both to the patient via
   WhatsApp. The patient uses these credentials to sign in — there is no
   patient self-signup flow.
2. **Sign-up vs. sign-in scope.**
   This requirement covers sign-in only, for accounts already provisioned
   by Hospital Admin (staff) or generated at patient-registration time
   (patients). No self-registration/sign-up UI is in scope.
3. **Token/session strategy.**
   Authentication uses a JWT access token paired with a refresh session:
   the access token is short-lived and used to authorize API calls; a
   longer-lived refresh session/token is used to silently obtain new access
   tokens without forcing the user to log in again, until the refresh
   session itself expires or is revoked.

## Functional Requirements

1. **Sign-in form.** A single sign-in screen accepts a user ID and
   password, used by every role (Super Admin, all hospital staff roles, and
   Patient).
2. **Credential verification.** The Identity Service verifies the
   submitted user ID/password against the account provisioned earlier
   (by Hospital Admin for staff, or generated at patient registration and
   delivered via WhatsApp for patients). Invalid credentials are rejected
   with a generic error (no distinction between "unknown user" and "wrong
   password," to avoid user enumeration).
3. **Token issuance.** On successful verification, the Identity Service
   issues:
   - a short-lived **JWT access token**, carrying the user's role/tenant
     claims for RBAC and tenant-scoping downstream, and
   - a **refresh session** (opaque refresh token or server-side session
     record) usable to obtain new access tokens.
4. **Session refresh.** While the refresh session is valid, the client can
   exchange it for a new access token without re-entering credentials. If
   the refresh session is expired, revoked, or invalid, the user is
   required to sign in again.
5. **Sign-out.** Signing out invalidates the current refresh session
   server-side (so a stolen refresh token can't be reused after logout) and
   discards the access token client-side.
6. **RBAC enforcement.** The issued access token's role claim is used by
   downstream services to enforce the role permissions defined in the
   blueprint's Roles & Permissions table (e.g. only USG Staff/Lady Doctor
   can register patients; only Hospital Admin manages tenant configuration,
   etc.). Enforcing those specific per-role permissions is the concern of
   each downstream service; this requirement is only responsible for
   issuing a token that correctly identifies the user's role and tenant.
7. **Tenant scoping.** For staff and patients, the token identifies the
   hospital (tenant) the account belongs to, so downstream services can
   scope data access accordingly (per the multi-tenancy model). Super Admin
   accounts are platform-wide and are not scoped to a single tenant.
8. **No self-signup surface.** There is no "create account" or "sign up"
   entry point anywhere in this flow — the only account-creation paths are
   Hospital Admin provisioning staff and the platform generating patient
   credentials at registration time, both outside this requirement's scope.

## Out of Scope

- Staff account creation/provisioning (owned by Hospital Admin, separate
  requirement).
- Patient registration and credential generation/WhatsApp delivery
  mechanics (owned by the Antenatal/Patient Registry registration flow,
  separate requirement) — this requirement only consumes the resulting
  credentials at sign-in time.
- Password reset / forgot-password flow (not covered by the QnA answers;
  should be scoped separately if needed).
- Hospital self-registration/onboarding sign-up (explicitly excluded per
  Clarification 2).

## Open Follow-ups

- Password reset/forgot-password was not addressed by this requirement's
  QnA and will need its own requirement if needed.
