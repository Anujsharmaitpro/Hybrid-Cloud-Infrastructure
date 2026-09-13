# MODULE 4 — ENTRA ID AUTHENTICATION & CONDITIONAL ACCESS — DEEP DIVE

**Covers Part 5 (Entra ID Authentication) and Part 6 (Conditional Access — L3 Depth)**

---

# SECTION A — ENTRA ID AUTHENTICATION

---

## 4.1 CONCEPT

**Authentication** answers: "Who are you?"
**Authorization** answers: "What are you allowed to do?"

These are fundamentally different stages, and confusing them is one of the most common causes of misdiagnosis in enterprise troubleshooting.

```
Authentication (Entra ID's job)
  → "I verify your identity"
  → You present credentials (password, FIDO2, biometric, etc.)
  → Entra ID confirms you are who you claim to be
  → Issues tokens proving your identity

Authorization (Resource's job, guided by Entra ID policies)
  → "Now that I know who you are, what can you access?"
  → Resource checks your tokens, RBAC roles, group memberships
  → Resource determines if you have permission for the requested action
```

**The critical L3 insight:**
- An authentication problem = you cannot get tokens at all.
- An authorization problem = you get tokens but cannot use them for what you need.
- A Conditional Access problem = you may authenticate successfully but tokens are modified/limited/blocked.

**In practice:**
```
"I can't sign in" = Authentication issue (check Entra ID, credentials, MFA, CA)
"I can sign in but get Access Denied" = Authorization issue (check RBAC, Policy, resource-specific permissions)
"I can sign in but some features are blocked" = Conditional Access issue (check CA grant controls, session controls)
```

---

## 4.2 ARCHITECTURE

```
┌─────────────────── USER / APP ───────────────────┐
│                                                  │
│  Sign-in Request                                 │
│  (Username/UPN + Credential + MFA + Token Request)│
│                                                  │
└──────────────────┬───────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────┐
│            ENTRA ID ENDPOINT                      │
│            (login.microsoftonline.com)            │
│                                                  │
│  ┌─────────────┐  ┌──────────────────────────┐   │
│  │  TENANT     │  │  TENANT SETTINGS         │   │
│  │  LOOKUP     │  │  - Authentication Methods│   │
│  │  (which      │  │  - SSPR configuration    │   │
│  │   tenant?)   │  │  - CA policies           │   │
│  │              │  │  - Password policies     │   │
│  │              │  │  - Risk policies         │   │
│  │              │  │  - Named locations       │   │
│  │              │  │  - Cross-tenant settings │   │
│  └─────────────┘  └──────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │  AUTHENTICATION ENGINE                    │    │
│  │  1. Find user object in directory         │    │
│  │  2. Validate credentials                  │    │
│  │     - Local password → compare hash       │    │
│  │     - Federated → redirect to IdP         │    │
│  │     - Passwordless → validate FIDO2/WH   │    │
│  │  3. Account checks                        │    │
│  │     - Enabled? Blocked? Locked?           │    │
│  │     - MFA required? (CA policy)           │    │
│  │     - Risk level? (Identity Protection)   │    │
│  │     - Risk-based CA triggered?            │    │
│  │  4. MFA challenge (if required)           │    │
│  │     - Push notification, OTP, FIDO2, etc. │    │
│  │  5. Conditional Access EVALUATION         │    │
│  │     - All applicable policies checked     │    │
│  │     - Grant controls applied/modified     │    │
│  │     - Session controls set                │    │
│  │  6. TOKEN ISSUANCE                        │    │
│  │     - ID Token (identity)                 │    │
│  │     - Access Token (for resource)         │    │
│  │     - Refresh Token (session persistence) │    │
│  │  7. Return tokens + response              │    │
│  └──────────────────────────────────────────┘    │
│                                                  │
│  ┌──────────────────────────────────────────┐    │
│  │  TOKEN STORE (browser cookie / app cache) │    │
│  │  - Tokens cached for reuse                │    │
│  │  - Refresh token long-lived               │    │
│  │  - Access token short-lived               │    │
│  └──────────────────────────────────────────┘    │
└──────────────────────────────────────────────────┘
                   │
                   ▼ (Token presented)
┌──────────────────────────────────────────────────┐
│            TARGET RESOURCE                        │
│            (Azure VM, Storage, App, etc.)         │
│                                                  │
│  1. Validate token signature (JWKS)               │
│  2. Validate issuer (correct tenant)              │
│  3. Validate audience (token for this resource)   │
│  4. Validate expiry (not expired)                 │
│  5. Check RBAC permissions (role assignments)     │
│  6. Check resource-specific access                │
│  7. Check CA claims in token (if present)         │
│  8. GRANT or DENY access                         │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

## 4.3 COMPONENTS — DETAILED

### 4.3.1 AUTHENTICATION METHODS — COMPLETE DEEP DIVE

Authentication methods are configured at two levels:
- **Tenant-level** (Authentication Methods policy): Which methods are enabled globally?
- **Per-user** (My Profile → Authentication methods): Which methods has the user registered?

**For authentication to succeed, BOTH must allow it:**
- The tenant must have the method enabled
- The user must have the method registered

---

#### 4.3.1.1 Password-Based Authentication

**How it works:**
```
User enters UPN + password
→ Entra ID looks up user object
→ If local password: compares hash of entered password with stored hash
→ If federated: redirects to on-prem IdP (AD FS or Entra Connect PTA)
→ Match → Authentication successful
→ Mismatch → Failure (account locked out after attempts)
```

**Password properties:**
| Property | Description |
|----------|-------------|
| **Storage** | Hashed in Entra ID (for local) or on-prem (for federated) |
| **Complexity** | Configurable per tenant (length, complexity, history) |
| **Expiry** | Configurable (days until expiration) |
| **Lockout** | After X failed attempts, account temporarily locked |
| **Federated** | For hybrid environments — password validated on-prem |

**L3 troubleshooting:**
- "User locked out" → Wait for lockout duration, or admin unlocks account
- "Password expired" → Admin or SSPR resets
- "Federated password change not reflected" → PTA/PHS sync issue — check Entra Connect/Cloud Sync health

---

#### 4.3.1.2 Microsoft Authenticator

**What it provides:**
| Capability | Description |
|-----------|-------------|
| **Push notification** | User approves sign-in via notification on registered device |
| **OTP (TOTP)** | Time-based one-time password generated in app |
| **Passwordless** | Use Authenticator as primary sign-in method (no password) |
| **FIDO2 key** | Authenticator app can host FIDO2 credentials (on supported devices) |

**How push notification works:**
```
1. User enters UPN + password
2. Entra ID determines MFA required (CA policy or tenant setting)
3. Entra ID sends push notification to user's registered device(s)
4. User's Authenticator app displays: "Approve sign-in to [app] from [location]?"
5. User approves (device must be registered, network must reach Entra ID)
6. Entra ID receives approval → Completes authentication → Issues tokens
```

**L3 troubleshooting — Push notification issues:**
| Symptom | Cause | Fix |
|---------|-------|-----|
| No push notification received | Device not registered, app not installed, device offline | Register device, check network |
| Push received but "Approve" button missing | Outdated app, device not trusted | Update app, re-register |
| Push says "request from unrecognized location" | Location not in Named Locations, CA location condition | Check CA named locations |
| User cannot approve (device lost) | No backup method registered | Admin resets MFA, user re-registers |

---

#### 4.3.1.3 FIDO2 Security Key

**What it is:**
A hardware-based phishing-resistant authentication method (YubiKey, Titan, etc.). FIDO2 uses public-key cryptography.

**How it works:**
```
1. User enters UPN
2. Entra ID sends challenge to browser
3. Browser forwards challenge to FIDO2 key via USB/NFC/Bluetooth
4. FIDO2 key signs challenge with its private key
5. Entra ID verifies signature with the registered public key
6. Authentication successful
7. Tokens issued
```

**L3 operational impact:**
- FIDO2 is the **gold standard** for phishing-resistant authentication.
- Requires user registration (admin or self-service).
- If FIDO2 is the only method and user loses key → Admin must register backup method.
- Requires Entra ID Premium (P1 or P2) for full features.

---

#### 4.3.1.4 Passwordless Authentication

**What it is:**
Authentication without a password. Uses other credentials:
- FIDO2 key
- Microsoft Authenticator (with device binding)
- Windows Hello for Business (WHFB)
- TOTP (one-time password from app)
- Certificate-based authentication

**Tenant configuration:**
Authentication Methods policy → Enable passwordless methods → Set as required.

**Flow (FIDO2 passwordless example):**
```
User enters UPN → No password prompt → "Use FIDO2 key instead"
→ User touches FIDO2 key → Entra ID verifies → Token issued
```

**L3 best practice:**
- Passwordless reduces helpdesk burden (no password resets).
- Passwordless requires careful rollout — users must register methods in advance.
- Always provide at least two passwordless methods for redundancy.

---

#### 4.3.1.5 Temporary Access Pass (TAP)

**What it is:**
A time-limited, one-time-use password generated by an admin. Used for:
- Onboarding new users
- Reset scenarios where user has no registered MFA
- Break-glass emergency access

**Properties:**
| Property | Description |
|----------|-------------|
| **Validity** | Configurable (days: 1-30, default 14) |
| **One-time use** | After first successful sign-in, TAP is automatically deleted |
| **Generation** | Admin only (Global Admin, Privileged Role Admin, or delegated) |
| **MFA bypass** | TAP does NOT bypass MFA — but for some scenarios, MFA can be configured differently for TAP users |
| **Sent via** | Email or SMS to the user |

**L3 troubleshooting:**
- "TAP expired" → TAP is single-use and time-limited. Generate a new one.
- "User used TAP but now can't sign in" | TAP was consumed. User must now use registered MFA method or get new TAP.
- "TAP not received" → Check email/SMS delivery, spam filters, correct contact info.

---

#### 4.3.1.6 SMS and Voice (Deprecated trajectory)

**Current status (Sept 2026):**
- SMS and voice OTP for authentication are increasingly discouraged by Microsoft.
- They are being phased out in favor of Authenticator push, FIDO2, and TOTP.
- For SSPR: SMS/voice are deprecated for SSPR in favor of Authenticator-based methods.
- For CA: SMS and voice OTP are still functional but flagged as legacy.
- Risk: SMS is vulnerable to SIM-swapping attacks.

**L3 guidance:**
- Plan migration away from SMS/voice for authentication.
- If using SMS as a fallback, ensure Authenticator or FIDO2 is primary.
- Monitor deprecated feature notices in Portal.

---

### 4.3.2 SSPR (Self-Service Password Reset) — DEEP DIVE

**SSPR allows users to reset their own passwords without admin intervention.**

**Configuration (Authentication Methods policy):**
| Setting | Options |
|---------|---------|
| **Authentication methods** | Authenticator (push/OTP), email, security questions, SMS (deprecated), TOTP |
| **Who can reset** | All users, specific groups, or specific directory roles |
| **Require registration** | Users must register methods BEFORE using SSPR (recommended) |

**SSPR Registration Flow:**
```
1. User navigates to My Profile → Security info
2. User adds authentication methods (Authenticator app, email, phone)
3. Each method requires verification (code sent to registered channel)
4. Once registered, methods are stored in Entra ID for the user
5. When SSPR is triggered, user uses registered methods to verify identity
6. After verification (X out of Y methods, configurable), user sets new password
7. Password updated in Entra ID (or written back to on-prem via PTA)
```

**SSPR Reset Flow:**
```
1. User enters UPN on sign-in page or SSPR page
2. "Forgot password?" → SSPR flow starts
3. User selects registered authentication method(s)
4. User verifies identity (code from Authenticator push, email OTP, etc.)
5. User enters new password
6. Password updated
7. User can now sign in with new password
8. Audit log records: SSPR event (who, when, method used, success/failure)
```

**L3 operational impact:**
- SSPR reduces helpdesk password reset tickets by 70-90% in well-configured environments.
- If SSPR is not configured or users haven't registered methods → Helpdesk bears full burden.
- If PTA is configured and on-prem AD password policy differs from Entra ID policy → Confusion about which password "is current."
- SSPR audit logs are critical for security investigations (could a malicious user reset an admin password?).

---

### 4.3.3 MFA — Multi-Factor Authentication

**What it is:**
Authentication requiring more than one factor:
- Something you **know** (password, PIN)
- Something you **have** (phone, FIDO2 key, authenticator app)
- Something you **are** (biometric — fingerprint, face, Windows Hello)

**In Entra ID context:**
MFA in Entra ID is implemented through:
- **Authentication Methods policy** — which methods are available and required
- **Conditional Access** — which users/situations require MFA (this is the modern approach)
- **Legacy MFA** — per-user MFA settings (deprecated, being phased out)

**L3 critical concept:**
```
Legacy MFA (per-user) → User has MFA settings directly on their account
  → Applies to ALL apps uniformly
  → Cannot be context-aware (location, device, risk)
  → Being deprecated

