# Mock: Sign-In Screen

```
+--------------------------------------------------+
|                 Maternal Care Platform            |
|                                                    |
|                    Sign In                        |
|                                                    |
|   User ID                                         |
|   +--------------------------------------------+  |
|   |                                              | |
|   +--------------------------------------------+  |
|                                                    |
|   Password                                        |
|   +--------------------------------------------+  |
|   |  ********                                   | |
|   +--------------------------------------------+  |
|                                                    |
|   [ ] Remember this device                         |
|                                                    |
|              +----------------------+              |
|              |       Sign In        |              |
|              +----------------------+              |
|                                                    |
|   Invalid user ID or password.   <- (error state)  |
|                                                    |
+--------------------------------------------------+

  Patients: your user ID & password were sent to you
  via WhatsApp when you were registered.
```

# Mock: Session/Token Flow

```
  Client                         Identity Service
    |                                   |
    |  POST /auth/sign-in              |
    |  { userId, password }            |
    |---------------------------------->|
    |                                   |  verify credentials
    |                                   |  (staff: HA-provisioned)
    |                                   |  (patient: WhatsApp-issued)
    |                                   |
    |  200 OK                          |
    |  { accessToken (JWT, short-lived)|
    |    refreshToken (session) }      |
    |<----------------------------------|
    |                                   |
    |  ... calls other services with   |
    |      Authorization: Bearer <JWT> |
    |                                   |
    |  accessToken expires             |
    |                                   |
    |  POST /auth/refresh              |
    |  { refreshToken }                |
    |---------------------------------->|
    |                                   |  validate refresh session
    |  200 OK                          |
    |  { accessToken (new) }           |
    |<----------------------------------|
    |                                   |
    |  POST /auth/sign-out             |
    |  { refreshToken }                |
    |---------------------------------->|
    |                                   |  revoke refresh session
    |  204 No Content                  |
    |<----------------------------------|
```

# Mock: Role-Aware Landing After Sign-In

```
  JWT claims: { sub, role, hospitalId (tenant), exp }

  role = "SUPER_ADMIN"        -> Platform-wide dashboard (no tenant scope)
  role = "HOSPITAL_ADMIN"     -> Hospital config dashboard (tenant-scoped)
  role = "USG_STAFF"          -> Registration / EDD worklist
  role = "LADY_DOCTOR"        -> Antenatal worklist + chat
  role = "ENTRY_STAFF"        -> Patient entry worklist
  role = "LR_STAFF"           -> Gate 1 unlock / delivery flag screen
  role = "BILLING_RECEPTION"  -> Charges & billing worklist
  role = "RMO"                -> Clinical worklist + chat
  role = "INCIDENT_MANAGER"   -> Incident/escalation dashboard
  role = "PATIENT"            -> Own record + chat (while ACTIVE)
```
