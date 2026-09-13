# 🔐 PKI / CERTIFICATE SERVICES — COMPLETE ENTERPRISE TEACHING GUIDE

> **Context:** This is part of your L3 Windows & Virtualization Administrator interview prep. PKI/AD CS sits at the intersection of **Windows Server**, **Active Directory**, **Group Policy**, and **DNS** — all covered in Part 1. This guide assumes you understand those foundations.

---

## 📌 PHASE 1 — THE COMPLETE PKI CHAIN (Conceptual Architecture)

### 1.1 The Hierarchy: How It All Connects

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        PUBLIC KEY INFRASTRUCTURE (PKI)                   │
│                                                                          │
│  ┌─────────────┐                                                         │
│  │  ROOT CA    │  ← Self-signed, offline (best practice), trusted anchor │
│  │  (e.g.,     │  ← Stored in secure hardware (HSM) if possible          │
│  │   Contoso     │  → Signs Intermediate CA certificates                  │
│  │   Root CA)   │  → Highest trust level                                 │
│  └──────┬───────┘                                                         │
│         │ Signs                                                          │
│         ▼                                                                │
│  ┌─────────────┐                                                         │
│  │ INTERMEDIATE │  → One or more levels (depends on org policy)           │
│  │    CA       │  → Online (issuing CA), handles daily enrollment        │
│  │  (e.g.,     │  → Subordinate to Root CA                              │
│  │   Issuing     │  → Signs end-entity certificates                      │
│  │   CA)       │  → Can be multiple (Dev, Prod, Email, etc.)            │
│  └──────┬───────┘                                                         │
│         │ Signs                                                          │
│         ▼                                                                │
│  ┌─────────────┐                                                         │
│  │   CERTIFICATE│  ← End-entity (server, user, computer, code signing)   │
│  │   (e.g.,     │  ← Contains: Subject, Public Key, Issuer, Validity,   │
│  │    server.    │    Thumbprint, Serial, Extensions                       │
│  │    contoso    │  ← Signed by Intermediate CA                           │
│  │    .com)     │  → Presented during SSL/TLS handshake                   │
│  └──────┬───────┘                                                         │
│         │                                                                │
│  ┌──────┴───────┐                                                         │
│  │  PRIVATE KEY │  ← NEVER transmitted; stored on the server/host        │
│  │              │  ← Used to decrypt data and sign messages              │
│  │  (corresponding│  ← If compromised → entire certificate is suspect     │
│  │   to cert)   │  → Password protected (if software-based)              │
│  └──────────────┘                                                         │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ ENROLLMENT LIFECYCLE (Automated or Manual):                         │ │
│  │                                                                      │ │
│  │   CSR → Enrollment → Certificate Issued → Renewal → Revocation       │ │
│  │     │         │                  │          │            │           │ │
│  │     ▼         ▼                  ▼          ▼            ▼           │ │
│  │   Request   Submit to CA     Issued    Before expiry  Cancel        │ │
│  │   (request  (CAS/PKI)        (signed    or keys        (CRL/OCSP)   │ │
│  │   of public +                   cert)      compromised)             │ │
│  │   private key)                                                                  │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌──────────────────┐    ┌──────────────────┐                           │
│  │    CRL            │    │     OCSP          │                          │
│  │  (Certificate     │    │  (Online          │                          │
│  │   Revocation List)│    │   Protocol)       │                          │
│  │                   │    │                   │                          │
│  │  Published by CA  │    │  Real-time check  │                          │
│  │  Periodically (e.g│    │  Checks if cert   │                          │
│  │  every 12 hours)  │    │  is revoked NOW   │                          │
│  │  Distributed via  │    │  Faster but needs │                          │
│  │  LDAP/HTTP/CDP    │    │  internet/latency │                          │
│  │  points in cert   │    │                   │                          │
│  └──────────────────┘    └──────────────────┘                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 📌 PHASE 2 — EVERY COMPONENT EXPLAINED (Deep, Not Shallow)

### 2.1 Root CA (Certificate Authority)

**What it is:** The top of the trust chain. A self-signed certificate that is the **trust anchor** for the entire PKI. It signs (cryptographically) the certificates of Intermediate CAs.