Conditional Access MFA → CA policy triggers MFA based on conditions
  → Granular: different users, different situations, different requirements
  → Can use Authentication Strengths (confidence levels)
  → Preferred approach
```

**Per-user MFA deprecation:**
Microsoft is actively migrating users from per-user MFA settings to Conditional Access-based MFA. If your environment still uses per-user MFA:
- It will eventually stop working.
- All MFA should be managed through Conditional Access policies.
- The "Per-user" and "Cloud apps" settings in the legacy MFA portal are being retired.

---

### 4.3.4 MFA CHALLENGE FLOW (Detailed)

```
1. User enters UPN + password
2. Entra ID authenticates credentials → SUCCESS
3. Entra ID evaluates: Is MFA required for this sign-in?
   a. Per-user MFA setting (legacy, being deprecated) → YES
   b. Conditional Access policy → YES (if conditions met)
   c. Risk-based CA → YES (if risk level triggers)
   d. No MFA required → SKIP to token issuance
4. MFA required → MFA challenge initiated
   a. Entra ID identifies user's registered MFA methods
   b. Default method used (or user can select if multiple)
   c. Challenge sent to default method (push notification, OTP, etc.)
5. User responds to challenge
   a. Approve push notification → Entra ID receives approval
   b. Enter OTP → Entra ID validates OTP
   c. FIDO2 challenge → User touches key → Entra ID verifies
6. MFA verified → Authentication complete
7. Tokens issued (ID, Access, Refresh)
```

---

### 4.3.5 AUTHENTICATION LOGS

**Sign-in logs** are the primary source for authentication troubleshooting.

**Key fields in sign-in log:**
| Field | Description | Troubleshooting Value |
|-------|-------------|----------------------|
| **Id** | Unique log entry identifier | Correlation across systems |
| **Created DateTime** | Timestamp of sign-in attempt | Timeline analysis |
| **UserPrincipalName** | User who attempted sign-in | Identify affected users |
| **Status** | Success or Failure | Initial pass/fail |
| **StatusDetail** | Error code and message | Specific failure reason |
| **ConditionalAccessStatus** | NotApplied, Failed, Success, Applied | CA evaluation result |
| **AuthenticationRequirement** | MFA Required, Password Required, etc. | Why MFA was/is required |
| **AuthenticationDetails** | Per-auth method detail | Which MFA method used/failed |
| **IsInteractive** | Yes (user) or No (SP/daemon) | Interactive vs non-interactive |
| **AppliedConditionalAccess** | List of CA policies evaluated | Which policies applied |
| **ConditionalAccessOrganizations** | CA policy IDs and names | Link to specific CA policies |
| **IP Address** | Source IP | Location, VPN, Tor detection |
| **Device Detail** | Device info, compliance state | Device-based CA conditions |
| **Location** | City, State, Country (derived from IP) | Location-based CA conditions |
| **Risk Detail** | Sign-in risk level | Identity Protection |
| **Token Issuer** | Which tenant/IdP issued token | Multi-tenant/federation |
| **Correlation ID** | Links to Activity Log | Cross-system tracing |

**How to access:**
- Portal: Entra ID → Monitoring → Sign-in logs
- CLI: `Get-AzureADSignInLog` (via Microsoft Graph PowerShell)
- Graph API: `GET /auditLogs/signIns`
- Log Analytics: If diagnostic settings route sign-in logs to Log Analytics workspace

---

## 4.4 COMMUNICATION FLOW — COMPLETE AUTHENTICATION PIPELINE

### User → Entra ID → Authentication → Token → Resource Access

```
STEP 1: USER INITIATES SIGN-IN
  User opens app (e.g., Azure Portal, Office 365, custom app)
  → App redirects browser to:
    https://login.microsoftonline.com/{tenant-id}/oauth2/authorize
  → Parameters: client_id, redirect_uri, response_type, scope, state, nonce

STEP 2: TENANT RESOLUTION
  Entra ID receives request
  → Validates tenant ID from URL
  → Finds tenant configuration
  → If tenant-specific URL (login.microsoftonline.com/{tenant}/): use it
  → If common URL (login.microsoftonline.com/common/): resolves tenant from user's UPN

STEP 3: USER IDENTIFICATION
  User enters UPN
  → Entra ID searches directory for matching user
  → If found: proceed
  → If not found: error (invalid username)
  → If multiple matches (rare): prompt for more detail

STEP 4: AUTHENTICATION REQUIREMENT EVALUATION
  BEFORE password collection, Entra ID evaluates:
  → Is user account enabled? (AccountEnabled = true)
  → Is user blocked sign-in? (isBlocked = true) → Block immediately
  → Is user in risk list? (Identity Protection) → Risk-based CA may trigger
  → What authentication methods are available?
  → Is MFA required? (CA policy, per-user, or risk-based)

STEP 5: CREDENTIAL VALIDATION
  User enters password
  → IF local password:
    Entra ID compares entered password hash with stored hash
    → Match: continue
    → No match: fail, increment failed attempt counter
    → After N failures: account locked (temporary)
  
  → IF federated (PTA/AD FS):
    Entra ID redirects to federation endpoint
    → On-prem AD validates password
    → Result sent back to Entra ID
    → PTA: password validated on-prem, may be written back to cloud
    → AD FS: SAML response returned

  → IF passwordless (FIDO2/Authenticator/WHFB):
    No password step
    → Proceed directly to MFA/step-up challenge

STEP 6: MFA EVALUATION AND CHALLENGE
  MFA required? → YES
  → Determine user's registered MFA methods
  → Select default method (or let user choose)
  → Send challenge:
    - Authenticator push: notification to registered device
    - FIDO2: cryptographic challenge to key
    - OTP: code generated/sent to registered method
  → User responds
  → Entra ID validates response
    - Push: check approval signal from device
    - FIDO2: verify cryptographic signature
    - OTP: validate code against stored seed
  → MFA success → Continue
  → MFA failure → Fail authentication, log event

STEP 7: CONDITIONAL ACCESS EVALUATION
  During/after authentication, CA evaluates:
  → For each applicable CA policy (matched by user/group/app/condition):
    a. Conditions evaluated:
       - Users/Groups: Is this user in scope?
       - Locations: IP geolocation, device trust, named locations
       - Device platforms: What OS is user on?
       - Device state: Compliant? Hybrid joined? Registered?
       - Client apps: Browser, mobile app, legacy auth?
       - Authentication strength: Does user meet confidence level?
       - Terms of Use: Has user agreed?
    b. Grant controls:
       - Require MFA: Is MFA satisfied? (from Step 6)
       - Require compliant device: Is device compliant?
       - Require hybrid joined: Is device hybrid joined?
       - Require app guarded browser: Is Browser Isolation active?
       - Require authentication strength: Which strength was used?
       - Block access: → BLOCK
       - Grant/modify/session: Apply controls
    c. Result:
       - All conditions met AND grant controls satisfied → Policy SUCCESS
       - Conditions not met → Policy SKIP (move to next policy)
       - Grant controls not satisfied → Authentication fails or modified
  
  → Policy precedence: Policies are evaluated by priority (lower number = higher priority)
  → First policy that matches and has a decision wins

STEP 8: SESSION CONTROL APPLICATION
  If CA policy includes session controls:
  → Persistent browser caching: enabled/disabled
  → Sign-in frequency: every X hours (re-authenticate)
  → Conditional Access Authorization (token modification):
    - Token includes CA claims (modified claims in token)
    - Resource reads these claims and enforces

STEP 9: TOKEN ISSUANCE
  Entra ID generates tokens:
  
  ID Token (JWT):
    - Header: alg (RS256), typ (JWT), kid (key ID)
    - Payload:
      - aud: Application ID (resource for which token is issued)
      - iss: https://sts.windows.net/{tenant-id}/
      - sub: Subject (user's Object ID)
      - oid: User's Object ID
      - tid: Tenant ID
      - preferred_username: UPN
      - name: Display name
      - given_name, family_name
      - iat: Issued at (timestamp)
      - exp: Expiry (typically 1 hour for ID token)
      - nbf: Not before
      - jti: Token ID (unique)
      - groups: Group Object IDs (up to 200, or encoded for >200)
      - amr: Authentication methods reference (how authenticated)
      - acr: Authentication context class reference (CA authentication strength)
    - Signature: RSA signed with Microsoft's private key
  
  Access Token (JWT):
    - For specific resource (e.g., Microsoft Graph, Storage)
    - Payload includes:
      - aud: Resource's App ID URI
      - scp: Scopes (permissions)
      - roles: Application roles (for SP auth)
      - id_token: Contains sub (user or SP Object ID)
    - Lifetime: Typically 60-90 minutes (configurable via CA session controls: 1-96 hours)
    - Signature: RSA signed
  
  Refresh Token:
    - Long-lived (configurable: default 90 days, up to 180 days with notification)
    - Used to get new Access Tokens without re-authentication
    - Session-bound (bound to device/IP in modern setups)
    - Can be revoked (Conditional Access, admin)
    - Contains: OID, TID, scp, roles

STEP 10: TOKEN RETURN
  Tokens returned to app via redirect URI (browser) or direct API response (SP)
  → Browser receives tokens (in response to OAuth flow)
  → App stores tokens:
    - Web app: In server-side session (NOT in client-side storage)
    - SPA: In memory (best practice)
    - Mobile: Secure OS storage
    - SP/daemon: Server-side secure storage (Key Vault recommended)

STEP 11: RESOURCE ACCESS
  App presents Access Token to target resource
  → Resource validates token:
    1. Check signature (via JWKS endpoint)
    2. Check issuer (correct tenant)
    3. Check audience (correct resource)
    4. Check expiry (not expired)
    5. Check subject (who is requesting)
  → Resource checks authorization:
    1. RBAC role assignments (for Azure resources)
    2. App-level permissions (for Microsoft Graph)
    3. Resource-specific ACLs
  → GRANT or DENY

STEP 12: TOKEN REFRESH (when Access Token expires)
  App uses Refresh Token to get new Access Token
  → POST to: https://login.microsoftonline.com/{tenant}/oauth2/token
  → Body: grant_type=refresh_token, client_id, refresh_token, etc.
  → Entra ID validates Refresh Token:
    1. Not revoked
    2. Not expired
    3. Session still valid (CA still allows)
    4. Not used before (replay prevention)
  → New Access Token issued (and possibly new Refresh Token — refresh rotation)

STEP 13: SIGN-OUT
  User clicks sign-out
  → Session cookies cleared
  → Refresh tokens revoked (or flagged)
  → Tokens become invalid when they expire (even before expiry, if revoked)
  → User must re-authenticate for next session
```

---

## 4.5 TOKENS — COMPLETE DEEP DIVE

### 4.5.1 TOKEN TYPES

| Token | Purpose | Lifetime | Audience | Contains |
|-------|---------|----------|----------|----------|
| **ID Token** | Identity of the user (for the app) | ~1 hour | Application's Client ID | User info: oid, upn, name, email, groups, auth methods, CA claims |
| **Access Token** | Access to a resource (API, Azure) | ~60-90 min (configurable) | Resource's App ID URI | Scopes/permissions, roles, user or SP identity, CA claims |
| **Refresh Token** | Get new access tokens silently | 90 days default (up to 180) | N/A (sent only to app) | oid, tid, scopes, roles, session info |

### 4.5.2 TOKEN LIFETIME CONCEPTS

| Concept | Description |
|---------|-------------|
| **Access Token Lifetime** | Time until Access Token expires. Default: 60 min. Can be reduced by Conditional Access session controls (down to 1 hour). Cannot be increased by Conditional Access beyond service default. |
| **Refresh Token Lifetime** | Time until Refresh Token expires. Default: 90 days. With notification flow: up to 180 days. Can be revoked by Conditional Access or admin at any time. |
| **Refresh Token Rotation** | Each time a Refresh Token is used, a new Refresh Token is issued and the old one is invalidated (rotation). If an old Refresh Token is reused, all tokens in that session are revoked. This detects token theft. |
| **Session Lifetime** | How long the user's session lasts before requiring re-authentication. Controlled by CA session controls: Sign-in frequency = 1-96 hours. Default: no limit (until Refresh Token expires). |
| **Token Revocation** | Admin can revoke all tokens for a user at any time. Conditional Access can revoke on risk detection. SP tokens can be revoked by disabling the SP. |

### 4.5.3 ID Token vs Access Token — When to Check Which

| Scenario | Check This Token |
|----------|-----------------|
| "Who is the user?" | ID Token (contains user identity claims) |
| "Can this app access Graph API?" | Access Token (aud = `https://graph.microsoft.com`, check scopes) |
| "Can this SP access Storage?" | Access Token (aud = Storage resource ID, check roles) |
| "What groups is the user in?" | ID Token (groups claim, up to 200) or Graph API |
| "Did CA modify the session?" | Check Access Token for CA claims (amr, acr, and modified claims) |

### 4.5.4 How to Decode a Token

Go to `https://jwt.ms` (or any JWT decoder), paste the token, and examine:
```
Header:
  alg: RS256 (RSA signature)
  kid: Key ID (which Microsoft key signed it)

Payload:
  aud: Who the token is for
  iss: Who issued it (tenant)
  sub: Subject (user SP's Object ID for SP tokens, user's Object ID for user tokens)
  oid: Object ID of the user or SP
  tid: Tenant ID
  uti: Triangle ID (session identifier for token revocation)
  ver: Token version (2.0 for v2 tokens)
  iat: Issued at
  exp: Expiry
  nbf: Not before
  jti: JWT ID

For Access Tokens (resource-specific):
 scp: Scopes (space-separated permissions)
  roles: App roles assigned to SP

For ID Tokens (user-specific):
  preferred_username: UPN
  name: Display name
  given_name: First name
  family_name: Last name
  email: Email
  upn: User Principal Name
  groups: Group Object IDs
  amr: Authentication method references
  acr: Authentication context (CA strength)
```

---

## 4.6 CONDITIONAL ACCESS — COMPLETE DEEP DIVE

### 4.6.1 WHAT IS CONDITIONAL ACCESS

Conditional Access is Azure's **real-time policy engine** that makes access decisions based on conditions evaluated at sign-in time. It is the primary mechanism for:
- Requiring MFA
- Blocking access based on risk, location, or device state
- Constraining session behavior
- Enforcing compliance requirements
- Implementing Zero Trust principles

**CA is NOT:**
- A firewall (it doesn't inspect traffic after authentication)
- A network control (it doesn't block network connections directly)
- An RBAC role (it doesn't grant permissions — it controls WHEN and HOW access is granted)

**CA IS:**
- An identity-centric access control policy engine
- Evaluated at authentication time
- Applied based on signals about user, device, location, and risk

### 4.6.2 CONDITIONAL ACCESS ARCHITECTURE

```
Sign-in Request
│
├─ STEP 1: POLICY MATCHING
│  For each CA policy in the tenant (evaluated by priority):
│  ├─ Users: Is this user in "Include"? NOT in "Exclude"?
│  ├─ Groups: Is any group in "Include"? NOT in "Exclude"? (recursive evaluation)
│  ├─ Roles: Is this directory role in "Include"? NOT in "Exclude"?
│  ├─ Apps: Is this app in "Include"? NOT in "Exclude"?
│  │   (All apps, Microsoft apps, Third-party apps, by app ID)
│  │
│  └─ If ALL conditions met → Policy is "Applicable"
│
├─ STEP 2: CONDITIONS EVALUATION (AND logic — ALL must match for the condition to be true)
│  ├─ Locations:
│  │   ├─ User IP geolocation
│  │   ├─ Is this in "Named Locations" (trusted/untrusted)?
│  │   ├─ Device compliance status (compliant/non-compliant/not detected)
│  │   └─ Trust settings (trusted IPs, MFA requirement)
│  │
│  ├─ Device Platforms:
│  │   ├─ What OS is the device running? (Windows, Mac, iOS, Android, Linux)
│  │   └─ Filter: Include/Exclude specific platforms
│  │
│  ├─ Device State:
│  │   ├─ Is device "Marked as compliant"? (Intune/MDM compliance)
│  │   ├─ Is device "Registered"? (Entra device registration)
│  │   ├─ Is device "Hybrid Joined"? (Azure AD + on-prem AD)
│  │   ├─ Is device "Compliant"? (checks all compliance requirements)
│  │   └─ Note: "Device state" in CA ≠ Intune compliance state.
│  │       CA "device state" is derived from multiple signals,
│  │       while Intune compliance is a separate MDM assessment.
│  │
│  ├─ Client Apps:
│  │   ├─ How did user authenticate? (Browser, legacy auth, modern app, etc.)
│  │   ├─ Which client app is being used?
│  │   └─ "Legacy authentication" = protocols that cannot do MFA
│  │       (IMAP, POP, SMTP, older Office clients, etc.)
│  │
│  ├─ Authentication Strength:
│  │   ├─ Which authentication method was used? (FIDO2, PHS, compliant device + MFA, etc.)
│  │   ├─ Strength levels: High (FIDO2), Medium (compliant device + MFA), Low (MFA only)
│  │   └─ Requirement: Does user's current auth strength meet the required level?
│  │
│  ├─ Risk-Based (Identity Protection):
│  │   ├─ Sign-in risk: Level detected for THIS sign-in attempt
│  │   ├─ User risk: Level detected for THIS user account
│  │   └─ Risk levels: Low, Medium, High
│  │
│  ├─ Terms of Use:
│  │   ├─ Has user accepted Terms of Use app?
│  │   └─ Required for some compliance/legal scenarios
│  │
│  └─ Conditions use AND logic:
│     ALL conditions in a section must be TRUE for the section to match.
│     If ANY condition in a section is FALSE → Policy does not apply.
│
├─ STEP 3: GRANT CONTROLS EVALUATION
│  If policy matches and all conditions are TRUE:
│  ├─ Grant controls specify what must happen:
│  │   ├─ "Require MFA" → Is MFA satisfied?
│  │   ├─ "Require compliant device" → Is device compliant?
│  │   ├─ "Require hybrid joined device" → Is device hybrid joined?
│  │   ├─ "Require authentication strength" → Which strength was used?
│  │   ├─ "Require app guarded browser" → Is Browser Isolation active?
│  │   ├─ "Block access" → Deny immediately
│  │   ├─ "Grant access" → Allow (but may modify session)
│  │   └─ "Require TOTP" / "Require FIDO2" / etc.
│  │
│  └─ All grant controls must be satisfied for GRANT.
│     If ANY grant control is NOT satisfied → DENY or MODIFY.
│
├─ STEP 4: SESSION CONTROLS (if grant is modified/approved)
│  ├─ Persistent browser caching: ON/OFF
│  ├─ Sign-in frequency: X hours (re-authenticate)
│  └─ Conditional Access Authorization:
│     Token claims modified to reflect CA decision
│     Resource reads these claims and enforces session behavior
│
└─ STEP 5: DECISION
   ├─ GRANT: Policy matched, all conditions true, all grant controls satisfied → ACCESS ALLOWED
   ├─ GRANT WITH MODIFICATIONS: Policy matched, grant controls applied but session modified → ACCESS ALLOWED WITH CONTROLS
   ├─ DENY: Grant control = Block → ACCESS DENIED
   └─ SKIP: Policy does not match (conditions false, user excluded) → Evaluate next policy
```

---

### 4.6.3 POLICY PRECEDENCE — CRITICAL CONCEPT

**CA policies are evaluated in priority order (lowest number = highest priority).**

**Decision logic:**
```
1. Policies are evaluated starting from highest priority (lowest number).
2. The FIRST policy that:
   a. Matches the user/group/app conditions AND
   b. Has ALL conditions met AND
   c. Has a decision (Grant/Deny)
   → WINS. Its decision is applied.
3. If a matching policy has a "Deny" decision → DENY, no further evaluation.
4. If a matching policy has a "Grant" decision → GRANT (with controls), stop evaluating.
5. If a matching policy SKIPs (conditions not met) → Move to next priority.
6. If NO policy matches → Access is determined by standard authentication (no CA control).
```

**Critical rule:** **Deny wins immediately.** If a higher-priority policy denies, even if a lower-priority policy would grant, the user is denied.

**Example:**
```
Policy 1 (Priority 1): Block all users in Finance group from accessing any app from untrusted locations → DENY
Policy 2 (Priority 5): Require MFA for all users on all apps → GRANT WITH MFA

Result: Finance user accessing from untrusted location → DENIED (Policy 1 wins)
         Finance user accessing from trusted location → MFA required (Policy 2 wins)
         Non-Finance user anywhere → MFA required (Policy 2 wins)
```

**L3 troubleshooting:**
When a CA policy "appears correct" but doesn't apply:
1. Is there a higher-priority policy skipping or blocking?
2. Is the user/group condition matching correctly?
3. Is the user excluded by another policy's "Exclude" setting?
4. Are the conditions all actually TRUE? (Device state is NOT what you think?)
5. Is the app the user is signing into correctly targeted?

---

### 4.6.4 USERS AND GROUPS CONDITION

**How CA evaluates Users/Groups condition:**
```
Include: User A, Group G1
Exclude: User B, Group G2

Evaluation:
  Is sign-in user in Include list? → YES (User A or member of Group G1)
  Is sign-in user in Exclude list? → YES → Policy SKIP (excluded)
  Is sign-in user in Exclude list? → NO → Policy APPLIES

CRITICAL: Exclude always takes precedence over Include.
If a user is in both Include and Exclude → User is EXCLUDED (policy does NOT apply).
```

**L3 troubleshooting — "User is in the group but CA doesn't apply":**
1. Is the user's group membership dynamic? → Dynamic membership may not be updated yet (refresh delay).
2. Is the user ALSO in an excluded group? → Check exclude settings (often overlooked).
3. Is the user in a nested group that's excluded? → CA evaluates recursively.
4. Is the group's membership stale? → Check Entra ID Connect/Cloud Sync.
5. Is "All users" the Include and specific groups in Exclude? → User may be in Exclude group.

**Service principals and CA:**
By default, CA policies apply to service principals. You can:
- Include/exclude specific service principals
- Use "All users" → includes SPs unless explicitly excluded
- **Break-glass accounts should be explicitly excluded** from CA policies (or managed differently)

---

### 4.6.5 APPLICATIONS CONDITION

**CA can target specific applications:**

| App Type | Description | Example |
|----------|-------------|---------|
| **All cloud apps** | Every app using Entra ID for auth | Catch-all |
| **Microsoft apps** | Microsoft-owned apps (Office, Portal, etc.) | Azure Portal, Exchange Online |
| **Third-party apps** | SaaS apps federated with Entra ID | Salesforce, SAP, custom apps |
| **Specific app by Client ID** | Target by App Registration ID | Custom line-of-business app |

**L3 scenario — "CA policy applies to Exchange but not to Salesforce":**
1. Check the policy's Apps condition — is Salesforce included?
2. Is Salesforce's App Registration configured correctly in Entra ID?
3. Is the user authenticating to Salesforce with an Entra ID token?
4. Is there another policy with higher priority that excludes Salesforce?

---

### 4.6.6 LOCATIONS CONDITION — DEEP DIVE

**Named Locations** are pre-defined trusted/untrusted locations.

**Types:**
| Type | Description |
|------|-------------|
| **Named Location (IP range)** | Define trusted IP ranges (corporate offices, VPN IPs) |
| **Named Location (Country/Region)** | Define trusted countries (for B2B scenarios) |
| **Microsoft-managed (MFA registration)** | Auto-created: MFA registration and Azure Portal access locations |

**Important:**
- Named Locations are **NOT** directly used in CA conditions. Instead, CA uses:
  - **"Require one or more authentication methods"** under Locations condition
  - **"Mark as trusted"** under Locations condition (sets the "isManaged: true" attribute on the IP)
  - **"Require MFA for authenticated users"** under Locations condition
  - **"Require compliant or hybrid joined device"** under Locations condition (requires device compliance AND a trusted location)

**L3 troubleshooting — "User in office should be trusted but CA still requires MFA":**
1. Is the user's public IP actually in the Named Location's IP range? (check CIDR)
2. Is the user behind a different public IP than expected (VPN misconfiguration, roaming device)?
3. Is "Mark as trusted" enabled in the location AND the CA policy? (both must be configured)
4. Is the Named Location enabled? (sometimes disabled)
5. Is the user's device NOT compliant? (some CA location conditions require compliance + trusted location)
6. Is the user using a different authentication method that bypasses location-based MFA? (CA considers authentication strength, not just location)

---

### 4.6.7 DEVICE STATE CONDITION — CRITICAL CONFUSION POINT

**This is where the most confusion occurs. CA has multiple device-related conditions:**

| Condition | What It Checks | How It's Determined |
|-----------|---------------|---------------------|
| **Device Platform** | What OS the device is using (Windows, macOS, iOS, Android, Linux) | Detected from browser UA, device registration info, or client app info |
| **Marked as compliant** | Is device marked as "compliant" in Entra ID? | Based on device registration and compliance status reported by Intune/MDM |
| **Is registered** | Is device registered with Entra ID? | Device registration (Android, iOS, Windows) creates an Entra device object |
| **Hybrid joined** | Is device joined to both on-prem AD AND Entra ID? | Device is domain-joined + Entra-registered (via Entra Connect/Cloud Sync) |
| **Is compliant** | Comprehensive compliance check (updates dynamically) | Similar to "Marked as compliant" but includes dynamic compliance checks |

**Why this matters for CA:**
- A CA condition might say: "If device is NOT compliant → require MFA"
- But what does "compliant" mean? It depends on Intune/MDM configuration AND Entra device registration.
- A Windows device that is domain-joined (hybrid joined) but NOT Intune-managed may show as "Not compliant" in CA even though it's managed by on-prem SCCM/Group Policy.

**L3 troubleshooting — "Compliant device still triggers CA MFA requirement":**
1. Check the device's compliance state in Endpoint Manager (Intune).
2. Check device registration in Entra ID (Portal → Devices).
3. Check Conditional Access → Device state — does the specific device show as "Compliant"?
4. **CA Device State and Intune Compliance are NOT always synchronized in real-time.** There can be delays (minutes to hours).
5. Check if the device is hybrid joined vs Entra registered vs compliant — these are different states.
6. CA Device State is derived from: Intune compliance + device registration. If either is false, device state = Not compliant.
7. CA uses signal from the sign-in request. If the device info is not sent (e.g., browser doesn't support device signal), device state = "Not detected" → CA treats as NOT compliant.

---

### 4.6.8 CLIENT APPS CONDITION

**How CA evaluates Client Apps:**
```
CA checks HOW the user is authenticating:

Modern authentication:
  → Uses OAuth 2.0/OpenID Connect
  → Supports MFA
  → Examples: Modern Office apps (Office 365 CLI, Office 2019+), browser, Entra portal, Microsoft Teams (new), Outlook (new)

Legacy authentication:
  → Uses older protocols that CANNOT do MFA
  → Examples: IMAP, POP3, SMTP, Active Directory Protocol, Office 2010/2013 (without updates), older Outlook (2013 without updates), XAML-based apps, older PowerShell cmdlets

Browser:
  → Can support modern auth (depends on browser)
  → Legacy auth in browser = older web interfaces

Mobile apps:
  → Outlook mobile, OneDrive mobile, etc.
  → May use modern or legacy depending on version
```

**CA Client App conditions:**
| Setting | Description |
|---------|-------------|
| **Any client app** | All authentication methods |
| **Modern authentication** | Only modern (OAuth 2.0) apps |
| **All client apps except...** | Exclude specific types |
| **Microsoft Exchange ActiveSync** | Legacy Exchange protocol |

**L3 critical: Legacy authentication ALWAYS bypasses Conditional Access unless explicitly targeted.**

**Why?** Legacy auth protocols (IMAP, POP, SMTP) cannot handle MFA challenges. CA evaluates at sign-in time, and if the protocol can't do MFA, CA can't enforce it. Microsoft recommends:
1. Use CA to **Block** legacy authentication (entirely, or for risky users/locations)
2. Or **Monitor** (Report Only) legacy auth to identify what's using it
3. Migrate apps to modern authentication

**Current status (Sept 2026):**
- Microsoft is actively pushing legacy authentication deprecation.
- Conditional Access for legacy auth is a key Zero Trust requirement.
- Many legacy protocols have been updated to support modern auth (Outlook, PowerShell, etc.).

---

### 4.6.9 AUTHENTICATION STRENGTH CONDITION

**What it is:**
Authentication Strength is a CA feature that categorizes authentication methods into confidence levels:

| Strength Level | Authentication Methods Included | Confidence |
|---------------|-------------------------------|------------|
| **High** | FIDO2 key, Windows Hello for Business (compliant device), Certificate-based auth | Highest — phishing-resistant |
| **Medium** | MFA via Authenticator/FIDO2 on compliant/hybrid joined device | Good — requires device + MFA |
| **Low** | MFA (password + Authenticator push, OTP, SMS) | Basic — MFA only, no device requirement |
| **Unknown** | Password only, or method not categorized | No additional assurance |

**How it works in CA:**
```
CA Policy condition:
  → Authentication Strength requirement: "High"
  
Sign-in: User authenticates with password + Authenticator push
  → Authentication Strength result: "Low" (MFA without compliant device)
  → Strength requirement NOT met → Policy DENY (or skip)

Sign-in: User authenticates with FIDO2 key
  → Authentication Strength result: "High"
  → Strength requirement MET → Policy GRANT

Sign-in: User authenticates with password + Authenticator on compliant device
  → Authentication Strength result: "Medium"
  → Strength requirement "Medium" → MET → Policy GRANT
  → Strength requirement "High" → NOT MET → Policy DENY
```

**L3 troubleshooting — "Authentication strength not matching expected":**
1. Check which authentication method the user actually used (sign-in log → Authentication Details).
2. Check the authentication strength mapping in CA settings.
3. If the user is on a compliant device with MFA → should be "Medium."
4. If the user is NOT on a compliant device with MFA → only "Low."
5. If the user used FIDO2 → should be "High."
6. If the strength is "Unknown" → something is wrong with the method detection.

**Common mistake:**
Engineers expect that "MFA required" in a CA policy means "High" authentication strength. It doesn't. "Require MFA" maps to "Low" strength. If you want "Medium" or "High," you must explicitly select "Require authentication strength" and pick the level.

---

### 4.6.10 GRANT CONTROLS — COMPLETE LIST

| Grant Control | What It Does | L3 Notes |
|--------------|-------------|----------|
| **Require MFA** | User must authenticate with MFA | If already done → passes. If not → challenge or block. |
| **Require compliant device** | Device must be Intune/MDM compliant | Device state must show "compliant" in CA evaluation. |
| **Require hybrid joined device** | Device must be hybrid joined (AD + Entra) | Requires Entra Connect/Cloud Sync. |
| **Require authentication strength** | Specific auth method strength (Low/Medium/High) | Most powerful — forces specific methods. |
| **Require app guarded browser** | Browser must use Microsoft Edge Application Guard | Isolated browser for untrusted locations. |
| **Block access** | Deny access immediately | Highest priority — cannot be overridden by lower-priority policies. |
| **Grant access** | Allow access (no modification) | Default — no additional controls. |
| **Require TOTP** | Time-based OTP required | Specific MFA method requirement. |
| **Require FIDO2** | FIDO2 key required | Phishing-resistant. |

**Multiple grant controls — AND logic:**
If a CA policy has multiple grant controls (e.g., "Require MFA" AND "Require compliant device"), ALL must be satisfied for the policy to grant.

---

### 4.6.11 SESSION CONTROLS

| Control | Description | Configuration |
|---------|-------------|---------------|
| **Sign-in frequency** | How often user must re-authenticate | 1-96 hours. After expiry, user must re-do CA evaluation. |
| **Persistent browser caching** | Whether browser session persists after closing browser | On/Off. If Off, user must re-authenticate each session. |
| **Conditional Access Authorization** | Modifies token claims to reflect CA state. Resources read these claims. | On/Off. Required for resources that enforce CA session controls. |

**L3 operational impact:**
- Sign-in frequency controls how often CA is re-evaluated. If set to 4 hours, the user gets a new token every 4 hours (not just every 60 min for access token lifetime).
- Without Conditional Access Authorization enabled, session controls may not work for all resources.
- Session controls are evaluated at resource access time, not just sign-in time.

---

### 4.6.12 RISK-BASED ACCESS (IDENTITY PROTECTION)

**Risk Types:**
| Risk Type | What It Measures | Levels |
|-----------|-----------------|--------|
| **Sign-in risk** | Risk associated with THIS sign-in attempt | None, Low, Medium, High |
| **User risk** | Risk associated with THIS user account | None, Low, Medium, High |
| **Risky users** | Users that Identity Protection flags for review | In a review queue |
| **Risky sign-ins** | Individual sign-in attempts flagged as risky | In a review queue |

**Sign-in Risk:**
- Detected by analyzing patterns:
  - Impossible travel (user signed in from two locations too far apart in too little time)
  - Anonymous IP address (Tor, proxy)
  - IP address anonymization
  - Malware linked IP
  - Azure AD tested credentials (leaked password database)
  - Unfamiliar sign-in patterns
  - Brute force attacks

**User Risk:**
- Detected by analyzing:
  - Leaked credentials (passwords found in public dumps)
  - Malware infections (sign-ins from malware-associated IPs)
  - Compromised credentials (checked against known compromises)

**CA Risk-based controls:**
```
Policy: If Sign-in Risk = Medium → Require MFA
Policy: If Sign-in Risk = High → Block access
Policy: If User Risk = Medium → Require MFA
Policy: If User Risk = High → Block access
```

**L3 troubleshooting — "Sign-in risk not triggering CA":**
1. Is the Identity Protection policy actually enabled for that risk level? (Risk-based CA must be configured separately from risk detection)
2. Did the sign-in actually trigger a risk detection? (Check Risk Detections, not just sign-in log riskDetail)
3. Is the risk level high enough to trigger the CA policy threshold? (if policy triggers at High, Medium won't trigger it)
4. Is the user excluded from the CA policy?
5. Has the user already remediated the risk? (User clicked "Dismiss" on risk detection — risk level may change but session may still be active)
6. Is Identity Protection license enabled? (Premium P2 required for user risk)

---

### 4.6.13 NAMED LOCATIONS

**Purpose:** Define trusted and untrusted physical locations (IP ranges, countries) used by CA conditions.

**Configuration:**
| Setting | Description |
|---------|-------------|
| **Name** | Friendly name (e.g., "HQ Office," "VPN") |
| **IP ranges** | List of CIDR ranges (IPv4, IPv6) |
| **Country/Region** | List of countries |
| **Outlook Options** | MFA registration, Azure Portal access (Microsoft-managed) |
| **State** | Enabled/Disabled |

**How Named Locations are used in CA:**

Under the **Locations** condition, there are two parts:
1. **"Mark as trusted" (require MFA from locations not marked as trusted)** — When user is in the Named Location's IP range, their sign-in is "marked as trusted" (the `isManaged: true` attribute is set).
2. **Location conditions in the CA policy itself** — Determine what happens when user IS or ISN'T in the Named Location:
   - "Require one or more authentication methods" (from this location)
   - "Require MFA for authenticated users" (from this location)
   - "Require compliant or hybrid joined device" (from this location)
   - "Require TOTP" etc.

**L3 troubleshooting — Named Location not working:**
1. **Location is Disabled** → Check state.
2. **IP range doesn't match user's actual IP** → User's public IP may differ (NAT, VPN, roaming).
3. **"Mark as trusted" is not enabled in both places** → Named Location must have "Mark as trusted" enabled, AND the CA policy must include the Locations condition with "Mark as trusted."
4. **Named Location uses IPv6, user on IPv4 (or vice versa)** → Both must be covered.
5. **User is on VPN, VPN IP not in Named Location** → VPN terminates at different IP.
6. **User is on roaming device, IP is from different location** → Travel triggers untrusted location.
7. **Microsoft-managed Named Locations cannot be modified** → MFA registration and Azure Portal are handled automatically by Microsoft.

---

### 4.6.14 EXCLUSIONS — CRITICAL FOR TROUBLESHOOTING

**CA exclusions are the #1 reason a policy "doesn't work."**

**Where exclusions can be set:**
In EVERY CA policy, under each condition (Users/Groups, Locations, Apps, etc.), there is an **"Exclude"** section.

**Default exclusion behavior:**
```
If ANY "Exclude" condition matches → Policy DOES NOT APPLY to that entity.
Exclude takes precedence over Include.
```

**Common exclusion pitfalls:**
| Scenario | Why It Happens |
|----------|---------------|
| Service account bypassing MFA | SP was in "Exclude" under Users/Groups (by default, or manually added) |
| Break-glass account bypassing CA | Break-glass account is excluded from CA policies (should be, by design) |
| "All users" policy not applying to some users | Those users are in an excluded group (often accidentally) |
| Guest user not subject to CA | Guests may be excluded by default depending on configuration |
| Legacy auth not blocked | Legacy auth client apps are sometimes excluded from CA policies |
| Device compliance requirement not applying | Device state = "Not detected" (excluded from device condition evaluation) |

**L3 troubleshooting checklist — "Why CA isn't applying":**
1. Is the user/group explicitly EXCLUDED? (Check all conditions' Exclude tabs)
2. Is the user in a group that is excluded? (Check nested groups)
3. Is there a HIGHER PRIORITY policy that matches and skips? (Precedence issue)
4. Is there a higher priority policy that DENIES? (Another policy wins before this one evaluates)
5. Is the user in the right application condition? (Wrong app targeted)
6. Are ALL conditions met? (Check device state, location, platform, client app individually)
7. Is the Authentication Strength requirement met? (User's actual strength vs required)
8. Is the grant control actually satisfiable? (e.g., "Require compliant device" but no devices are compliant)

---

### 4.6.15 TERMS OF USE

**What it is:**
An app registration that users must explicitly accept before accessing certain applications. Used for legal/compliance scenarios (e.g., acceptable use policy).

**Flow:**
```
1. Admin creates Terms of Use app registration
2. CA policy references it under Terms of Use condition
3. User signs in → CA evaluates → Terms of Use required
4. User sees Terms of Use screen → Must click "I agree"
5. Acceptance recorded in audit log
6. User proceeds to application
7. If user declines → Access blocked
```

**L3 relevance:** Rarely the cause of major incidents, but can block user access unexpectedly if Terms of Use app is misconfigured or users are incorrectly excluded.

---

### 4.6.16 BREAK-GOSS ACCOUNTS — L3 CRITICAL

**What they are:**
Emergency accounts that bypass Conditional Access policies to allow administrators to access the environment during a CA-related incident.

**Best practices:**
| Practice | Description |
|----------|-------------|
| **Dedicated accounts** | Use separate accounts (e.g., `breakglass-admin01@contoso.com`) NOT in normal user groups |
| **Excluded from CA** | Explicitly excluded from ALL CA policies (Users/Groups exclude tab) |
| **Separate tenant (recommended)** | In a different tenant (B2B) for maximum isolation |
| **PIM-eligible (alternative)** | Use PIM just-in-time elevation instead of dedicated break-glass accounts |
| **Strong authentication** | FIDO2 key + long complex password (never compromised) |
| **Limited scope** | Can only be used for emergency CA bypass recovery, not daily admin |
| **Documented procedure** | Step-by-step instructions for using break-glass in an incident |
| **Regular review** | Verify break-glass accounts are still functional and not compromised |

**L3 troubleshooting:**
- During a CA outage, break-glass accounts are the ONLY way in if CA blocks all other access.
- If break-glass accounts don't work → Complete emergency (requires Microsoft support).
- Verify break-glass functionality quarterly in DR drills.

---

### 4.6.17 SERVICE ACCOUNTS AND CA

**Critical behavior:**
```
By default, Conditional Access policies apply to ALL authentication, including service principals.

A CA policy with "Require MFA" targeting "All cloud apps":
  → User sign-in: MFA challenged ✓
  → Service principal sign-in: MFA is NOT possible (no human to challenge)
  → Result: Service principal may be BLOCKED by "Require MFA" CA policy!

This is a common production outage cause:
  → Engineer creates CA policy: "Require MFA for all users"
  → Service principal in production app suddenly cannot authenticate
  → Application returns 403/Authorization errors
  → Root cause: CA policy blocking SP authentication
```

**Solutions:**
1. **Exclude service principals** from MFA-required CA policies:
   - Under Users/Groups → Exclude → Select "All service principals" (or specific SPs)
2. **Use different CA policies** for SPs vs users (separate policies with different conditions).
3. **For high-security SPs**: Use Privileged Identity Management (PIM) for just-in-time activation instead of CA-based MFA.

**L3 interview question:**
Q: "A CA policy requiring MFA was deployed. Why did a background service stop working?"
A: "The CA policy applies to all authentication, including service principals. Service principals cannot perform MFA (there's no human to respond to a challenge). The SP needs to be excluded from the MFA-required CA policy, or a separate CA policy should be created for SPs that uses different authentication strength requirements."

---

### 4.6.18 LEGACY AUTHENTICATION AND CA

**Legacy authentication protocols:**
| Protocol | Description | MFA Capable? |
|----------|-------------|-------------|
| IMAP | Email retrieval | No |
| POP3 | Email retrieval | No |
| SMTP | Email sending | No |
| Active Directory Protocol (ADAL) | Directory access | No |
| XAML-based apps | Older WPF/WinForms apps using ADAL | No |
| Office 2013 (unupdated) | Older Office versions | No |
| Older Outlook (2013, 2016 without updates) | Email client | No (unless updated) |

**CA interaction with legacy auth:**
```
Legacy authentication CANNOT send MFA tokens.
CA evaluates at sign-in time:
  → Legacy auth request arrives
  → CA checks Client Apps condition: "Legacy authentication" is detected
  → CA checks grant controls: "Require MFA" → BUT MFA is impossible
  → Decision: If "Require MFA" is the grant control, legacy auth is BLOCKED
  → If "Require MFA" is NOT in grant controls, legacy auth is allowed

IMPORTANT: Conditional Access does NOT apply to protocols that use legacy authentication UNLESS explicitly targeted with a Client Apps condition.

This means:
  A CA policy without a Client Apps condition → Does NOT evaluate for legacy auth requests → Legacy auth PASSES THROUGH CA
  A CA policy WITH "Any client app" (or "Legacy authentication") condition → Legacy auth IS evaluated and can be BLOCKED
```

**L3 troubleshooting — "CA policy not blocking legacy auth":**
1. Check CA policy's Client Apps condition — is it set to "Any client app" (which includes legacy)?
2. If not set, the legacy auth request doesn't match the policy → not evaluated.
3. Solution: Update CA policy to include "Any client app" or specifically target legacy auth.

---

## 4.7 MULTIPLE POLICIES — INTERACTION AND CONFLICTS

### 4.7.1 How Multiple CA Policies Interact

```
Tenant has 10 CA policies. User signs in.

1. All policies are evaluated by priority (priority 1 first, then 2, etc.).
2. For each policy:
   a. Does user match Users/Groups condition?
   b. Are all conditions met? (AND logic within each condition)
   c. If yes → Evaluate grant controls
   d. If grant controls satisfied → Policy decision (Grant/Deny)
   e. If grant controls NOT satisfied → Policy decision (Deny/Block)
3. First policy with a definitive decision (Grant or Deny) → WINS
4. All remaining policies are NOT evaluated
5. If no policy makes a decision → No CA control applied (standard authentication)
```

### 4.7.2 Common Multiple Policy Problems

| Problem | Cause | Solution |
|---------|-------|----------|
| **Policy doesn't apply, even though it's correct** | Higher-priority policy has "Exclude" covering this user | Check all higher-priority policies for exclusions |
| **Unexpected MFA required** | Multiple policies require MFA; one is unexpected | Check policy priority, conditions, and grant controls for all policies |
| **User can access risky resource** | Lower-priority policy grants access before higher-priority policy denies | Reorder priorities; Deny policies should be highest priority |
| **CA configuration never takes effect** | Policy is in Report Only mode | Check policy mode (Report Only vs Enforced) |
| **Different behavior for different users** | Different CA policy conditions match different users | Review each user's applicable policies (use Sign-in log → Applied Conditional Access) |
| **Guest access policy conflicts** | Guest-specific policy and general policy both match | Check exclusions and priority; ensure guest policies are higher priority or more specific |

### 4.7.3 Debugging Multiple CA Policies

**Use the Sign-in Log:**
```
Sign-in log → ConditionalAccessStatus field:
  - NotApplied: No CA policy matched
  - Failed: CA policy matched but couldn't be enforced (error)
  - Success: CA policy matched and was successfully applied (no additional controls)
  - Applied: CA policy matched and additional controls were applied (MFA, session controls, etc.)

Sign-in log → Applied Conditional Access section:
  Lists EVERY policy that was evaluated, with:
  - Policy name
  - Policy ID
  - Decision (Report Only, Success, Failure)
  - Enforced vs Report Only
  - Grant controls applied
  - Conditions that were checked
```

---

## 4.8 ADMINISTRATION — CONDITIONAL ACCESS

### Key Administrative Operations

| Operation | Tool | Scope |
|-----------|------|-------|
| Create/manage CA policies | Portal, CLI | Tenant |
| Enable/disable CA policies | Portal | Per policy |
| Set policy mode (Enforced/Report Only) | Portal | Per policy |
| Assign users/groups to CA conditions | Portal, CLI | Per policy |
| Define Named Locations | Portal | Tenant |
| Configure authentication strengths | Portal, CLI | Tenant |
| Configure authentication methods | Portal, CLI | Tenant |
| Manage Terms of Use | Portal, CLI | Tenant |
| Enable/disable legacy auth | Portal, CLI, Exchange admin | Tenant |
| Audit CA decisions | Sign-in logs, Audit logs | Tenant |
| Test CA policy | Sign-in log analysis, What-If tools | Tenant |
| Monitor CA compliance | Secure Score, CA compliance dashboard | Tenant |
| CA policy naming/versioning | Naming convention, documentation | Tenant governance |

---

## 4.9 SECURITY CONSIDERATIONS — AUTHENTICATION & CA

| Concern | Risk | Mitigation |
|---------|------|------------|
| **MFA fatigue / push bombing** | User overwhelmed with push notifications → eventually approves one | Rate limiting, number matching (Authenticator shows number to enter), FIDO2 preferred |
| **Phishing-resistant authentication** | Passwords and push notifications can be phished | FIDO2, Windows Hello, certificate-based auth as primary |
| **MFA bypass via legacy auth** | Legacy protocols can't do MFA → CA can't enforce | Block legacy auth via CA or disable protocols |
| **CA Report Only mode accidentally left** | Policies look enforced but only log | Regular review of all CA policy modes |
| **Break-glass account compromise** | If compromised, attacker has full access | Strong auth, separate tenant, regular rotation, monitoring |
| **Service principal bypass** | SPs can bypass MFA-required CA by default | Explicit exclusion or separate SP-focused policies |
| **Exclusion drift** | Over time, users/groups added to exclusions without review | Regular CA audit, naming conventions, change management |
| **Token theft with long session lifetime** | Stolen token valid for long time | Short session frequencies (1-4 hours), CA session controls |
| **Authentication strength downgrade** | User authenticates with weak method despite strong requirement | CA Authentication Strength condition, correct mapping |
| **Consent phishing** | Users consent to malicious apps | Block user consent, admin consent workflow, app consent policies |
| **Password spray** | Attacker tries many passwords against many accounts | Account lockout, alerting, MFA, passwordless |
| **SIM swapping (SMS-based MFA)** | Attacker transfers phone number, intercepts SMS | Move away from SMS, use Authenticator/FIDO2 |
| **Conditional Access DDoS** | Too many CA policy evaluations overload service | Not typically an issue, but large tenant with many policies can experience delays |
| **Stale CA policies** | Old policies not reviewed, conflicting with current needs | Regular policy audits, deprecation of unused policies |
| **Token replay** | Stolen token reused from different device/IP | Refresh token rotation, session controls, device binding |

---

## 4.10 MONITORING — AUTHENTICATION & CA

### Key Monitoring Targets

| Metric/Log | What It Shows | Alert Trigger |
|------------|---------------|---------------|
| **Sign-in failure rate** | Total failed sign-ins over time | Sudden increase (attack, lockout storm) |
| **Sign-in success rate by method** | Password, MFA, passwordless, FIDO2 | Decrease in MFA/passwordless usage (users bypassing) |
| **MFA challenge rate** | How often MFA is challenged | Unexpected increase (CA change, attack) or decrease (CA not applying) |
| **MFA failure rate** | MFA challenges that fail | Increase → users not registering, or attacker trying to bypass |
| **Legacy auth usage** | Sign-ins using legacy protocols | Any usage in a Zero Trust environment = alert |
| **CA policy evaluation count** | How many times each CA policy is evaluated | Sudden drop (policy disabled?) or spike (possible attack) |
| **CA Report Only policies** | List of policies in Report Only | Should be zero in production (or justified) |
| **Risk detections** | Sign-in risk, user risk, risky users | Elevated risk levels, unresolved risky users |
| **SSPR usage** | Password reset events via SSPR | Excessive resets (helpdesk issue), suspicious resets (account compromise) |
| **Account lockouts** | Locked accounts | Multiple lockouts → password issue or brute force |
| **Authentication method registration** | Users registering MFA methods | Decline in registrations → users avoiding MFA |
| **Token issuance rate** | How many tokens issued per user/app | Abnormal volumes → compromise or misconfiguration |
| **SP sign-in patterns** | Service principal authentication patterns | Unusual times, volumes, locations → SP compromise |
| **Guest invitation/acceptance rate** | B2B collaboration events | Unusual patterns → guest account compromise |
| **Named Location mismatches** | Users accessing from unexpected locations | Impossible travel, unexpected countries |

---

## 4.11 PRODUCTION EXAMPLE

**Scenario: Enterprise Zero Trust deployment with CA as primary access control.**

```
Tenant: contoso.onmicrosoft.com (Premium P2)

Named Locations:
  "HQ-Office": 203.0.113.0/24, 198.51.100.0/24 (corporate IPs)
  "VPN-Corporate": 192.0.2.0/24 (VPN pool)
  "Break-Glass-IP": 203.0.113.100/32 (specific management jump box)

Authentication Methods:
  Enabled: FIDO2, Microsoft Authenticator (push + TOTP + passwordless), Windows Hello
  Disabled: SMS, Voice
  SSPR: Enabled, requires registration, Authenticator + FIDO2

CA Policies (by priority):

  Priority 1: "Block-Unlikely-Breach"
    Users: All users, all groups
    Apps: All cloud apps
    Conditions:
      - Sign-in risk = Medium or High → Block
      - User risk = High → Block
      - Client app = Legacy authentication → Block
    Grant: Block access
    Mode: Enforced

  Priority 5: "Require-High-Strength-For-Sensitive-Apps"
    Users: All users
    Groups: Exclude "Break-Glass-Admins"
    Apps: Exchange Online, SharePoint Online, Admin centers
    Conditions:
      - Authentication Strength ≠ High
      - OR Location not in "HQ-Office" or "VPN-Corporate"
      - OR Device state NOT compliant
    Grant: Require authentication strength = High
    Session: Sign-in frequency = 4 hours
    Mode: Enforced

  Priority 10: "Require-MFA-For-All"
    Users: All users, Exclude "Break-Glass-Admins"
    Apps: All cloud apps
    Locations: Mark as trusted from "HQ-Office" and "VPN-Corporate"
    Conditions:
      - Locations: User NOT in trusted location
      - OR Device state NOT compliant
      - OR Client app = Legacy authentication (already blocked by Priority 1)
    Grant: Require MFA
    Session: Sign-in frequency = 8 hours
    Mode: Enforced

  Priority 15: "Block-Unmanaged-Devices"
    Users: All users
    Apps: All cloud apps
    Conditions:
      - Device state = Not compliant OR Not registered
    Grant: Require compliant or hybrid joined device
    Session: Sign-in frequency = 4 hours
    Mode: Enforced

Break-Glass:
  Account: breakglass-admin01@contoso.com (excluded from ALL CA policies)
  Auth: FIDO2 key + 20-character password stored in safe
  Location: Access only from "Break-Glass-IP" Named Location

Monitoring:
  Alert on: Any CA policy failure, Legacy auth usage, Sign-in risk triggers,
            Unsuccessful MFA attempts > 3, Break-glass account usage
```

---

## 4.12 FAILURE SCENARIOS — AUTHENTICATION & CA

| Scenario | Root Cause | Resolution | Evidence |
|----------|-----------|------------|----------|
| **User cannot sign in — "MFA required but no methods registered"** | User hasn't registered any MFA methods, CA requires MFA | User must register MFA method (admin-assisted), then CA passes | Sign-in log: MFA requirement, no methods found in profile |
| **User gets MFA push but cannot approve** | Device offline, app not installed, device not trusted | Re-register device, check network, use alternate method (FIDO2, TOTP) | Sign-in log: Authentication Details → push notification failure |
| **CA policy not applying to user** | User is excluded (directly or via group), higher-priority policy wins, conditions not all met | Check all CA policies, exclusions, and conditions via sign-in log's Applied Conditional Access | Sign-in log → ConditionalAccessStatus = NotApplied or Success (not Applied) |
| **Service principal blocked by CA requiring MFA** | CA policy applies to SPs, SPs can't do MFA | Exclude service principals from MFA-required policy, or use different policy | Sign-in log: SP sign-in, CA = Applied, error = MFA required but not possible |
| **Legacy auth bypassing CA** | Client Apps condition not set to include legacy authentication | Add "Legacy authentication" to Client Apps condition in CA policy | Sign-in log: Authentication Requirement = Legacy, CA = NotApplied |
| **"Require compliant device" doesn't trigger"** | Device state is "Not detected" (not "Not compliant") in CA, or device compliance changes haven't synced | Check Intune compliance state, CA device state (can take time to sync), ensure device registration is complete | Sign-in log: Device state = Not detected, Intune portal: Device compliant |
| **Risk-based CA not blocking risky sign-ins** | Risk-based CA not configured, risk level below threshold, or user excluded | Configure risk-based CA policies, verify thresholds, check exclusions | Identity Protection: Risk detections exist, CA: No risk-based policy matching |
| **"Authentication Strength" not matching expected level** | Wrong auth method used, or strength mapping incorrect | Check Authentication Details in sign-in log, verify authentication strength configuration | Sign-in log: Auth Method = TOTP (should be FIDO2 for High) |
| **User cannot sign in from office despite being in Named Location** | Named Location disabled, IP range mismatch, "Mark as trusted" not enabled, CA location condition misconfigured | Check Named Location state, IP coverage, CA Locations condition, and "Mark as trusted" settings | Sign-in log: Location = corporate IP, Named Location = Disabled, CA: Locations condition = Not met |
| **Multiple CA policies creating unexpected behavior** | Policy precedence wrong, overlapping conditions, exclusions in unexpected places | Analyze sign-in log Applied Conditional Access section, sort policies by priority, identify conflicting ones | Sign-in log: Multiple policies listed with decisions |
| **Guest user cannot access resource after invitation accepted** | CA policy blocks guests, guest device not compliant, guest RBAC insufficient, guest not in correct group | Check CA policies for guest conditions, guest group memberships, guest RBAC assignments | Sign-in log: UserType = Guest, CA = Applied (block), RBAC = None |
| **MFA fatigue — user accidentally approves malicious sign-in** | Push bombing attack, user tired of notifications | Enable number matching in Authenticator, switch to FIDO2, alert on rapid MFA prompts, monitor MFA request patterns | Sign-in log: Multiple MFA challenges in short time, user reports suspicious |
| **CA policy in Report Only mode still appears as "Applied"** | Policy mode was changed from Enforced to Report Only, but logs still show evaluation | Check policy mode in CA settings, sign-in log reports it as "Report Only" | CA dashboard: Policy mode = Report Only |

---

## 4.13 TROUBLESHOOTING METHODOLOGY — AUTHENTICATION & CA

### Authentication Failure Troubleshooting

```
Step 1: Identify the sign-in failure
  → Who? (which user/SP?)
  → When? (timestamp)
  → From where? (IP, device, location)
  → Which application?
  → Interactive or non-interactive?

Step 2: Check sign-in log (FIRST AND MOST IMPORTANT)
  → Status: Success or Failure
  → StatusDetail: Specific error code
  → ConditionalAccessStatus: NotApplied / Failed / Success / Applied
  → AuthenticationRequirement: What was required?
  → ConditionalAccessOrganizations: Which CA policies evaluated?
  → Risk Detail: Any risk detected?
  → Authentication Details: Which auth method used/failed?
  → Device Detail: Device state, compliance

Step 3: Categorize the failure
  → Authentication failure: Wrong password, account disabled, MFA failure
  → CA failure: Policy blocks, grant controls not met
  → Authorization failure: Token valid but RBAC/Policy denies resource access

Step 4: Authentication failure deep dive
  → Account enabled? (AccountEnabled = true)
  → Account blocked? (isBlocked = true)
  → Password valid? (local or federated)
  → MFA methods registered? (user's security info)
  → MFA challenge completed? (sign-in log authentication details)
  → SSPR issues? (user registered for SSPR?)
  → Federation issues? (if hybrid — PTA/AD FS connectivity)

Step 5: CA failure deep dive
  → Check Applied Conditional Access in sign-in log
  → For each policy:
    a. Is user included/excluded?
    b. Are all conditions met? (check each condition individually)
    c. Are grant controls satisfiable?
    d. Is policy in Report Only mode?
  → Check policy precedence — which policy wins?
  → Check for exclusions (user might be in an excluded group)
  → Check service principals — SPs might be blocked by MFA-required policies

Step 6: Authorization failure deep dive (if authentication succeeded but access denied)
  → Token valid? (check jwt.ms)
  → Token for correct resource? (aud claim)
  → Token not expired? (exp claim)
  → RBAC role assignment? (check at resource/RG/subscription scope)
  → Azure Policy? (could be a Policy block, not RBAC)
  → Resource-specific access controls? (Storage firewall, NSG, etc.)

Step 7: Fix and validate
  → Address root cause
  → Test sign-in/access
  → Check sign-in log for success
  → Monitor for recurrence
```

### CA Policy Not Applying — Detailed Checklist

```
□ Is the policy enabled? (not disabled)
□ Is the policy in "Enforced" mode? (not Report Only)
□ Does the user match Include users/groups?
□ Is the user NOT in Exclude users/groups? (directly or via nested groups)
□ Are ALL conditions met?
  □ Locations: User's IP in expected range? Named Location enabled?
  □ Device Platform: User's OS matches include criteria?
  □ Device State: Device shows correct state in CA? (Not just Intune — CA device state)
  □ Client Apps: Authentication method matches condition? (Legacy vs Modern)
  □ Authentication Strength: User's method meets required strength?
  □ Risk: Risk level meets threshold?
□ Is there a higher-priority policy that matches and takes precedence?
□ Is there a higher-priority policy with a Deny decision?
□ Does the app match the Applications condition?
□ For service principals: Are they included or excluded?
□ Are Named Locations enabled and correct?
□ Has the user recently joined/left a group? (group membership propagation delay)
□ Check the sign-in log → Applied Conditional Access section for EVERY policy evaluated
```

---

## 4.14 LOGS / EVIDENCE

| Evidence Source | What It Shows | Access Method |
|----------------|---------------|---------------|
| **Sign-in logs** | All authentication events with detailed status, CA evaluation, device info, location, risk, auth method | Portal → Entra ID → Monitoring → Sign-in logs; Graph API; Log Analytics |
| **Audit logs** | Administrative changes to CA policies, authentication methods, users, groups, roles | Portal → Entra ID → Monitoring → Audit logs; Graph API |
| **Non-interactive sign-in logs** | SP/daemon authentication events, token-only requests | Sign-in logs filtered by `IsInteractive = false` |
| **Conditional Access compliance dashboard** | Overall CA enforcement status, compliance scores, policy health | Portal → Security Center → Conditional Access compliance |
| **Risk detections** | Identity Protection risk items: user risk, sign-in risk, risky users | Portal → Entra ID → Identity Protection → Risk detections |
| **Token decode (jwt.ms)** | Actual token contents — claims, audience, issuer, expiry, group claims, CA claims | Paste token at jwt.ms |
| **Activity Log (Azure)** | Azure control plane operations with identity context | Portal → Subscription/RG → Activity Log |
| **Authentication Methods policy** | Current tenant authentication configuration | Portal → Entra ID → Security → Authentication Methods |
| **CA policy evaluation trace** | Per-policy, per-condition evaluation details (available via sign-in logs and Graph API) | Sign-in log → Applied Conditional Access |
| **Provisioning logs** | Enterprise app SCIM provisioning events | Enterprise Application → Provisioning → History |

---

## 4.15 VERSION/CURRENT SERVICE CONSIDERATIONS (Sept 2026)

| Aspect | Current State | L3 Impact |
|--------|--------------|-----------|
| **Legacy MFA deprecation** | Per-user MFA being fully retired. All MFA must be managed via Conditional Access. | If still using per-user MFA: migrate immediately. Old settings will stop working. |
| **Passwordless** | FIDO2 and Authenticator passwordless are GA and recommended. SMS/OTP for primary auth deprecated. | Plan passwordless rollout. Reduce password-dependent authentication. |
| **Authentication Strength** | GA and core feature. Maps auth methods to High/Medium/Low confidence levels. | Use Authentication Strength in CA for granular control (e.g., require High for admin tasks). |
| **Conditional Access** | Core feature in P1+. Most enterprise environments have many policies. | Policy management and auditing is critical. Use naming conventions, document policies, audit regularly. |
| **SMS/voice for MFA** | Still functional but strongly discouraged. Microsoft will deprecate further. | Plan migration to Authenticator/FIDO2. |
| **Named Locations** | GA. IPv4 and IPv6 support. Microsoft-managed locations for MFA registration. | Critical for Zero Trust. Define trusted locations for your organization. |
| **Session controls** | GA. Sign-in frequency (1-96 hours), persistent browser, CA authorization. | Essential for limiting token lifetime and enforcing re-authentication. |
| **Risk-based CA** | GA. Sign-in risk (all SKUs), user risk (P2). Risk-based block/require MFA. | Enable Identity Protection and risk-based CA policies. |
| **Cross-Tenant Access** | Updated portal. Settings for B2B, B2B direct connect, inbound/outbound. | Critical for organizations with B2B collaboration. |
| **Terms of Use** | GA. App-based legal acceptance. | Used for compliance scenarios. Not commonly the cause of incidents. |
| **SSPR** | GA across SKUs. Registration required before reset recommended. | Enterprise best practice: require SSPR registration. |
| **Identity Protection** | Premium feature (P2). Risk detections, risk-based CA integration. | Enable P2 license for security-sensitive environments. |
| **Cloud Sync vs Entra Connect** | Cloud Sync preferred for new deployments. Entra Connect (full) still supported. | Understand both for hybrid identity. Cloud Sync doesn't do PTA/PHS (those are Connect features). |
| **TOTP deprecation trajectory** | TOTP still supported but not recommended as primary. Microsoft pushing FIDO2/Authenticator. | Accept TOTP as fallback only. |
| **Push notification fatigue attacks** | Widely reported. Microsoft added number matching to Authenticator to counter. | Enable number matching, monitor rapid MFA prompts, prefer FIDO2. |
| **Interactive authentication for MS Graph** | MS Graph OAuth endpoints handle interactive and non-interactive auth. Token audiences differ for different resources. | Know which resource you're authenticating to (Graph vs Storage vs Key Vault). |
| **Token lifetime changes** | Microsoft has been reducing default token lifetimes for security. | Monitor token expiry, implement CA session controls, implement token refresh strategies. |

---

## 4.16 L3 INTERVIEW QUESTIONS

### Basic
**Q: What is the difference between authentication and authorization?**
A: Authentication verifies "who are you" (identity verification via credentials). Authorization determines "what are you allowed to do" (permissions based on roles, policies, and resource access controls). Authentication happens first; authorization happens after.

### Intermediate
**Q: What is Conditional Access?**
A: Conditional Access is Azure's real-time policy engine that makes access decisions at sign-in time based on conditions about the user, device, location, risk, and authentication method. It can require MFA, block access, enforce device compliance, or modify session behavior. It evaluates at every sign-in and is evaluated by policy priority.

### L3
**Q: A CA policy shows as "Applied" in the sign-in log, but the user was not required to do MFA. How?**
A: Possible reasons: (1) The user already has a valid token from a recent sign-in within the session lifetime — CA session controls may not force re-authentication yet. (2) The policy's grant control is "Require authentication strength = Medium" and the user already authenticated with a compliant device + MFA, which satisfies it — but they might not have experienced a separate MFA challenge if their initial sign-in already satisfied it. (3) The policy might have "Grant access" (no additional controls) as the grant control — it's "Applied" because it was evaluated and matched, but didn't require any additional action. (4) The policy might be in "Report Only" mode. (5) The user might be excluded by a higher-priority policy. Always check the specific grant control in the policy definition, not just "Applied."

### Senior L3
**Q: Explain why a service principal cannot satisfy a CA policy requiring MFA, and how you would architect around this.**
A: Service principals authenticate using client secrets or certificates, not through interactive sign-in. MFA requires a human to respond to a challenge (push notification, OTP entry, FIDO2 touch). Since there's no human present for an SP authentication, MFA cannot be performed. If a CA policy requiring MFA applies to SPs (via "All users" or "All service principals" in the Include), the SP will be blocked. The solution is to either (1) exclude service principals from MFA-required CA policies (explicit exclusion in the Users/Groups condition), or (2) create separate CA policies that handle SPs differently (e.g., require specific authentication strength but not interactive MFA), or (3) use PIM for elevated SP scenarios where human approval is needed.

### Expert
**Q: What happens internally when Conditional Access evaluates a sign-in request?**
A: Entra ID receives the sign-in request → Identifies the tenant and user → Authenticates credentials → Checks account state → Starts CA evaluation: For each policy (by priority): (1) Checks Users/Groups condition — is the user included and not excluded? (2) For each condition section (Locations, Device Platforms, Device State, Client Apps, Authentication Strength, Risk, Terms of Use) — are ALL individual conditions within that section TRUE? If any condition is FALSE, the policy is skipped. (3) If all conditions TRUE, evaluate Grant Controls: Are ALL grant controls satisfiable? (e.g., MFA done? Device compliant? Auth strength sufficient?) If YES → Grant/Deny as specified. If NO → Deny/Block. (4) First policy with a definitive decision wins. (5) Token is issued with any session modifications/CA claims. (6) If no policy makes a decision → standard authentication proceeds without CA controls.

### Scenario
**Q: "A user was added to a security group that's included in a CA policy requiring MFA. The user signed in 5 minutes later and MFA was NOT required. Why?"**
A: Possible reasons: (1) RBAC/group membership propagation delay — group membership changes can take up to 15 minutes to propagate to all systems including CA evaluation. (2) The user's sign-in was from a trusted location with "Require MFA only from untrusted locations" — but the CA policy wasn't configured that way, so if the user was in a trusted location, MFA wasn't triggered. (3) There might be a higher-priority CA policy that skips this user (exclusion). (4) The group membership might be dynamic and the user's attributes don't match the rule (or the rule hasn't been refreshed). (5) The user already has a valid token/session from a previous sign-in — CA session controls might not force re-evaluation. Check the sign-in log for applied policies and the exact CA evaluation result.

### Tricky
**Q: "I have a CA policy: Require MFA for all users on all apps from untrusted locations. A user is from an untrusted location but was NOT challenged for MFA. Why?"**
A: Multiple possible answers (this is designed as a tricky question):
1. The user might be in an **excluded group** in the CA policy (even though the policy says "All users" in Include, the user's group is in Exclude — and Exclude always wins).
2. There might be a **higher-priority CA policy** that grants the user access without MFA.
3. The user might have already completed MFA in this session, and the **session lifetime** hasn't expired yet (CA session controls say re-authenticate every X hours).
4. The user might be using a **trusted device** or location that the CA condition doesn't detect correctly (device state = "Not detected" means the condition might not evaluate as expected).
5. The user's authentication might be using **Authentication Strength** that already exceeds the requirement (e.g., FIDO2 already done, and "Require authentication strength = Medium" is already satisfied).
6. The policy might be in **Report Only** mode (looks enforced but only logs).

### Tricky 2
**Q: "I created a CA policy to block legacy authentication. It's not blocking SMTP/IMAP access. Why?"**
A: The CA policy's Client Apps condition likely does NOT include "Legacy authentication." CA will only evaluate a policy for authentication methods that match the Client Apps condition. If the Client Apps condition is set to "Any client app" but the policy doesn't explicitly call out legacy auth, the legacy authentication request may not match the policy's conditions. The correct configuration is to ensure the Client Apps condition explicitly includes "Microsoft Exchange ActiveSync" or set it to "Any client app" to catch legacy auth. Microsoft has documented this behavior extensively — CA does not evaluate policies for protocols that cannot perform the requested grant controls (like MFA).

### Tricky 3
**Q: "A user has FIDO2 as their default auth method, but CA shows they authenticated with 'Low' authentication strength. How?"**
A: Authentication Strength "Low" = MFA without a compliant/hybrid joined device. If the user's FIDO2 key was used but the device is NOT compliant or NOT hybrid joined, the overall authentication strength is "Low" — not "High." Authentication Strength is determined by BOTH the authentication method AND the device state. FIDO2 alone gives High confidence, but the CA system may classify the overall session differently depending on device state context. Check: Is the device in Intune compliance? Is it hybrid joined? The Authentication Strength mapping considers the weakest factor.

---

## 4.17 SCENARIO-BASED QUESTIONS

### Scenario 1: "User reports being repeatedly prompted for MFA every 30 minutes"
**Architecture:** CA session controls → Token lifetime → Authentication pipeline.
**Dependencies:** Session frequency setting, Token lifetime, CA Authorization setting.
**Checks:**
- Which CA policy controls their session? (check Applied CA in sign-in log)
- What is the Sign-in frequency setting? (if 30 minutes, that's the cause)
- Is Conditional Access Authorization enabled? (required for session controls to work at resource level)
- Is this user hitting a different policy than expected? (maybe they're in a different group)
- Is the token being refreshed silently or forcing re-auth?
**Root Cause:** CA session "Sign-in frequency" is set very low (30 min), or two conflicting policies with different session settings.
**Fix:** Increase sign-in frequency to reasonable value (e.g., 4-8 hours), validate that Conditional Access Authorization is properly configured.
**Validation:** User can work for extended period without MFA prompt, security not compromised.

### Scenario 2: "After enabling CA to require MFA, our automated deployment scripts stopped working"
**Architecture:** CA evaluation → Service principal authentication → Grant controls → Token issuance.
**Dependencies:** Service principal authentication, CA policy scope, MFA capability.
**Checks:**
- Are SPs included in the "Require MFA" CA policy? (default: yes, via "All users")
- Can SPs perform MFA? (no — no human to respond to challenge)
- Is the sign-in non-interactive (IsInteractive = false in log)?
- What grant control is blocking? ("Require MFA" — impossible for SP)
**Root Cause:** CA policy applies to service principals, which cannot perform MFA.
**Fix:** Exclude "All service principals" from the MFA-required policy, or create a separate SP-specific policy.
**Validation:** Scripts run successfully, no MFA prompt in automation logs, CA still enforced for users.

### Scenario 3: "Risk-based CA should have blocked a risky sign-in but didn't"
**Architecture:** Identity Protection risk detection → CA risk condition → Grant control → Decision.
**Dependencies:** Risk level threshold, CA policy configuration, risk detection type.
**Checks:**
- Is risk-based CA actually configured? (separate from risk detection)
- What risk level was detected vs the policy threshold? (policy set to block High, but only Medium was detected)
- Is the user in the CA policy's Include/Exclude?
- Was the risk "Resolved" before CA evaluated?
- Check the sign-in log: riskDetail field and Applied CA
**Root Cause:** Risk detection exists but risk-based CA policy not configured, or threshold mismatch, or user excluded.
**Fix:** Configure risk-based CA policy with correct thresholds, verify user is included, test.
**Validation:** Risky sign-in triggers CA action, user blocked or required to remediate.

### Scenario 4: "Guest user from partner tenant cannot access our app, CA blocks them"
**Architecture:** B2B collaboration → Guest authentication → CA evaluation → Resource access.
**Dependencies:** Cross-Tenant Access settings, CA policy targets, guest RBAC.
**Checks:**
- Is the guest in an excluded group in the CA policy?
- Does the CA policy have conditions that match guest users differently? (device compliance won't apply)
- Does the guest's home tenant have a CA policy blocking?
- Are Cross-Tenant Access settings correct?
- Is the guest's device in any state? (usually "Not detected" for B2B guests)
- What does the sign-in log say about the CA decision?
**Root Cause:** CA policy blocks guests because device state is "Not detected/Not compliant" and "Require compliant device" is in grant controls.
**Fix:** Modify CA policy to handle guests differently (exclude guests from device compliance requirement, or create separate guest policy).
**Validation:** Guest can authenticate and access app.

### Scenario 5: "User's device is compliant in Intune, but CA evaluates device state as 'Not compliant'"
**Architecture:** Intune compliance → Device registration → CA device state signal → CA evaluation.
**Dependencies:** Device registration, CA signal, Intune sync, compliance policies.
**Checks:**
- Is device registered with Entra ID? (Portal → Devices)
- Is device showing as "Compliant" in CA device state (not just Intune)?
- Is there a delay between Intune compliance and CA device state sync? (yes — can be minutes to hours)
- Is the device Hybrid Joined (required for some CA conditions)?
- Is the user's device sending device state signals during sign-in? (browser/device must support it)
**Root Cause:** CA device state sync delay, or device not registered in Entra ID (even though Intune-compliant), or device signal not transmitted during sign-in.
**Fix:** Wait for sync (or force sync), verify device registration in Entra ID, ensure device is properly registered and signaling.
**Validation:** CA device state = "Compliant" or "Is registered," user passes CA.

---

## 4.18 KNOWLEDGE TEST

1. **What is the difference between authentication and authorization?**
   Authentication = verifying identity ("who are you?"). Authorization = determining permissions ("what can you do?"). Authentication comes first and must succeed before authorization is evaluated.

2. **What are the three token types issued by Entra ID?**
   ID Token (user identity info), Access Token (resource access permission), Refresh Token (long-lived credential to get new access tokens).

3. **What is the difference between Application and Delegated permissions?**
   Application permissions: App acts as itself (Service Principal), no user involved. Delegated permissions: App acts on behalf of a signed-in user. Application permissions require admin consent.

4. **Why does Conditional Access sometimes not apply even when the policy looks correct?**
   Common reasons: (1) User/group is in an Exclude list (exclude always wins). (2) Higher-priority policy takes precedence. (3) Policy is in Report Only mode. (4) Not ALL conditions within a condition section are met (AND logic). (5) Grant controls are already satisfied. (6) Service principals can't satisfy MFA grant controls. (7) Legacy auth bypasses CA unless explicitly targeted.

5. **What is the difference between "Marked as compliant" and "Is compliant" in Conditional Access?**
   "Marked as compliant" checks if the device has a specific compliance attribute set in the directory. "Is compliant" is a broader check that also considers device registration and compliance state. In practice, both evaluate similar device state, but "Is compliant" includes additional dynamic checks.

6. **What is Authentication Strength?**
   A CA feature that categorizes authentication methods into confidence levels (High = FIDO2/certificate, Medium = MFA on compliant device, Low = basic MFA). CA policies can require specific strength levels.

7. **What is the session "Sign-in frequency" control?**
   A CA session control that forces the user to re-authenticate after a specified period (1-96 hours). Even if tokens are still valid, the user must go through CA evaluation again.

8. **What is the difference between Sign-in Risk and User Risk?**
   Sign-in Risk: Risk associated with a specific sign-in attempt (e.g., impossible travel from this IP). User Risk: Risk associated with the user account itself (e.g., leaked credentials, malware infection).

9. **Why do service principals get blocked by MFA-required CA policies?**
   SPs authenticate with secrets/certificates, not through interactive sign-in. There's no human to respond to an MFA challenge. The CA policy is evaluated, determines MFA is required, but it's impossible for the SP to satisfy it → blocked.

10. **What is the most common cause of "CA policy doesn't apply" issues?**
    Exclusions. The user or their group is in the Exclude section of the CA policy (or a higher-priority policy has an exclusion that covers this user). Exclude always takes precedence over Include.

11. **What is a Named Location and how is it used in CA?**
    A Named Location defines trusted IP ranges or countries. In CA, it's used under the Locations condition to determine whether a user is in a trusted location and what access controls apply based on that.

12. **What is legacy authentication and why is it a security concern?**
    Legacy auth uses older protocols (IMAP, POP, SMTP, older Office clients) that cannot perform MFA. CA cannot enforce MFA for these protocols unless explicitly targeted. This bypasses Zero Trust controls.

13. **What is the priority/precedence rule for CA policies?**
    Policies are evaluated in priority order (lowest number = highest priority). The first policy that matches and makes a decision (Grant or Deny) wins. Deny always takes precedence immediately.

14. **What is a Break-Glass account and why should it be excluded from CA?**
    A dedicated emergency account used to access the environment during CA-related incidents. Must be excluded from CA to ensure access is always possible even if CA is misconfigured or blocking everyone.

15. **What is the 200 group claim limit's impact on CA?**
    If a user is in more than 200 groups, the group claims in the token are truncated/encoded. CA evaluation might not correctly reflect all group memberships (CA also evaluates live directory, so this is more of an RBAC issue, but it can compound).

16. **What is token refresh rotation and why does it matter?**
    Each time a refresh token is used, a new refresh token is issued and the old one is invalidated. If an attacker steals a refresh token and tries to reuse it (after the legitimate user already used it), the old token is rejected and ALL tokens in that session are revoked. This detects token theft.

---

## 4.19 L3 GAP CHECK

| Topic | Status |
|-------|--------|
| Authentication vs Authorization distinction | ✅ Covered |
| Authentication Methods (all types) | ✅ Covered |
| Microsoft Authenticator (push, OTP, passwordless) | ✅ Covered |
| FIDO2 Security Key | ✅ Covered |
| Passwordless authentication | ✅ Covered |
| Temporary Access Pass | ✅ Covered |
| SMS/voice (deprecated trajectory) | ✅ Covered |
| Password management and SSPR | ✅ Covered |
| MFA (per-user vs CA-based) | ✅ Covered |
| Legacy MFA deprecation | ✅ Covered |
| Token types (ID, Access, Refresh) | ✅ Covered |
| Token lifetime concepts | ✅ Covered |
| Token refresh and rotation | ✅ Covered |
| Complete authentication flow (13 steps) | ✅ Covered |
| Sign-in logs and evidence | ✅ Covered |
| Conditional Access — overview | ✅ Covered |
| CA — Users/Groups condition (include/exclude logic) | ✅ Covered |
| CA — Applications condition | ✅ Covered |
| CA — Locations/Named Locations | ✅ Covered |
| CA — Device Platforms | ✅ Covered |
| CA — Device State (compliance, registered, hybrid joined) | ✅ Covered |
| CA — Client Apps (modern vs legacy) | ✅ Covered |
| CA — Authentication Strength | ✅ Covered |
| CA — Risk-based access | ✅ Covered |
| CA — Terms of Use | ✅ Covered |
| CA — Grant Controls (complete list) | ✅ Covered |
| CA — Session Controls | ✅ Covered |
| CA — Policy precedence | ✅ Covered |
| CA — Service principals and CA | ✅ Covered (critical) |
| CA — Legacy authentication and CA | ✅ Covered |
| CA — Multiple policy interactions/conflicts | ✅ Covered |
| CA — Break-glass accounts | ✅ Covered |
| CA — Exclusion analysis (troubleshooting) | ✅ Covered |
| CA — Security considerations | ✅ Covered |
| CA — Monitoring targets | ✅ Covered |
| CA — Production example | ✅ Covered |
| CA — Failure scenarios | ✅ Covered |
| CA — Troubleshooting methodology | ✅ Covered |
| CA — Evidence sources | ✅ Covered |
| Current service considerations | ✅ Covered |
| L3 interview questions (all levels) | ✅ Covered |
| Scenario-based questions | ✅ Covered |
| Knowledge test | ✅ Covered |
| L3 gap check | ✅ Covered |

---

## 4.20 WHAT AN EXPERIENCED AZURE L3 ENGINEER SHOULD NOW BE ABLE TO EXPLAIN CONFIDENTLY

After Module 4, you should be able to confidently explain:

1. **The complete authentication pipeline** — from user entering credentials, through tenant lookup, credential validation, MFA challenge, Conditional Access evaluation, token issuance, to resource access. Every step can fail independently and each has distinct troubleshooting approaches.

2. **The three different token types** (ID, Access, Refresh) — what each contains, their lifetimes, how they're used, and how to decode them for troubleshooting.

3. **Why Conditional Access is the primary Zero Trust enforcement mechanism** — and how it differs fundamentally from RBAC (RBAC says "what you can do," CA says "when and how you can do it").

4. **The critical importance of CA policy precedence** — and why a policy that "looks correct" doesn't apply (higher-priority policy wins, exclusion takes precedence, Report Only mode, conditions not all met).

5. **Why service principals cannot satisfy MFA-required CA policies** — and the resulting production outages when SPs aren't explicitly excluded. This is one of the most common CA-related production incidents.

6. **The difference between legacy and modern authentication** — and why legacy auth (IMAP, POP, SMTP) bypasses Conditional Access unless explicitly targeted via Client Apps condition.

7. **How Authentication Strength works** — and why a user with FIDO2 can still authenticate with "Low" strength (if device is not compliant). Authentication Strength = method × device context.

8. **The difference between Intune compliance and CA device state** — they are related but not identical, and sync delays between them cause many troubleshooting confusion.

9. **The complete CA evaluation flow** — matching → conditions evaluation → grant controls → session controls → decision. Understanding that AND logic within each condition section means ALL must be true for the condition section to match.

10. **How to read a sign-in log for authentication/CA troubleshooting** — extracting StatusDetail, ConditionalAccessStatus, Applied Conditional Access, Authentication Details, Risk Detail, and Device Detail.

11. **Why token refresh rotation is a critical security feature** — detecting stolen tokens through replay prevention, and how it affects session recovery after token theft.

12. **Why break-glass accounts are essential** — and how they must be excluded from ALL CA policies, with separate authentication methods and access restrictions.

13. **The most common CA troubleshooting checklist** — policy enabled? Enforced mode? User included/excluded? All conditions met? Grant controls satisfiable? Higher-priority policy? This checklist is the foundation of every CA troubleshooting investigation.

---

# Ready for Module 5 — Entra Connect / Cloud Sync (Hybrid Identity)?

It covers:
- Entra Connect (full) vs Cloud Sync (modern) architecture
- Sync rules, filtering, attribute flow
- Source Anchor / Immutable ID
- Password Hash Sync, Pass-through Authentication, Federation
- Password Writeback
- Device synchronization
- Hybrid Join (Soft Match vs Hard Match)
- Duplicate identities
- Sync errors and troubleshooting
- Staging mode, High Availability
- And ALL required module components

Say **"Next module"** to continue.