**Why it's used:**
- Establishes **trust boundary** — anything signed by this CA (directly or via chain) is trusted by clients who have this Root CA in their Trusted Root store
- In enterprise: your internal Root CA is distributed via GPO to all domain computers' Trusted Root Certification Authorities store
- In public: Root CAs (DigiCert, Let's Encrypt, etc.) are pre-installed in OS/browser trust stores

**How it works:**
```
1. Root CA generates a self-signed certificate:
   - Subject: CN=Contoso Root CA, O=Contoso, C=US
   - Public Key: RSA 2048/4096 or ECC
   - Private Key: Secured in HSM (Hardware Security Module) or encrypted file
   - Validity: Often 5-20 years (long-lived)
   - Extensions: Key Usage (Certificate Signing, CRL Signing), Basic Constraints (CA:TRUE, Path Length: 0 or more)

2. Uses private key to sign Intermediate CA certificate requests (CSR)

3. Best practice: OFFLINE — used only to sign Intermediate CA CSRs, then powered off
```

**Key Components:**
- **Root CA Certificate** — Self-signed, in Trusted Root store
- **Root CA Private Key** — THE most sensitive piece in entire PKI
- **Root CA Database** — Tracks issued certificates (serial numbers)
- **Key Recovery Agent** — For private key recovery if needed

**Dependencies:**
```
Root CA depends on:
├── AD CS (Certificate Services role installed)
├── Enterprise PKI (AD CS Enterprise CA type)
├── GPO (to distribute Root CA certificate to all domain computers)
├── (Ideally) HSM for private key storage
└── Time service (certificate validity relies on correct time)

Root CA is depended upon by:
├── Intermediate CAs (they get their cert signed by Root CA)
├── Every certificate in the entire forest
└── Clients (trust store must have Root CA to validate anything)
```

---

### 2.2 Intermediate CA

**What it is:** A subordinate CA that is signed by the Root CA. It acts as the **issuing CA** for end-entity certificates (servers, users, computers).

**Why it's used:**
- **Security**: Root CA can be kept offline; Intermediate CA handles all day-to-day issuance
- **Delegation**: Different Intermediate CAs for different purposes (Web, Code Signing, Email, Internal)
- **Scalability**: Multiple issuing CAs can exist; each can have its own policies and certificate templates
- **Compromise isolation**: If an Intermediate CA is compromised, only certificates it issued need revocation — the Root CA stays safe, and you can issue new Intermediate CAs

**How it works:**
```
1. Root CA generates Intermediate CA certificate:
   - Subject: CN=Contoso Issuing CA 01, O=Contoso
   - Public Key: Generated on Intermediate CA server
   - CSR sent to Root CA → Root CA signs it
   - Signed certificate returned to Intermediate CA

2. Intermediate CA now can:
   - Accept CSR from servers/users wanting certificates
   - Issue (sign) end-entity certificates
   - Publish CRL (Certificate Revocation List)
   - Respond to OCSP requests

3. Validity: Typically shorter than Root CA (5-10 years)

4. Can have multiple levels: Root CA → Sub-Intermediate CA → Issuing CA → Certificate
```

**Dependencies:**
```
Intermediate CA depends on:
├── Root CA (for signing its certificate)
├── AD CS role
├── Certificate Templates (defines what certs can be issued)
├── Enrollment policy (who can request what)
├── CRL publication points
└── Time service

Intermediate CA is depended upon by:
├── All end-entity certificates it issues
├── Clients validating certificate chain (must trust Root CA → which signed Intermediate CA)
└── OCSP responders (often hosted on Intermediate CA servers)
```

---

### 2.3 Certificate (End-Entity)

**What it is:** A digital document that binds a public key to an identity (subject). It's the actual artifact used in SSL/TLS, S/MIME, code signing, smart card logon, etc.

**Why it's used:**
- Prove identity (I am server01.contoso.com)
- Enable encrypted communication (TLS uses certificate's public key)
- Enable digital signatures (code signing, email signing)
- Enable authentication (client certificates, smart card logon)

**How it works:**
```
Certificate contains:
├── Version (v3)
├── Serial Number (unique, assigned by CA)
├── Signature Algorithm (e.g., sha256RSA)
├── Issuer (who signed it: CN=Contoso Issuing CA)
├── Validity Period (Not Before / Not After)
├── Subject (CN=server01.contoso.com, O=Contoso)
├── Subject Public Key Info (Public Key + Algorithm)
├── Extensions:
│   ├── Key Usage (Digital Signature, Key Encipherment, etc.)
│   ├── Enhanced Key Usage (Server Authentication = 1.3.6.1.5.5.7.3.1)
│   ├── Basic Constraints (CA:FALSE for end-entity)
│   ├── Subject Alternative Name (DNS names, IPs, emails)
│   ├── CRL Distribution Points (where to get CRL)
│   └── Authority Information Access (where to get OCSP response)
├── Certificate Signature (CA's signature over all above)
└── Thumbprint (hash of certificate, used for identification)
```

---

### 2.4 Private Key

**What it is:** A cryptographic key that is **never shared**. It corresponds to the public key in the certificate. Used to:
- **Decrypt** data encrypted with the public key (SSL/TLS handshake)
- **Sign** data (proving identity, code signing)
- **Unlock** encrypted communications

**Why it matters:**
- If the private key is compromised, an attacker can impersonate the certificate holder
- The entire trust chain is broken if the private key is leaked
- **Key escrow** and **Key Recovery Agents** are for emergency recovery

**Where it's stored:**
- Windows: `C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys\` (machine store)
- Windows: User profile certificate store (user certificates)
- Best: Hardware Security Module (HSM) for Root/Intermediate CA
- Certificate store: `certlm.msc` (Local Machine), `certmgr.msc` (Current User)

---

### 2.5 CSR (Certificate Signing Request)

**What it is:** A request sent to a CA to issue a certificate. It contains:
- The subject's **public key** (generated by the requester)
- Subject information (CN, O, OU, etc.)
- Requested extensions (Key Usage, EKU, etc.)
- The requester's **signature** over the above (proving they own the corresponding private key)

**Why it's used:**
- Formal process for requesting a certificate
- Proves requester owns the private key (without revealing it)
- CA uses the CSR to construct the issued certificate

**How it's created:**
```powershell
# Method 1: Using certreq (Windows)
certreq -new CSR.txt server01.contoso.com.cer

# Method 2: Using IIS Manager (web server)
# IIS → Server Certificates → Create Certificate Request

# Method 3: Using OpenSSL
openssl req -new -newkey rsa:2048 -nodes \
  -keyout server01.key \
  -out server01.csr \
  -subj "/CN=server01.contoso.com/O=Contoso/C=US"

# Method 4: Using PowerShell (New-SelfSignedCertificate for self-signed)
New-SelfSignedCertificate -DnsName "server01.contoso.com" -CertStoreLocation "Cert:\LocalMachine\My"

# Method 5: AD CS Web Enrollment (Enterprise)
# Users go to https://ca.contoso.com/certsrv → Request a certificate
```

**CSR File Format:**
```
-----BEGIN CERTIFICATE REQUEST-----
MIIBkTCB+wIBADBSMQswCQYDVQQGEwJBVTEPMA0GA1UEChMGQ09udG9zbw==
... (Base64-encoded DER containing public key + subject + signature)
-----END CERTIFICATE REQUEST-----
```

---

### 2.6 Enrollment

**What it is:** The complete process of requesting, validating, and issuing a certificate.

**Two Modes in AD CS:**

| Mode | Description | Who Uses It |
|------|-------------|-------------|
| **Enterprise Enrollment** | Uses AD CS; integrates with AD for auto-enrollment, template-based, authenticated | Domain-joined computers/users |
| **Standalone Enrollment** | Manual request (certreq, web enrollment, SCEP); no AD integration | Non-domain systems, cross-platform |

**Enterprise Enrollment Flow:**
```
Client → Requests certificate from AD CS Enrollment Policy
       → AD CS validates:
       │   ├── Is client domain-joined?
       │   ├── Does client have permission for this template?
       │   ├── Does template exist?
       │   ├── Is the client's security group membership appropriate?
       │   └── Are there enrollment restrictions?
       → Intermediate CA issues certificate using its private key
       → Certificate published in AD (optional, for auto-enrollment of other computers)
       → Certificate delivered to client
       → Certificate added to client's certificate store
```

**Key Components:**
- **Certificate Templates** — Define certificate type (Web Server, Smart Card Logon, etc.)
- **Enrollment Services** — CA web enrollment, SCEP, auto-enrollment
- **Enrollment Policy** — Rules governing who can request what

---

### 2.7 Renewal

**What it is:** Re-issuing a certificate before it expires (or after, depending on CA settings). Keeps the same key pair (or generates new one) and extends validity.

**Why it's used:**
- Certificates have expiration (typically 1-2 years for end-entity; 5-20 for CA)
- Expired certificates cause SSL/TLS failures, auth issues, and trust breaks
- Renewal ideally happens **before expiration** to avoid service disruption

**Two Types:**
- **In-Place Renewal**: Same private key, new certificate with extended validity (common for servers)
- **Re-Key Renewal**: New private key AND new certificate (more secure, recommended periodically)

**How it works (Enterprise Auto-Renewal):**
```
1. Auto-enrollment configured via GPO
2. Certificate auto-renews when:
   - Current certificate expires within renewal threshold (e.g., 20% of validity)
   - OR certificate reaches 80% of its validity period (configurable)
3. Process:
   ├── Client checks for renewal (certutil, auto-enrollment scheduler)
   ├── New CSR generated with same subject/existing key (or new key)
   ├── Request sent to CA
   ├── CA issues new certificate
   ├── Old certificate still valid until Not Before / Not After dates
   ├── New certificate installed
   └── Services using old cert need restart to pick up new cert (or auto-discovery)
```

**PowerShell:**
```powershell
# Force renewal of all auto-enrollment certificates
certutil -repairstore -user my "*"
# Or via PowerShell
Get-ChildItem Cert:\CurrentUser\My | Where-Object { $_.NotAfter -lt (Get-Date).AddMonths(3) }

# Renew a specific certificate using certreq
certreq -submit -renew existing_cert.cer new_cert.cer
```

---

### 2.8 Revocation

**What it is:** The process of **cancelling a certificate before its expiration date**. The certificate is added to a revocation list and clients check this list before trusting the certificate.

**Why it's used:**
- Private key compromised
- Certificate issued to wrong entity
- Employee left the company (smart card logon cert)
- Server decommissioned
- Certificate misissued
- Security incident

**Revocation Types:**
- **Revoke by CA**: CA marks certificate revoked in its database
- **KRA (Key Recovery Agent)**: Can revoke in specific scenarios

**Revocation Event:**
```
1. Admin or automated system requests revocation
2. CA marks certificate as revoked:
   ├── Adds serial number to CRL (next CRL publication)
   ├── Updates OCSP cache (if used)
   └── Records revocation reason (Key Compromise, CA Compromise, Affiliation Changed, etc.)
3. Revocation reason codes:
   0 = Unspecified
   1 = Key Compromise
   2 = CA Compromise
   3 = Affiliation Changed
   4 = Superseded
   5 = Cessation of Operation
   6 = Certificate Hold
   8 = Remove from CRL
   9 = Privilege Withdrawn
   10 = AA Compromise
```

---

### 2.9 CRL (Certificate Revocation List)

**What it is:** A list of certificate serial numbers that have been revoked by the CA. It's published periodically and distributed to clients.

**How it works:**
```
┌─────────────────────┐
│     CA Server       │
│  (Revocation Engine)│
└─────────┬───────────┘
          │ Publishes
          ▼
┌─────────────────────┐
│  CRL (Base64/DER)   │
│  Contains:           │
│  - Revoked serials   │
│  - This CRL period   │
│  - Next CRL period   │
│  - Signature by CA   │
└─────────┬───────────┘
          │ Distributed via
          ▼
  ┌─────────────┐    ┌─────────────┐
  │ LDAP URL     │    │ HTTP/HTTPS  │
  │ (default)    │    │ URL in cert │
  └─────────────┘    └─────────────┘
          │
          ▼
┌─────────────────────┐
│ Clients check CRL   │
│ during SSL/TLS      │
│ handshake            │
└─────────────────────┘
```

**CRL Properties:**
- Published every 12 hours (default in AD CS)
- Can be Base64 (human readable) or DER (binary)
- Signed by the issuing CA (validated by clients)
- Large enterprises may have delta CRLs (only changes since last CRL)

---

### 2.10 OCSP (Online Certificate Status Protocol)

**What it is:** A real-time protocol for checking whether a specific certificate is revoked. Instead of downloading entire CRL, client queries OCSP responder for one certificate's status.

**How it works:**
```
Client → Sends OCSP request (certificate serial number) → OCSP Responder
OCSP Responder checks CA database → Returns "Good" / "Revoked" / "Unknown"
Response is signed by OCSP responder (or CA) → Client validates response

Compared to CRL:
├── CRL: Download entire list → check locally → slower for large environments
├── OCSP: Query for one cert → real-time → faster but needs OCSP responder available
└── OCSP stapling: Server provides OCSP response during TLS handshake (reduces client-side OCSP dependency)
```

---

## 📌 PHASE 3 — DEPENDENCY MAP (Complete)

```
PKI / AD CS DEPENDENCY MAP
══════════════════════════
                          ┌──────────────┐
                          │  AD DS       │
                          │  (Enterprise │
                          │  PKI basis)  │
                          └──────┬───────┘
                                 │ AD CS Enterprise CA depends on AD for
                                 │ security groups, auto-enrollment, template
                                 │ security, and certificate publishing
                                 │
                          ┌──────▼───────┐
                          │ AD CS Role   │
                          │ (ADCS/DC01)  │
                          └──────┬───────┘
                                 │
               ┌─────────────────┼─────────────────┐
               ▼                 ▼                 ▼
        ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
        │ ROOT CA     │  │ INTERMEDIATE│  │ CERTIFICATE  │
        │ (Offline)   │  │ CA (Online) │  │ TEMPLATES    │
        │             │  │             │  │ (Web Server, │
        │ Depends:    │  │ Depends:    │  │  Client Auth,│
        │ - HSM       │  │ - Root CA   │  │  Email, etc.)│
        │ - AD CS     │  │ - AD CS     │  │              │
        │ - Key Storage│ │ - Templates │  │ Depends:     │
        │ - GPO (root   │ │ - Enrollment│  │ - Private Key│
        │  cert dist) │ │   Policy    │  │ - CSR        │
        └─────────────┘  └──────┬──────┘  │ - CA         │
                                 │        │ - SCP/Cert   │
                                 │        │   Publishing │
                                 │        └──────┬───────┘
                                 │               │
                                 │    ┌──────────┼──────────┐
                                 │    ▼          ▼          ▼
                                 │ ┌──────┐ ┌──────┐ ┌────────┐
                                 │ │ENROLL│ │RENEW │ │REVOKE  │
                                 │ │      │ │      │ │        │
                                 │ │Depends│ │Depends│ │Depends│
                                 │ │ - CA  │ │ - CA  │ │ - CA   │
                                 │ │ - GPO │ │ - GPO │ │ - CRL  │
                                 │ │ - AD  │ │ - Auto│ │ - OCSP │
                                 │ │ - Cert│ │ - Cert│ │        │
                                 │ │   Tmpl│ │   Exp │ │        │
                                 │ └──┬───┘ └──┬───┘ └──┬─────┘
                                 │    │        │        │
                                 │    ▼        ▼        ▼
                                 │ ┌──────────────────────────┐
                                 │ │ CRL & OCSP               │
                                 │ │ (Revocation Checking)     │
                                 │ │                          │
                                 │ │ Depends:                 │
                                 │ │ - CA Database            │
                                 │ │ - LDAP / HTTP Publication│
                                 │ │ - Time Service (CRL sig) │
                                 │ │ - Clients check during   │
                                 │ │   TLS handshake          │
                                 │ └──────────────────────────┘
                                 │
         ┌────────────────────────┼────────────────────┐
         │ GPO distributes:       │                    │
         │ - Root CA cert to      │                    │
         │   Trusted Root store   │                    │
         │ - Auto-enrollment      │                    │
         │   policy settings      │                    │
         │ - Certificate templates│                    │
         │ - CRL/OCSP behavior    │                    │
         └────────────────────────┘                    │
                                                      │
    ┌─────────────────────────────────────────────────┘
    ▼
┌──────────────────────┐
│ DNS                  │
│ (AD-integrated zones│
│  for CRL, SCP, cert │
│  publishing; SRV    │
│  for CA location)   │
└──────────────────────┘

Additional Dependencies:
├── Time Service (w32time): Certificate validity periods rely on correct time
├── Network: Clients must reach CA, CRL, OCSP endpoints
├── Storage: CA database, CRL files, certificate templates
├── Permissions: AD security groups for enrollment, template access
└── Certificates themselves: Chain validation requires all certs in chain
```

---

## 📌 PHASE 4 — SCENARIOS WITH TROUBLESHOOTING

---

## SCENARIO 1: Certificate Expired

### What Happens:
- Certificate's "Not After" date has passed
- Clients reject the certificate during SSL/TLS handshake or authentication
- Common for: Web servers (IIS), email (S/MIME), code signing, smart card logon, AD CS renewal

### Dependency Chain Affected:
```
Certificate Expired → TLS/SSL fails → Authentication fails → Application down → Users impacted
```

### Detection — Commands & Tools:

```powershell
# 1. Check certificate expiration (from server where cert is installed)
# Local Machine certificate store:
Get-ChildItem Cert:\LocalMachine\My | 
  Select-Object Subject, NotAfter, Thumbprint, 
    @{Name='DaysUntilExpiry';Expression={($_.NotAfter - (Get-Date)).Days}} |
  Sort-Object NotAfter | Format-Table -AutoSize

# Current User:
Get-ChildItem Cert:\CurrentUser\My |
  Select-Object Subject, NotAfter, 
    @{Name='DaysUntilExpiry';Expression={($_.NotAfter - (Get-Date)).Days}} |
  Sort-Object NotAfter | Format-Table -AutoSize

# 2. Check specific certificate by thumbprint:
Get-ChildItem Cert:\LocalMachine\My\THUMBPRINT | Select-Object *

# 3. Check ALL certificate stores for expiring certs:
Get-ChildItem Cert:\LocalMachine\* -Recurse |
  Where-Object { $_.NotAfter -lt (Get-Date).AddDays(30) } |
  Select-Object Subject, NotAfter, Thumbprint, PSPath |
  Sort-Object NotAfter | Format-Table -AutoSize

# 4. Check a remote server's certificate:
Invoke-Command -ComputerName web01.contoso.com -ScriptBlock {
  Get-ChildItem Cert:\LocalMachine\My |
    Where-Object { $_.NotAfter -lt (Get-Date).AddDays(90) } |
    Select-Object Subject, NotAfter, Thumbprint
}

# 5. Check from external perspective (like a client would):
# Test-TLSConnection (hypothetical) — use:
openssl s_client -connect server01.contoso.com:443 -servername server01.contoso.com 2>/dev/null | 
  openssl x509 -noout -dates
# Output: notBefore=Jan 1 12:00:00 2025 GMT; notAfter=Jan 1 12:00:00 2026 GMT

# 6. Check IIS bindings:
Get-WebBinding | Where-Object { $_.protocol -eq "https" }
Get-ChildItem IIS:\SslBindings\ | Select-Object IP, Port, CertificateThumbprint

# 7. Check specific site binding:
Import-Module WebAdministration
Get-ItemProperty "IIS:\SslBindings\0.0.0.0!443!" -Name certificateHash

# 8. Check Exchange certificates:
Get-ExchangeCertificate | Select-Object Subject, NotAfter, Thumbprint, Services

# 9. Check AD CS certificate:
Get-ChildItem Cert:\LocalMachine\My | Where-Object { $_.Subject -like "*CA*" }

# 10. Check Enterprise PKI health (now deprecated in 2012 R2+, use:
certutil -view -restrict "NotAfter<=2026-09-11" -out serial,notafter,raw.Certificate

# 11. Event Logs to check:
Get-WinEvent -FilterHashtable @{LogName='System'; ID=36882,36886,36887}
# 36882 = TLS/SSL alert (handshake failed)
# 36886 = Certificate is expired
# 36887 = Certificate is not yet valid
Get-WinEvent -FilterHashtable @{LogName='Application'; Source='Schannel'} -MaxEvents 30
Get-WinEvent -FilterHashtable @{LogName='Application'; Source='IIS'} -MaxEvents 30
```

### Troubleshooting Steps:

```
STEP 1: IDENTIFY THE EXPIRED CERTIFICATE
  ├── Run the above commands to find which cert expired
  ├── Identify: Server, Application, Certificate Purpose
  └── Note: Thumbprint, Subject, Expiration Date

STEP 2: ASSESS IMPACT
  ├── What service uses this certificate? (IIS site, SMTP, IMAP, etc.)
  ├── How many clients are affected?
  ├── Is this a public-facing or internal cert?
  └── Does it affect authentication (AD FS, Exchange OAB, Autodiscover)?

STEP 3: RENEW / REISSUE THE CERTIFICATE
  ├── Option A: AD CS Auto-Enrollment (if enterprise CA)
  │   ├── Force renewal: certutil -repairstore -user my "*" 
  │   ├── Or: certutil -pulse
  │   ├── Check auto-enrollment via: gpresult /r
  │   └── GPO: Computer Config → Windows Settings → Security Settings →
  │       Public Key Policies → Certificate Services → Auto Enrollment
  ├── Option B: IIS Certificate Renewal
  │   ├── IIS → Server Certificates → Renew (requires CA access)
  │   └── Or: New certificate from CA → Bind to IIS site
  ├── Option C: Manual CSR → CA → Install cert
  │   ├── Generate CSR via certreq or IIS
  │   ├── Submit to CA (Enterprise web enrollment or manual)
  │   ├── CA issues certificate
  │   ├── Install: certutil -addstore My new_cert.cer
  │   └── Bind to service/IIS
  └── Option D: Let's Encrypt (for public-facing, if applicable)
      └── Use Win-ACME or similar

STEP 4: INSTALL & BIND
  ├── Import certificate: certutil -addstore My cert.cer
  ├── For IIS: Bind to site using new thumbprint
  ├── For Exchange: Enable on services: Enable-ExchangeCertificate -Thumbprint X -Services IIS,SMTP
  ├── For AD FS: Set-ADFSPrincipalCertificate
  └── Restart service: Restart-WebAppPool, Restart-Service MSExchange…, iisreset

STEP 5: VERIFY
  ├── Check new cert is valid: Get-ChildItem Cert:\LocalMachine\My\THUMB
  ├── Test TLS: openssl s_client -connect server:443
  ├── Browser check (external)
  ├── Application test
  └── Event log check for errors

STEP 6: PREVENT RECURRENCE
  ├── Enable auto-enrollment (GPO-based)
  ├── Add monitoring/alerting: Alert at 30, 14, 7 days before expiry
  ├── Use centralized certificate management (Microsoft CA, Venafi, etc.)
  └── Document: Certificate inventory spreadsheet
```

### Real-World Interview Answer:
> "I'd first identify the expired certificate using `Get-ChildItem Cert:\LocalMachine\My` filtered by expiration date. Then I'd determine which service it's bound to. For IIS, I'd generate a CSR via IIS Manager, submit it to our Enterprise CA via web enrollment, receive the issued certificate, install it back into the Local Machine store, update the IIS binding with the new thumbprint, run `iisreset`, and verify via `openssl s_client` or a browser check. To prevent recurrence, I'd enable auto-enrollment via GPO and implement certificate expiration monitoring in our monitoring system."

---

## SCENARIO 2: Certificate Chain Broken

### What Happens:
- Client receives a certificate, but cannot build a trust chain back to a trusted Root CA
- Common causes: Intermediate CA cert missing from server, Root CA not in client's Trusted Root store, wrong intermediate cert installed, cross-certification issues, or broken AD CS replication

### Dependency Chain Affected:
```
Chain Broken → Client cannot validate certificate → TLS handshake fails → 
Authentication fails → Applications reject connection
```

### Detection:

```powershell
# 1. Check certificate chain on the server
Get-ChildItem Cert:\LocalMachine\My\THUMBPRINT | 
  Select-Object Subject, Issuer, Thumbprint, 
    @{Name='ChainStatus';Expression={
      $chain = New-Object System.Security.Cryptography.X509Certificates.X509Chain
      $chain.Build($_)
      $chain.ChainStatus | Select-Object Status, StatusInformation
    }}

# 2. Verify chain manually:
$cert = Get-ChildItem Cert:\LocalMachine\My\THUMBPRINT
$chain = New-Object System.Security.Cryptography.X509Certificates.X509Chain
$chain.ChainPolicy.RevocationMode = [System.Security.Cryptography.X509Certificates.X509RevocationMode]::NoCheck
$chain.Build($cert)
$chain.ChainElements | ForEach-Object {
    [PSCustomObject]@{
        CertificateSubject = $_.Certificate.Subject
        Issuer = $_.Certificate.Issuer
        Status = $_.ChainStatus.Status
        StatusInfo = $_.ChainStatus.StatusInformation
    }
}

# 3. From client perspective — test chain:
openssl s_client -connect server01.contoso.com:443 -showcerts 2>&1 | grep -A 5 "Verify return code"
# Expected: Verify return code: 0 (ok)
# Broken: Verify return code: 2 (unable to get issuer certificate) or 20 (unable to get local issuer certificate)

# 4. Check if Root CA is in Trusted Root store:
Get-ChildItem Cert:\LocalMachine\Root | Where-Object { $_.Subject -like "*Contoso Root CA*" }
Get-ChildItem Cert:\CurrentUser\Root | Where-Object { $_.Subject -like "*Contoso Root CA*" }

# 5. Check if Intermediate CA is in Intermediate store:
Get-ChildItem Cert:\LocalMachine\CA | Where-Object { $_.Subject -like "*Issuing CA*" }

# 6. Check chain using certutil:
certutil -chain THUMBPRINT
# Shows full chain: Root CA → Intermediate CA → Server Cert
# Flags issues with each link

# 7. From another machine — verify chain to server:
certutil -verify -CTL * https://server01.contoso.com
# Or use browser: navigate to site → click padlock → "Certificate is not valid" → "Details" tab → "Certificate Path"

# 8. Check if AD CS published intermediate cert:
Get-ChildItem "Cert:\LocalMachine\My" | Where-Object { $_.Issuer -like "*CA*" }

# 9. Cross-check on client:
certutil -verify -CTL * -server server01.contoso.com

# 10. Check AD-integrated CA certificate replication:
repadmin /replsummary
# Check if CA DC has the latest CA certificate

# 11. Check SCP (Schema Container) and CA certificate publication:
certutil -config - -ping
certutil -view -restrict "cn=NTDS Certificate" -out certificate

# 12. Windows Event Logs:
Get-WinEvent -FilterHashtable @{LogName='System'; ID=36887,36888,36889,36890,36891,36892}
# 36887: Certificate is not yet valid
# 36888: Certificate has expired
# 36889: Certificate does not contain server authentication EKU
# 36890: Certificate has unknown revocation status
# 36891: Certificate chain truncated (CHAIN_UNTRUSTED)
# 36892: Issuer does not match cert
Get-WinEvent -FilterHashtable @{LogName='Application'; Source='Schannel'} -MaxEvents 30
```

### Troubleshooting:

```
STEP 1: IDENTIFY WHICH LINK IS BROKEN
  ├── On the SERVER: Check server's cert chain (certutil -chain THUMBPRINT)
  │   ├── Does server have Intermediate CA cert in its store?
  │   ├── Does server have Root CA cert in its store?
  │   └── Chain status: NotRoot, Untrusted, PartialChain?
  ├── On the CLIENT: Test chain building
  │   ├── Does client have Root CA in Trusted Root store?
  │   ├── Does client have Intermediate CA in Intermediate store?
  │   └── Chain: "unable to get issuer certificate" = Intermediate missing
  │         "not trusted" = Root CA not in Trusted Root

STEP 2: FIX ROOT CA TRUST (Most Common)
  ├── Root CA not distributed to client machines?
  │   ├── Check GPO: Computer Config → Windows Settings → Security Settings →
  │   │   Public Key Policies → Trusted Root Certification Authorities
  │   │   → Should contain Root CA certificate
  │   ├── Verify GPO is applied: gpresult /r | findstr "Trusted Root"
  │   ├── Manually install Root CA: certutil -addstore Root rootca.cer
  │   └── Fix GPO, force gupdate /force, verify
  └── If Root CA was added manually (not via GPO) → risk of inconsistency

STEP 3: FIX INTERMEDIATE CA (Second Most Common)
  ├── Server didn't install Intermediate CA certificate
  ├── IIS might have sent only leaf cert without chain
  │   ├── In IIS: Server Certificates → check if chain is complete
  │   ├── Complete chain: Root CA + Intermediate CA + Server Cert (all on server)
  │   └── Or: Server sends chain (cert + intermediate), client has Root CA
  ├── Intermediate CA cert missing from server's "CA" store:
  │   ├── certutil -addstore CA intermediate.cer
  │   └── IIS will then send full chain
  └── Client missing Intermediate? Fix Root CA GPO → Intermediate is auto-retrieved
     (IF client can reach CA), otherwise must install Intermediate on client

STEP 4: FIX AD CS-SPECIFIC CHAIN ISSUES
  ├── CA certificate not replicated: repadmin /replsummary
  ├── CA needs re-issue (if CA cert expired or revoked)
  ├── Check SCP in AD: certutil -config - -ping
  ├── Check AD CS replication: certutil -view -restrict "Certificate" -out certificate
  ├── Force CA cert renewal (if needed): certutil -RR
  └── Restart AD CS services

STEP 5: VERIFY
  ├── From client: openssl s_client -connect server:443 (Verify return code: 0)
  ├── Browser: Padlock icon, no warnings
  ├── Application: Successfully connects
  └── Event logs: No Schannel errors

STEP 6: PREVENT RECURRENCE
  ├── Always install full certificate chain on servers
  ├── Use GPO to distribute Root CA to all domain computers
  ├── Use AD CS auto-enrollment for intermediate certs
  ├── Monitor chain validity
  └── Document: CA hierarchy documented and accessible to all admins
```

### Common "Chain Broken" Patterns:

| Error Message / Behavior | Root Cause | Fix |
|-------------------------|-----------|-----|
| "unable to get local issuer certificate" (OpenSSL) | Intermediate CA missing on server OR client | Install Intermediate CA on server |
| "not trusted" / NET::ERR_CERT_AUTHORITY_INVALID (Chrome) | Root CA not in client's Trusted Root store | Distribute Root CA via GPO |
| "certificate chain is incomplete" (IIS) | Server not sending intermediate | Install Intermediate in server's CA store |
| "chain is untrusted" (Windows) | Root or Intermediate not in trust stores | Check trust stores, fix GPO |
| Winhttp/WinInet errors | System proxy intercepting TLS, missing corporate root CA | Install corporate root CA via GPO |

---

## SCENARIO 3: Private Key Missing

### What Happens:
- Certificate exists in certificate store, but the corresponding private key is gone, inaccessible, or corrupted
- Certificate shows as having no private key
- Services cannot start (IIS, Exchange, SMTP) because they need the private key for TLS

### Detection:

```powershell
# 1. Check if certificate has private key
Get-ChildItem Cert:\LocalMachine\My | 
  Select-Object Subject, NotAfter, 
    @{Name='HasPrivateKey';Expression={$_.HasPrivateKey}},
    @{Name='PrivateKeyStatus';Expression={
      if ($_.HasPrivateKey) { "Private Key Present" } 
      else { "NO PRIVATE KEY" }
    }} | Format-Table -AutoSize

# 2. Check specific cert:
$cert = Get-ChildItem Cert:\LocalMachine\My\THUMBPRINT
$cert.PrivateKey
# If throws error or returns nothing → private key missing

# 3. Check if private key file exists:
# (Check Microsoft Crypto store paths)
Get-ChildItem "C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys\" |
  Where-Object { $_.Name -match $cert.PrivateKey.CspKeyContainerInfo.UniqueKeyContainerName }

# 4. For ECDSA/PKE certs (Windows CNG):
$cert | Get-CertificatePolicy

# 5. IIS check:
Get-ChildItem IIS:\SslBindings\ | Select-Object IP, Port, CertificateThumbprint
# If cert is bound but private key gone → site cannot start HTTPS

# 6. Event logs:
Get-WinEvent -FilterHashtable @{LogName='System'; ID=36882,36889,36896,36897}
# 36896: Private key does not exist or is invalid
# 36897: Private key is corrupted or cannot be used
# 36898: Private key is not exportable (but needed)

# 7. CertUtil chain with private key check:
certutil -store My "subject" -compact
# Shows if private key is available

# 8. Check cert with private key:
certutil -verifykeys THUMBPRINT

# 9. Exchange check:
Get-ExchangeCertificate | Where-Object { $_.PrivateKeyStatus -ne 'Valid' }
```

### Troubleshooting:

```
STEP 1: CONFIRM PRIVATE KEY IS MISSING
  ├── HasPrivateKey = False in PowerShell
  ├── certutil shows "No private key"
  ├── IIS binding fails with "Cannot find SSL certificate private key"
  └── Service fails to start (Event ID 36896, 36897)

STEP 2: DETERMINE IF CERTIFICATE WAS EXPORTED WITHOUT PRIVATE KEY
  ├── Check: Was a .cer file (without .pfx/.p7b) installed?
  ├── Compare: Certificate is present but marked "No Private Key"
  ├── Check backup: Is .pfx/.p7b backup available?
  └── Check: Was the cert in a different certificate store that got cleared?

STEP 3: RECOVERY OPTIONS
  ├── Option A: If .pfx backup exists:
  │   ├── Import: certutil -importpfx My backup.pfx /pfxpassword:secret
  │   └── Bind to service/IIS
  ├── Option B: If no backup, reissue certificate:
  │   ├── Generate new CSR (must generate new key pair too!)
  │   ├── Submit to CA for reissue
  │   ├── Install new certificate with new private key
  │   └── Bind to service/IIS/Exchange/etc.
  ├── Option C: If original key was on HSM and HSM failed:
  │   └── Contact HSM vendor; use key recovery procedure
  ├── Option D: If key is in Windows Certificate Store but permissions wrong:
  │   ├── Private key permissions: Right-click cert → All Tasks → Manage Private Keys
  │   ├── Ensure service account has Read/Read & Execute permission
  │   └── Typically: NETWORK SERVICE, IIS APPPOOL\AppPoolName, or specific service account
  │
  └── Option E: Use certutil repair:
      ├── certutil -repairstore My THUMBPRINT
      └── (This only works if the key is there but not recognized)

STEP 4: REBIND / RESTART
  ├── Re-bind certificate to service
  │   ├── IIS: New SSL binding with new thumbprint
  │   ├── Exchange: Enable-ExchangeCertificate
  │   ├── Remote Desktop: Configure RDP TLS certificate
  │   └── Any other service: Reconfigure with new cert
  ├── Restart service to pick up new cert
  └── Verify: TLS handshake works, service is running

STEP 5: PREVENT RECURRENCE
  ├── ALWAYS export .pfx (with private key) when creating CSR manually
  ├── Store .pfx securely (encrypted, in secure location)
  ├── Document thumbprint, private key location, password
  ├── Use HSM for critical certificates
  ├── Use AD CS auto-enrollment (private key generated on server, stays in machine store)
  ├── Never install .cer files (no private key) for production services
  └── Backup: Regular certificate store backups
```

---

## SCENARIO 4: Wrong Certificate Installed

### What Happens:
- Certificate is installed but doesn't match what the service expects
- Wrong CN (Common Name), wrong SAN (Subject Alternative Name), wrong EKU (Enhanced Key Usage), or wrong certificate entirely
- Service starts but TLS/SSL handshake fails because server presents wrong cert

### Detection:

```powershell
# 1. Check what cert is bound to service:
# IIS:
Get-ChildItem IIS:\SslBindings\ | Select-Object IP, Port, CertificateThumbprint
Get-WebBinding | Where-Object { $_.protocol -eq "https" }

# 2. Check the CN/SAN of installed cert:
Get-ChildItem Cert:\LocalMachine\My\THUMB | 
  Select-Object Subject, 
    @{Name='SANs';Expression={
      $ext = $cert.Extensions | Where-Object { $_.Oid -eq '2.5.29.17' }
      $ext.Format(0)
    }},
    EnhancedKeyUsageList, NotAfter, Issuer

# 3. Check what cert the server presents externally:
openssl s_client -connect server01.contoso.com:443 -servername server01.contoso.com 2>/dev/null |
  openssl x509 -noout -subject -issuer -ext subjectAltName

# 4. Compare expected vs actual:
# Expected: CN=server01.contoso.com, SAN includes server01.contoso.com
# Actual (from openssl): CN=server02.contoso.com or SAN=wrong.domain.com → MISMATCH

# 5. Check if SAN matches hostname client is connecting to:
Resolve-DnsName server01.contoso.com | Select-Object IPAddress
# Connect to that IP on 443:
openssl s_client -connect 192.168.1.10:443 -servername server01.contoso.com
# If cert CN doesn't match "server01.contoso.com" → hostname mismatch error

# 6. Exchange-specific check:
Get-ExchangeCertificate | Select-Object Subject, Thumbprint, Services, SANs
# Verify: Services enabled match what's expected

# 7. Check the cert's EKU (Enhanced Key Usage):
$cert = Get-ChildItem Cert:\LocalMachine\My\THUMB
$cert.Extensions | Where-Object { $_.Oid -eq '1.3.6.1.5.5.7.3.1' }  # Server Auth
$cert.Extensions | Where-Object { $_.Oid -eq '1.3.6.1.5.5.7.3.3' }  # Code Signing
$cert.Extensions | Where-Object { $_.Oid -eq '1.3.6.1.5.5.7.3.4' }  # Email

# 8. Check cert chain in browser:
# Click padlock → Certificate → verify CN and SAN

# 9. Event logs:
Get-WinEvent -FilterHashtable @{LogName='Application'; Source='Schannel'} | 
  Where-Object { $_.Message -like '*certificate*' }
# Look for: "A fatal alert was generated and sent to the remote endpoint"
# "The server certificate name is wrong" or similar
```

### Troubleshooting:

```
STEP 1: IDENTIFY MISMATCH
  ├── What certificate SHOULD be installed? (CN, SAN, EKU)
  ├── What IS installed? (Use openssl or certutil to check)
  ├── Common mismatches:
  │   ├── Wrong CN (server02 instead of server01)
  │   ├── Missing SAN (connecting via FQDN but cert has IP only)
  │   ├── Wrong EKU (cert lacks Server Authentication EKU 1.3.6.1.5.5.7.3.1)
  │   ├── Self-signed cert used instead of CA-issued cert
  │   ├── Old certificate still bound (expired or wrong hostname)
  │   └── Multiple certificates in IIS, wrong one selected in binding

STEP 2: REMOVE WRONG CERTIFICATE
  ├── Remove old/wrong cert from store:
  │   certutil -delstore My THUMBPRINT_OF_WRONG_CERT
  │   Or: Remove-Item Cert:\LocalMachine\My\WRONG_THUMBPRINT
  ├── Remove wrong IIS binding:
  │   Remove-WebBinding -Name "Default Web Site" -Protocol https
  ├── Remove from Exchange:
  │   Disable-ExchangeCertificate -Thumbprint WRONG_THUMB
  └── Remove other service bindings

STEP 3: INSTALL CORRECT CERTIFICATE
  ├── Obtain correct CSR/certificate
  ├── Install: certutil -addstore My correct_cert.cer
  │   Or: Import-PfxCertificate -FilePath correct.pfx -CertStoreLocation Cert:\LocalMachine\My
  ├── Verify: correct CN, SAN, EKU present
  ├── Bind to correct service/IIS/Exchange/etc.
  └── Restart service

STEP 4: VERIFY
  ├── openssl s_client shows correct CN/SAN
  ├── Browser shows padlock with correct certificate
  ├── Application connects successfully
  ├── Event logs clean
  └── SAN covers all required hostnames (wildcard or specific SANs)

STEP 5: PREVENT RECURRENCE
  ├── Maintain a certificate inventory (spreadsheet/CMDB)
  ├── Standardize SANs in CSR templates
  ├── Use SAN wildcard (*.contoso.com) where possible (dev, test, staging)
  ├── Verify SAN before binding
  ├── Use IIS "Server Name Indication" (SNI) for multiple sites
  └── Test after every cert deployment
```

---

## SCENARIO 5: SSL/TLS Handshake Failure

### What Happens:
- Client initiates TLS connection, but handshake fails at some step
- Error messages: "ssl_error_handshake_failure_alert", "ERR_SSL_PROTOCOL_ERROR", "TLS handshake failed"
- Common causes: protocol mismatch, cipher suite incompatibility, certificate chain issues, missing DH parameters, client/server TLS version mismatch

### Detection:

```powershell
# 1. Test TLS handshake (from client):
Test-NetConnection -ComputerName server01.contoso.com -Port 443
# Basic connectivity check

# 2. Detailed TLS test using OpenSSL:
# Test TLS 1.2:
openssl s_client -connect server01.contoso.com:443 -tls1_2 -servername server01.contoso.com
# Test TLS 1.3:
openssl s_client -connect server01.contoso.com:443 -tls1_3
# Test all protocols:
for proto in tls1 tls1_1 tls1_2 tls1_3; do
  echo "=== $proto ==="
  echo | openssl s_client -connect server01.contoso.com:443 -$proto 2>&1 | grep -E "Cipher|Verify|Protocol|Alert"
done

# 3. Windows Schannel test (from server):
Test-TlsConnection -RemoteServer server01.contoso.com -Port 443 -SSL
# Or:
Test-TlsConnection -RemoteServer server01.contoso.com -Port 443 -UseTls12

# 4. Check protocol enabled on server (IIS):
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server" -ErrorAction SilentlyContinue
# Check Enabled and DisabledByDefault values

# 5. Check cipher suites (Windows):
Get-TlsCipherSuite | Select-Object Name | Sort-Object Name
# Or:
ssdiag /tls /show

# 6. Check cipher negotiation:
openssl s_client -connect server01.contoso.com:443 2>&1 | grep -E "Cipher is|TLS handshake"

# 7. Event logs:
Get-WinEvent -FilterHashtable @{LogName='System'; ID=36882,36886,36887,36889,36890,36891,36892,36896,36897}
Get-WinEvent -FilterHashtable @{LogName='Application'; Source='Schannel'} -MaxEvents 30
Get-WinEvent -FilterHashtable @{LogName='Application'; Source='IIS'} -MaxEvents 30

# 8. Check if certificate is accessible to service:
# (Service account vs SYSTEM context)
Get-ChildItem Cert:\LocalMachine\My\THUMB | 
  Select-Object -ExpandProperty PrivateKey | Select-Object *

# 9. Check Schannel event logs (if enabled):
wevtutil sl System /q:*[System[(EventID=36882)]]

# 10. Detailed Windows TLS troubleshooting:
# Enable Schannel logging:
reg add "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL" /v "Diagnostics" /t REG_DWORD /d 1 /f
# Set EventLogging for Schannel:
wevtutil sl System /q:"*[System[(EventID=36886)]]" /e

# 11. Check for DH key issues:
# (for DHE cipher suites)
reg query "HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\KeyExchangeAlgorithms" /s
# Check if DH key length is sufficient
```

### Troubleshooting Steps:

```
STEP 1: DETERMINE TLS VERSION MISMATCH
  ├── Client tries TLS 1.3, server only supports TLS 1.2?
  ├── Server only allows TLS 1.0/1.1 (deprecated/disabled)?
  ├── Check server protocol settings:
  │   ├── HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\
  │   │   ├── TLS 1.0\Server (Enabled/DisabledByDefault)
  │   │   ├── TLS 1.1\Server
  │   │   └── TLS 1.2\Server
  ├── Check client protocol settings (same registry path)
  └── FIX: Enable matching protocol on both sides (prefer TLS 1.2, 1.3)

STEP 2: DETERMINE CIPHER SUITE INCOMPATIBILITY
  ├── Server supports only certain cipher suites (via Group Policy or registry)
  ├── Client doesn't support those cipher suites
  ├── Check server cipher suites:
  │   Get-TlsCipherSuite | Select-Object Name, Description
  │   Sch-UseStrongCrypto registry settings
  ├── Check client cipher suites
  └── FIX: Enable common cipher suites, or adjust server/client settings

STEP 3: CHECK CERTIFICATE CHAIN
  ├── Is certificate valid? (Not expired, not revoked)
  ├── Is chain complete? (Root CA trusted, Intermediate present)
  ├── Does server send full chain?
  │   └── openssl s_client -connect server:443 -showcerts → Count certs
  └── FIX: Install Intermediate CA on server, ensure Root CA trust

STEP 4: CHECK PRIVATE KEY ACCESS
  ├── Can service account access private key?
  ├── Is private key encrypted with password?
  ├── Is private key stored in TPM/HSM and unavailable?
  └── FIX: Adjust private key permissions

STEP 5: CHECK FOR PROTOCOL-SPECIFIC ISSUES
  ├── Missing DH parameters (for DHE cipher suites):
  │   └── Server may need DH parameters file
  ├── OCSP stapling issue:
  │   └── Server can't reach OCSP responder → handshake timeout
  │   └── FIX: Enable OCSP stapling or configure responder availability
  ├── SNI (Server Name Indication) mismatch:
  │   └── Server has multiple certs, SNI doesn't match any
  ├── Certificate requires CRL check but CRL unavailable:
  │   └── FIX: Enable "check CRL" or make CRL available
  └── TLS 1.3 requires different certificate properties:
      └── FIX: Ensure cert supports TLS 1.3 requirements

STEP 6: VERIFY
  ├── openssl s_client handshake successful (Cipher shows, Verify OK)
  ├── Browser connects successfully
  ├── Application connects
  └── Event logs: No Schannel/TLS errors
```

---

## SCENARIO 6: Application Cannot Bind Certificate

### What Happens:
- Certificate is valid and installed, but a specific application (IIS, Exchange, SQL, RDP, SMTP) cannot bind it
- Error: "Cannot bind SSL certificate", "No suitable certificate found", "Access is denied"
- Common for: Service account lacks private key permissions, wrong certificate store, or application can't find cert

### Detection:

```powershell
# 1. IIS-specific check:
Get-WebBinding | Where-Object { $_.protocol -eq "https" }
Test-WebBinding -Name "Default Web Site" -Protocol https
Get-ChildItem IIS:\SslBindings\
# Check: Certificate hash exists? Is it the correct one?

# 2. Check if application can access private key:
# (Check private key ACL)
$cert = Get-ChildItem Cert:\LocalMachine\My\THUMBPRINT
$privateKey = $cert.PrivateKey
$privateKey.CspKeyContainerInfo
# Check: UniqueKeyContainerName → path to key file
# Then: Check ACL on that file:
icacls "C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys\<keyfile>"

# 3. Check service account:
# (What account does the application run as?)
Get-WmiObject Win32_Service | Where-Object { $_.Name -eq "W3SVC" } | Select-Object StartName
# (For IIS App Pool)
Get-ItemProperty "IIS:\AppPools\AppPoolName" -Name processModel

# 4. Check if service account has access:
# (Run as service account)
runas /user:"DOMAIN\ServiceAccount" cmd
# Then inside that cmd:
certutil -store My
# If error: "Access denied" → permission issue on private key

# 5. Check IIS Crypto tool (Nartac Software) - recommended:
# (IIS Crypto enables/disables protocols and cipher suites, checks binding)
# Download and run: iiscrypto.exe

# 6. Check Exchange:
Get-ExchangeCertificate | Where-Object { $_.PrivateKeyStatus -ne "Valid" }
Get-ExchangeCertificate | Where-Object { $_.Services -notlike "*IIS*" } 
# If IIS not enabled → services can't use it

# 7. Check RDP certificate:
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" -Name SSLCertificateThumbprint
# Check if that cert exists and has private key

# 8. SQL Server:
# (Check SQL Config Manager → Protocols → Certificates tab)
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\MSSQLXX.MSSQLSERVER\MSSQLServer\SuperSocketNetLib\Certificate"

# 9. Event logs:
Get-WinEvent -FilterHashtable @{LogName='System'; Source='Schannel'}
Get-WinEvent -FilterHashtable @{LogName='Application'; Source='IIS'}
Get-WinEvent -FilterHashtable @{LogName='Application'; Source='Schannel'}
# Look for: "No SSL certificate found", "Could not create SSL certificate"

# 10. Check all certificates on server with their purpose:
Get-ChildItem Cert:\LocalMachine\My | 
  Select-Object Subject, NotAfter, Thumbprint, 
    @{Name='EKU';Expression={
      ($_.Extensions | Where-Object { $_.Oid -eq '1.3.6.1.5.5.7.3.1' }).Format(0)
    }},
    @{Name='KeyLength';Expression={ $_.PublicKey.Key.KeySize }},
    @{Name='HasPrivKey';Expression={ $_.HasPrivateKey }} |
  Format-Table -AutoSize

# 11. Try binding manually:
netsh http show sslcert ipport=0.0.0.0:443
# If no result: HTTP.SYS doesn't have any SSL binding
# Add: netsh http add sslcert ipport=0.0.0.0:443 certhash=THUMB appid={GUID}

# 12. Check HTTP.SYS (used by IIS, SQL, etc.):
netsh http show sslcert
# Shows all SSL bindings and their certificates
```

### Troubleshooting:

```
STEP 1: IDENTIFY APPLICATION-SPECIFIC ISSUE
  ├── Which application cannot bind the certificate?
  ├── Check application-specific config (IIS site bindings, Exchange services, RDP settings)
  ├── Check service/application event log for specific errors
  └── Determine: Is cert not found, found but no private key, or found but permission denied?

STEP 2: VERIFY CERTIFICATE IS IN CORRECT STORE
  ├── IIS requires cert in Local Machine\My store (Cert:\LocalMachine\My)
  ├── Exchange requires cert in Local Machine\My
  ├── SQL Server requires cert in Local Machine\My
  ├── Service accounts expect cert in specific store
  ├── NOT in Current User store (IIS runs as SYSTEM, looks in Local Machine)
  └── FIX: Import cert into correct store: 
      Import-PfxCertificate -FilePath cert.pfx -CertStoreLocation Cert:\LocalMachine\My

STEP 3: FIX PRIVATE KEY PERMISSIONS
  ├── Find private key file:
  │   certutil -repairstore My THUMB
  │   → Get PrivateKey.CspKeyContainerInfo.UniqueKeyContainerName
  │   → C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys\<container>
  ├── Check ACL on key file:
  │   icacls <keyfile>
  ├── Add service account with Read permission:
  │   icacls <keyfile> /grant "DOMAIN\ServiceAccount:R"
  │   Or (preferred for IIS): 
  │   Right-click cert → All Tasks → Manage Private Keys → Add service account with Read
  └── For IIS App Pool accounts:
      IIS App Pool\{AppPoolName} needs Read access
      (IIS auto-configures this when binding via IIS Manager; may not when using PowerShell)

STEP 4: FIX APPLICATION CONFIGURATION
  ├── IIS: Add HTTPS binding with correct thumbprint
  ├── Exchange: Enable certificate on needed services
  │   Enable-ExchangeCertificate -Thumbprint THUMB -Services IIS,SMTP,IMAP,POP
  ├── RDP: Set certificate thumbprint in registry
  ├── SQL: Configure in SQL Configuration Manager
  └── Other apps: Reconfigure with correct cert thumbprint/path

STEP 5: RESTART / RELOAD
  ├── IIS: iisreset or Restart-WebAppPool
  ├── Exchange: Restart services
  ├── RDP: Restart TermService
  ├── Other: Restart respective service
  └── Application should now bind successfully

STEP 6: VERIFY
  ├── Application runs correctly
  ├── HTTPS accessible
  ├── No errors in application event log
  └── netsh http show sslcert shows correct binding
```

---

## SCENARIO 7: Auto-Enrollment Failure

### What Happens:
- Certificates configured for auto-enrollment (via GPO) fail to renew/install
- Users/computers don't get expected certificates
- Event IDs logged: 41, 42, 46, 47, 48 (in Certificate Services Client log)
- Common: DC certificates, machine certificates, user certificates not auto-enrolling

### Detection:

```powershell
# 1. Check auto-enrollment event logs:
Get-WinEvent -FilterHashtable @{LogName='Application'; Source='CertificateServicesClient-AutoEnrollment'} -MaxEvents 50
# Key Event IDs:
# 41 = Auto enrollment succeeded
# 42 = Auto enrollment failed
# 44 = Auto enrollment succeeded (NEW in 2012 R2)
# 45 = Auto enrollment failed (NEW in 2012 R2)
# 46 = Auto enrollment succeeded (extended)
# 47 = Auto enrollment failed (extended)

# 2. Check GPO auto-enrollment settings:
gpresult /h C:\gpreport.html
# Check: Computer Config → Policies → Windows Settings → Security Settings → 
# Public Key Policies → Certificate Services → Auto Enrollment
# Is it Enabled? Configured?

# 3. Check auto-enrollment via registry:
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Cryptography\AutoEnrollment" -ErrorAction SilentlyContinue
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Cryptography\AutoEnrollment" -ErrorAction SilentlyContinue
# AEEnabled should be 1 for auto-enrollment
# AERefreshTimeout (how often)

# 4. Force auto-enrollment:
certutil -pulse
# Or: gpupdate /force → triggers auto-enrollment
# Or: certutil -repairstore -user my "*" (repairs/renrolls)

# 5. Check which certificates are configured for auto-enrollment:
certutil -view -restrict "AutoEnrollment" -out serial,notafter,raw.Certificate

# 6. Check certificate templates (must be published in AD):
Get-ChildItem "AD:\Microsoft\\Certification Authorities\\<CAName>\\Certificate Templates\\" 
# Or via certutil:
certutil -view -restrict "Template" -out raw.Certificate

# 7. Check enrollment status:
certutil -view -restrict "RequestType=Pending" -out raw.Request
certutil -view -restrict "RequestType=Approved" -out raw.Certificate

# 8. Check machine certificate:
Get-ChildItem Cert:\LocalMachine\My | Where-Object { $_.Subject -like "*$env:COMPUTERNAME*" }

# 9. Check specific event for error details:
Get-WinEvent -FilterHashtable @{LogName='Application'; ID=42,45,47} | 
  Select-Object TimeCreated, Message

# 10. Check if DC has auto-enrollment (DC certificates):
certutil -view -restrict "RequestType=Pending" -out Subject,RequestType,TimeStamp

# 11. Check certificates via RSoP:
rsop.msc
# Or PowerShell:
Get-GPResultantSetOfPolicy -Report Xml -Path C:\GPO.xml

# 12. Check enterprise PKI health:
certutil -config - -ping  # Check CA responsiveness
certutil -view -restrict "NotAfter<=2026-01-01" -out raw.Certificate  # Find expiring

# 13. Check if CA is responding:
certutil -config DC01.contoso.com\CA01 -ping
certutil -config DC01.contoso.com\CA01 -view -limit 1

# 14. Check template ACLs (enrollment permission):
certutil -TemplateAccess "WebServer"
# Shows which security groups/users can enroll for this template
```

### Troubleshooting:

```
STEP 1: IDENTIFY THE FAILURE
  ├── Run certutil -pulse manually → see error output
  ├── Check Event ID 42/45/47 → read error details
  ├── Common errors:
  │   ├── "Access is denied" → permission to template
  │   ├── "Certificate template not found" → template not published or wrong name
  │   ├── "The computer is not domain joined" → GPO auto-enrollment requires domain join
  │   ├── "Could not contact CA" → CA unreachable, DNS issue, firewall
  │   ├── "Template not configured for auto enrollment" → AE flag not set on template
  │   └── "Insufficient access rights" → template ACL blocks enrollment

STEP 2: CHECK GPO CONFIGURATION
  ├── Verify GPO auto-enrollment setting is enabled (not just configured):
  │   gpresult /h C:\report.html → check "Certificate Services - Auto Enrollment"
  │   Is it: Not Configured? Enabled?
  ├── Check: Computer Config → Policies → Windows Settings → Security Settings →
  │   Public Key Policies → Certificate Services
  ├── Verify GPO is linked to correct OU
  ├── Verify GPO applies to the affected computer (gpresult /r)
  └── If GPO not applied: WMI filter, security filtering, linked GPO broken

STEP 3: CHECK CERTIFICATE TEMPLATE
  ├── Is the template published in AD?
  │   └── ADUC → Issuing Certification Authority → Certificate Templates
  │   └── Or: certutil -view -restrict "Template=WebServer" -out cn
  ├── Is the template configured for auto-enrollment?
  │   └── In AD CS template → General → "Enrollment" → 
  │       "Allow auto-enrollment" must be checked (not just "configured")
  ├── Is the template security ACL allowing the requesting group?
  │   └── Domain Computers / Authenticated Users / specific groups need Enroll/AutoEnroll
  ├── Is the template compatible with the OS version?
  │   └── Windows 2016 template with Win7 client? May have compatibility issues
  └── Is the template expiration date in the past?

STEP 4: CHECK CA AVAILABILITY
  ├── Is CA online and responding?
  │   certutil -config DC01\CA01 -ping
  │   Get-Service CATools, ADCS, CertSvc (depending on version)
  ├── Can client reach CA? (network/firewall)
  ├── Is certificate that the CA uses for signing expired/revoked?
  ├── Check CA database: Is it full? Corrupt?
  └── Check: Windows Certificate Services log on CA server

STEP 5: CHECK AD REPLICATION (For Enterprise Auto-Enrollment)
  ├── Certificate templates are stored in AD → must replicate
  ├── repadmin /replsummary
  ├── If template recently created/modified → force replication
  └── Check client's DC has the updated template

STEP 6: FIX AND VERIFY
  ├── Fix root cause (permissions, template, GPO, CA)
  ├── Force re-enrollment: certutil -pulse
  ├── Or: gpupdate /force (triggers auto-enrollment scheduler)
  ├── Verify: New certificate appears in store
  ├── Check Event ID 41/44 (success)
  └── Verify: Application/service using new cert works

STEP 7: PREVENT RECURRENCE
  ├── Enable auto-enrollment with proper GPO settings
  ├── Publish templates correctly with auto-enrollment flag
  ├── Set proper template ACLs
  ├── Monitor: Event ID 42/45/47 for failures
  ├── Set up alerts for auto-enrollment failures
  └── Regular CA health checks
```

---

## SCENARIO 8: CRL Unavailable

### What Happens:
- Client checks if a certificate is revoked by querying CRL, but CRL cannot be retrieved
- Results: TLS handshake may fail (if hard-required), "certificate is revoked" error, or "unknown" status
- Common causes: CRL publication point unreachable, CRL not published, CA down, network/firewall blocking, OCSP responder down

### Detection:

```powershell
# 1. Find CRL Distribution Points from a certificate:
$cert = Get-ChildItem Cert:\LocalMachine\My\THUMBPRINT
$cert.Extensions | Where-Object { $_.Oid -eq '2.5.29.31' } | 
  ForEach-Object { $_.Format(0) }  # Shows CRL URLs

# Or directly:
certutil -CRLDistributionPoints THUMBPRINT

# 2. Test CRL URL accessibility:
$crlUrls = @("http://crl.contoso.com/crl/contoso.crl", "ldap://CN=Contoso CRL,...")
foreach ($url in $crlUrls) {
  try {
    $response = Invoke-WebRequest -Uri $url -UseBasicParsing -TimeoutSec 10
    Write-Host "$url → $($response.StatusCode)" -ForegroundColor Green
  } catch {
    Write-Host "$url → FAILED: $($_.Exception.Message)" -ForegroundColor Red
  }
}

# 3. Check local cached CRL:
certutil -urlcache CRL *
# Shows cached CRLs and their status

# 4. Force CRL check:
certutil -verify CTL
certutil -verify -urlcache CRL * https://server.contoso.com/certsrv/crl/contoso.crl

# 5. Check OCSP:
certutil -ocsp -url -verify -server DC01.contoso.com THUMBPRINT

# 6. Check if OCSP is configured on cert:
certutil -AuthorityInformationAccess THUMBPRINT

# 7. From external machine:
# Test CRL URL:
Invoke-WebRequest -Uri "http://crl.contoso.com/roots/contoso.crl" -UseBasicParsing
# Should return CRL file (Content-Type: application/pkix-crl)

# 8. Check CA CRL publication:
certutil -config DC01\CA01 -view -restrict "CRLNumber" -out crlnumber,Timestamp,raw.CRL

# 9. Check CRL publication settings:
certutil -getreg CA\CRLPeriod
certutil -getreg CA\CRLOverlapPeriod
certutil -getreg CA\CRLEventLogInterval

# 10. Check CA events:
Get-WinEvent -FilterHashtable @{LogName='Application'; Source='CryptographicServices'} -MaxEvents 30

# 11. Check GDI/CertUtil test:
certutil -verify CTL
certutil -verify -urlcache * -auth small

# 12. Check CRL size and distribution:
certutil -getreg CA\CRLFileStream
# Verify published CRL exists at CRL URL

# 13. DNS check (CRL published via AD-integrated → DNS):
Resolve-DnsName crl.contoso.com
Resolve-DnsName _ldap._tcp.dc._msdcs.contoso.com  # Also for AD CRL publishing

# 14. Check network path to CRL location:
Test-NetConnection -ComputerName crl.contoso.com -Port 80
Test-NetConnection -ComputerName crl.contoso.com -Port 443
Test-NetConnection -ComputerName crl.contoso.com -Port 389  # LDAP CRL
```

### Troubleshooting:

```
STEP 1: DETERMINE CRL UNREACHABILITY
  ├── Identify CRL URLs from certificates:
  │   └── certutil -CRLDistributionPoints THUMBPRINT
  ├── Test each URL:
  │   └── Invoke-WebRequest (for HTTP), ldap command (for LDAP)
  ├── Check if CRL published?
  │   └── On CA: certutil -config <CA> -view -restrict "CRLNumber" -out crlnumber
  │   └── Check CRL file exists at publication point (HTTP directory, LDAP CN)
  └── If CRL URL returns 404/403: CRL not published

STEP 2: FIX PUBLICATION
  ├── CRL not published:
  │   └── Publish: certutil -config DC01\CA01 -publishCRL
  │   └── Check: CRL published and accessible
  ├── CRL publishing schedule? Check:
  │   certutil -getreg CA\CRLPeriod
  │   certutil -getreg CA\CRLPeriodUnits
  │   Default: every 12 hours
  ├── If CRL publication fails:
  │   └── Check Event Log: Application, Cryptographic Services
  │   └── Check: permissions on CRL publication folder (IIS, LDAP)
  │   └── Check: enough disk space
  │   └── Check: CRL publication point directory exists

STEP 3: FIX NETWORK/FIREWALL
  ├── CRL URL hosted on specific server/web share:
  │   └── Can client reach it? (ping, Test-NetConnection port 80/443)
  ├── Firewall blocking CRL access?
  │   └── Check Windows Firewall, network firewall, proxy settings
  ├── If CRL on AD LDS server: Can client reach AD DS DC?
  └── Check: Clients and CA on same site? Cross-site CRL access slow/blocked?

STEP 4: FIX OCSP (If using OCSP)
  ├── OCSP responder unreachable?
  │   └── Check OCSP URL: certutil -AuthorityInformationAccess THUMBPRINT
  │   └── Test: Invoke-WebRequest OCSP URL
  ├── OCSP responder down?
  │   └── Restart OCSP service
  ├── OCSP cache stale?
  │   └── certutil -urlcache OCSP *
  └── Consider: Enable OCSP stapling on server (offloads to server)

STEP 5: TEMPORARY WORKAROUND (while fixing)
  ├── If CRL is critical for TLS, temporarily:
  │   └── Set clients to "Check for CRL" → "Offline" (not recommended long-term)
  │   └── Or: Enable "Revocation checks off" (NOT SECURE, only temporary)
  │   └── Or: Enable "Do not check CRL unless time-critical"
  ├── Registry fix (per machine or via GPO):
  │   HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Windows
  │   "CertSignerRevocation" → 0 (check), 1 (offline), 2 (hard fail)
  └── Restore after fixing CRL availability

STEP 6: VERIFY
  ├── After fix: certutil -verify CTL (should succeed)
  ├── Client can access CRL URL
  ├── TLS handshake succeeds (if was failing due to CRL)
  ├── Event logs: No CRL/cert errors
  └── Monitor: Ongoing CRL availability

STEP 7: PREVENT RECURRENCE
  ├── Host CRL on highly available infrastructure (load balanced, multiple servers)
  ├── Use multiple CRL URLs (HTTP + LDAP) in certificates
  ├── Enable OCSP as backup/alternative
  ├── Monitor CRL publication (scheduled task checking CRL freshness)
  ├── Use OCSP stapling (offloads revocation checking to server)
  ├── Publish CRL to CDP points with redundancy
  └── Alert on CRL staleness (CRL not updated within expected period)
```

---

## 📌 PHASE 5 — MASTER TROUBLESHOOTING FLOW (ALL PKI SCENARIOS)

```
┌─────────────────────────────────────────────────────────────┐
│         UNIVERSAL PKI TROUBLESHOOTING FLOW                  │
│                                                             │
│  Step 1: SYMPTOM IDENTIFICATION                             │
│  ├── What is the error message?                            │
│  ├── Which service/application affected?                   │
│  ├── Which users/machines affected?                        │
│  ├── When did it start? (recent cert renewal? CA change?)  │
│  └── Scope: One server? One site? All users?               │
│                                                             │
│  Step 2: QUICK DIAGNOSTICS (2-5 min)                       │
│  ├── Check certificate validity:                           │
│  │   Get-ChildItem Cert:\LocalMachine\My | Where Expiry    │
│  ├── Test TLS handshake:                                   │
│  │   openssl s_client -connect server:443                   │
│  ├── Check chain:                                          │
│  │   certutil -chain THUMBPRINT                            │
│  ├── Check CA reachable:                                   │
│  │   certutil -config CA -ping                             │
│  ├── Check CRL/OCSP:                                       │
│  │   Check CRL URL accessible?                             │
│  └── Check event logs:                                     │
│      Get-WinEvent System/Application (Schannel/Crypto)      │
│                                                             │
│  Step 3: ISOLATE CAUSE                                     │
│  ├── Certificate expired → RENEW                           │
│  ├── Chain broken → INSTALL MISSING INTERMEDIATE/ROOT       │
│  ├── Private key missing → REISSUE                         │
│  ├── Wrong cert → REPLACE                                  │
│  ├── TLS handshake → CHECK PROTOCOL/CIPHER                 │
│  ├── Cannot bind → CHECK STORE/PERMISSIONS                 │
│  ├── Auto-enrollment → CHECK GPO/TEMPLATE/CA              │
│  └── CRL unavailable → PUBLISH/FIX CRL                     │
│                                                             │
│  Step 4: REMEDIATE                                        │
│  ├── Apply fix                                           │
│  ├── Restart service                                      │
│  ├── Verify fix                                           │
│  └── Document                                              │
│                                                             │
│  Step 5: PREVENT RECURRENCE                                │
│  ├── Monitoring/alerting (30/14/7 day expiry alerts)       │
│  ├── Auto-enrollment enabled & tested                      │
│  ├── Root CA distributed via GPO                           │
│  ├── CA health monitoring                                  │
│  └── Certificate inventory maintained                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 📌 PHASE 6 — POWERCERT: L3 INTERVIEW ANSWER TEMPLATE

When asked "Walk me through how you would troubleshoot PKI/certificate issues in production," use this structure:

```
"I'd follow a systematic approach:

1. GATHER: Identify the symptom — what specific error are users/services seeing?
   What service is affected (IIS, Exchange, AD FS, RDP)? When did it start?

2. VERIFY CERTIFICATE STATE:
   - PowerShell: Get-ChildItem Cert:\LocalMachine\My → check NotAfter, HasPrivateKey
   - certutil -chain <thumbprint> → check chain status
   - openssl s_client → check handshake, cipher, verify chain

3. CHECK COMMON FAILURE POINTS (in order of likelihood):
   a) Certificate expired → Renew/Reissue
   b) Chain incomplete → Install Intermediate/Root CA
   c) Private key permission → Grant service account Read access
   d) Service restart needed → Restart affected service
   e) CRL/OCSP unreachable → Check revocation (network or CRL publication)
   f) Protocol/cipher mismatch → Check TLS settings (Schannel/registry)

4. VERIFY: After fix, test TLS handshake, confirm service works, check event logs

5. PREVENT: Auto-enrollment, monitoring, certificate inventory, GPO distribution of Root CAs

6. For AD CS specifically: Check auto-enrollment GPO, certificate template ACLs,
   CA health (certutil -config -ping), AD replication of templates, and CA event logs."
```

---

## 📌 PHASE 7 — QUICK REFERENCE TABLE: ALL PKI COMMANDS

| Purpose | Command |
|---------|---------|
| List all certs | `Get-ChildItem Cert:\LocalMachine\My` |
| Check expiry | `Get-ChildItem Cert:\LocalMachine\My\|Where{$_.NotAfter-lt(Get-Date).AddDays(30)}` |
| Check chain | `certutil -chain THUMBPRINT` |
| Force auto-enrollment | `certutil -pulse` |
| Import cert | `certutil -addstore My cert.cer` |
| Import PFX | `Import-PfxCertificate -File cert.pfx -CertStoreLocation Cert:\LocalMachine\My` |
| Delete cert | `certutil -delstore My THUMBPRINT` |
| Show CSR | `certutil -view -restrict "RequestType=Pending" -out Subject,RequestType` |
| CA ping | `certutil -config CA01.contoso.com\IssuingCA -ping` |
| CA view | `certutil -config CA01.contoso.com\IssuingCA -view -limit 10` |
| CRL check | `certutil -urlcache CRL *` |
| CRL publish | `certutil -config CA01 -publishCRL` |
| OCSP check | `certutil -ocsp -url -verify -server DC01 THUMBPRINT` |
| Verify CTL | `certutil -verify CTL` |
| Distribution points | `certutil -CRLDistributionPoints THUMBPRINT` |
| AIA | `certutil -AuthorityInformationAccess THUMBPRINT` |
| Repair store | `certutil -repairstore My "*"` |
| CA registry config | `certutil -getreg CA\CRL*` |
| GPO cert policy | `gpresult /r \| findstr "Certificate"` |
| Test TLS | `Test-TlsConnection -RemoteServer server.contoso.com -Port 443` |
| Check private key ACL | `icacls "C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys\<file>"` |
| HTTP SSL binding | `netsh http show sslcert` |
| Export PFX (with key) | `certutil -exportPFX My THUMB password exported.pfx` |
| Export CRT (no key) | `certutil -exportPFX -pfx SixtyFourTHUMB.cer` |
| Certificate inventory | `Get-ChildItem Cert:\* -Recurse\|Select Subject,NotAfter,Thumbprint` |

---

This covers all 8 scenarios plus the complete PKI chain, dependencies, and enterprise troubleshooting methodology. Want me to go deeper into any specific sub-topic (e.g., AD CS enterprise vs. standalone, HSM setup, cross-forest PKI trust, or specific Exchange/AD FS PKI integration)?