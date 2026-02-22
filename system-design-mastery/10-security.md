# L9: Security

> Security is cross-cutting — it must be designed in from the start, not bolted on. This covers the security knowledge needed to build systems that aren't easily compromised.

---

## 1. Authentication & Authorization

### Password Security

**Hashing (never store plaintext)**:
- **bcrypt**: adaptive, slow by design (work factor adjustable), includes salt
- **Argon2** (winner of Password Hashing Competition): memory-hard, resistant to GPU/ASIC attacks
  - Argon2id: hybrid of Argon2i (side-channel resistant) and Argon2d (GPU resistant)
- **scrypt**: memory-hard, less studied than Argon2
- **PBKDF2**: NIST standard, not memory-hard (less resistant to GPU cracking)
- **Never use**: MD5, SHA-1, SHA-256 (for passwords) — too fast, GPU can try billions/sec

**Password Policies**:
- NIST 800-63B: length (≥8, ≥64 max), check against breached passwords, no complexity requirements
- Have I Been Pwned API: check if password in known breach lists

### Session Management

- **Session tokens**: cryptographically random, ≥128 bits, stored in HttpOnly, Secure, SameSite=Lax cookie
- **Session fixation attack**: attacker sets victim's session ID before login — regenerate session ID on login
- **Session timeout**: idle timeout (e.g., 30min), absolute timeout (e.g., 24hr)
- **Revocation**: invalidate server-side session on logout (stateful sessions)
- **Stateless (JWT)**: no server-side revocation without blocklist

### JWT (JSON Web Token)

**Structure**: `header.payload.signature`
```json
Header:  {"alg": "RS256", "typ": "JWT"}
Payload: {"sub": "user123", "iat": 1700000000, "exp": 1700003600}
Signature: RS256(base64url(header) + "." + base64url(payload), private_key)
```

**Signing algorithms**:
- **HS256**: HMAC-SHA256, symmetric key — simpler but shared secret
- **RS256**: RSA asymmetric — private key signs, public key verifies
- **ES256**: ECDSA — smaller keys, faster than RSA
- **NEVER use alg=none**: disables signature verification (CVE)

**Common JWT Vulnerabilities**:
- `alg=none` attack: change algorithm to none to bypass signature verification
- Algorithm confusion: server accepts both HS256 and RS256 — attacker uses public key as HMAC secret
- Weak secrets: brute-force HMAC secret if short
- Long expiry + no revocation: stolen token valid until expiry

**Refresh Token Pattern**:
- Short-lived access token (15min) + long-lived refresh token (7 days)
- Access token: stateless JWT
- Refresh token: opaque token stored in DB (can be revoked)
- Rotation: issue new refresh token on each use (detect token theft via reuse detection)

### OAuth 2.0

**Roles**: Resource Owner (user), Client (app), Authorization Server, Resource Server

**Authorization Code Flow** (for web apps):
```
1. Client → Auth Server: redirect with client_id, scope, redirect_uri, state
2. User authenticates with Auth Server
3. Auth Server → Client: authorization code at redirect_uri
4. Client → Auth Server: exchange code for access_token (+ refresh_token)
5. Client → Resource Server: API call with access_token in Authorization header
```

**PKCE (Proof Key for Code Exchange)**:
- Prevents authorization code interception attack for public clients (mobile, SPA)
- Generate random `code_verifier`, hash it to `code_challenge`
- Send `code_challenge` with auth request
- Send `code_verifier` when exchanging code — server verifies hash

**Client Credentials Flow**: server-to-server, no user involved
**Device Code Flow**: for devices with limited input (TV, CLI tools)

### OpenID Connect (OIDC)

- OAuth 2.0 extension for identity (not just authorization)
- Adds **ID token** (JWT) containing user identity claims
- **UserInfo endpoint**: fetch additional user claims
- **Discovery**: `/.well-known/openid-configuration` — metadata about OIDC provider

### SAML (Security Assertion Markup Language)

- XML-based federation protocol (older than OAuth, enterprise use)
- **SP-initiated SSO**: user goes to Service Provider → redirected to IdP → assertion back to SP
- **IdP-initiated SSO**: user goes to IdP, selects application → assertion sent to SP
- Common in: enterprise SSO, Okta, Azure AD, AWS SSO

### Multi-Factor Authentication (MFA)

- **TOTP** (Time-based One-Time Password): RFC 6238, shared secret + current time → 6-digit code
  - Google Authenticator, Authy, Microsoft Authenticator
  - Algorithm: HOTP(secret, floor(unix_time/30))
- **FIDO2/WebAuthn**: hardware security keys or device biometrics
  - **Phishing-resistant**: credential bound to origin (domain), not transmittable
  - Passkeys: device-bound or synced credentials, replace passwords entirely
- **FIDO2 Hardware Keys**: YubiKey, Titan Key — strongest MFA
- **Push notifications**: Duo, Okta Verify — user approves push
- **SMS/email codes**: NIST recommends against (SIM swapping, interception)

### Kerberos

- Enterprise authentication protocol (Active Directory)
- Ticket-based: Ticket-Granting Ticket (TGT) from KDC, service tickets from TGT
- SSO within domain: authenticate once, access all services with service tickets
- Not used in modern web apps but critical in corporate environments (Windows, SQL Server, etc.)

### Zero Trust Architecture

**Principles**:
1. Never trust, always verify (even internal traffic)
2. Assume breach (design for compromise)
3. Least privilege access
4. Verify explicitly (identity, device health, location)

**Components**:
- **Identity Provider** (IdP): Okta, Azure AD
- **Device trust**: MDM (Mobile Device Management), Endpoint Detection & Response
- **Micro-segmentation**: network policies per workload
- **Continuous authorization**: re-verify on every request
- **Implementations**: BeyondCorp (Google), Cloudflare Access, Zscaler ZPA

---

## 2. Cryptography

### Symmetric Encryption

**AES (Advanced Encryption Standard)**:
- Block cipher, 128-bit blocks, 128/192/256-bit keys
- **AES-GCM**: authenticated encryption — encrypts AND integrity-checks (most common mode)
  - IV (nonce): 96-bit random, unique per message — NEVER reuse with same key
  - Authentication tag: 128-bit MAC
- **AES-CBC**: older, requires separate MAC (vulnerable to padding oracle if not authenticated)
- **AES-CTR**: stream mode, parallelizable but no built-in authentication

**ChaCha20-Poly1305**:
- Stream cipher + MAC
- Faster than AES on systems without AES hardware acceleration
- Used in: TLS 1.3, WireGuard, Android, mobile

### Asymmetric Encryption

**RSA**:
- Key generation: two large primes p, q → n=pq, choose e coprime to φ(n), d=e⁻¹ mod φ(n)
- Encrypt: c = m^e mod n; Decrypt: m = c^d mod n
- Key sizes: 2048-bit minimum, 3072+ recommended for new systems
- **RSA-OAEP**: padding scheme for encryption security
- **RSA-PSS**: padding scheme for signature security
- **Weakness**: quantum computers (Shor's algorithm breaks RSA)

**ECC (Elliptic Curve Cryptography)**:
- Smaller keys, faster operations vs RSA (256-bit ECC ≈ 3072-bit RSA)
- Curves: P-256, P-384 (NIST), Curve25519 (Bernstein, preferred)
- **ECDSA**: digital signatures (used in Bitcoin, TLS)
- **EdDSA (Ed25519)**: deterministic, faster, safer than ECDSA

### Hash Functions

- **SHA-256**: 256-bit output, collision-resistant, widely used
- **SHA-3** (Keccak): different construction than SHA-2, quantum-safe
- **BLAKE3**: fast, parallel, used in Sigstore, Bao
- **MD5, SHA-1**: broken — collisions found, don't use for security

**Properties**:
- **Pre-image resistance**: given hash H, can't find M such that hash(M)=H
- **Second pre-image resistance**: given M₁, can't find M₂ with same hash
- **Collision resistance**: can't find any M₁≠M₂ with hash(M₁)=hash(M₂)

### MACs and Signatures

**HMAC** (Hash-based MAC): HMAC(key, message) = hash(key⊕opad || hash(key⊕ipad || message))
- Used for: JWT HS256, request signing, cookie integrity

**Digital Signature**:
- Sign: sig = private_key_operation(hash(message))
- Verify: public_key_operation(sig) == hash(message)
- Provides: authentication + integrity + non-repudiation

### Key Exchange

**Diffie-Hellman (DH)**:
- Alice: a=random, A=g^a mod p
- Bob: b=random, B=g^b mod p
- Shared secret: A^b = B^a = g^(ab) mod p
- No need to transmit shared secret

**ECDH**: DH on elliptic curves (smaller keys, faster)

**Perfect Forward Secrecy (PFS)**:
- Use ephemeral DH keys for each session
- Even if long-term private key compromised, past sessions safe
- TLS 1.3 mandates PFS (no RSA key exchange)

### PKI (Public Key Infrastructure)

**Certificate structure (X.509)**:
- Subject, issuer, validity period, public key, signature algorithm, extensions (SAN, keyUsage)
- **SAN (Subject Alternative Names)**: hostnames/IPs the cert is valid for (replaces CN)

**Certificate Authority chain**:
- Root CA: offline, self-signed, in browser/OS trust store
- Intermediate CA: online, signed by Root
- Leaf cert: signed by Intermediate, for your server

**Certificate Transparency (CT)**:
- All certs must be logged in public CT logs
- Browsers reject certs not in CT log
- Detects misissuance: if CA issues cert for your domain without you knowing, you see it in CT log

**OCSP (Online Certificate Status Protocol)**: check if cert is revoked in real-time
**CRL (Certificate Revocation List)**: downloadable list of revoked certs
**OCSP Stapling**: server includes OCSP response in TLS handshake (reduces client latency)

### Key Management

**HSM (Hardware Security Module)**:
- Physical device that generates, stores, uses keys
- Keys never leave HSM in plaintext
- FIPS 140-2 Level 3/4 certified
- Used by: CAs, payment systems, cloud KMS backends

**AWS KMS**:
- Managed key management service
- Customer Managed Keys (CMK): you control lifecycle
- Envelope encryption: encrypt data with data key, encrypt data key with KMS
- Automatic key rotation

**HashiCorp Vault**:
- Open-source secrets management
- Dynamic secrets: generate short-lived DB credentials on demand
- PKI secrets engine: generate short-lived certs
- Seal/Unseal: master key distributed using Shamir's Secret Sharing

---

## 3. Application Security

### OWASP Top 10 (2021)

**A01 Broken Access Control**: users accessing resources they shouldn't
- Fix: server-side authorization checks, deny by default

**A02 Cryptographic Failures**: sensitive data exposed due to weak/missing encryption
- Fix: TLS everywhere, strong algorithms, proper key management

**A03 Injection**: SQL injection, LDAP injection, OS command injection
- Fix: parameterized queries, input validation, ORMs

**A04 Insecure Design**: missing threat modeling, insecure patterns
- Fix: security design reviews, threat modeling

**A05 Security Misconfiguration**: default credentials, open S3 buckets, verbose errors
- Fix: security hardening, configuration scanning

**A06 Vulnerable and Outdated Components**: known CVEs in dependencies
- Fix: dependency scanning, update regularly

**A07 Identification and Authentication Failures**: weak passwords, session issues
- Fix: strong MFA, secure session management

**A08 Software and Data Integrity Failures**: insecure deserialization, CI/CD attacks
- Fix: code signing, integrity checks

**A09 Security Logging Failures**: not logging security events
- Fix: comprehensive audit logging, anomaly detection

**A10 SSRF (Server-Side Request Forgery)**: server makes requests to attacker-controlled URLs
- Fix: whitelist allowed URLs/IPs, block internal network access

### SQL Injection Prevention

**Parameterized queries** (ALWAYS use this):
```sql
-- Vulnerable
SELECT * FROM users WHERE id = '" + userId + "'";

-- Safe (parameterized)
SELECT * FROM users WHERE id = ?  -- with userId as parameter
```

**ORMs**: inherently parameterized if used correctly (but raw SQL calls can still be injected)
**Input validation**: whitelist allowed characters, but don't rely on it alone

### XSS (Cross-Site Scripting) Prevention

**Reflected XSS**: attacker's script in URL parameter, reflected in response
**Stored XSS**: attacker's script stored in DB, served to all users
**DOM-based XSS**: client-side code reads from URL, writes to DOM unsafely

**Fixes**:
- **Content Security Policy (CSP)**: browser only executes scripts from allowed sources
- **Output encoding**: escape < > " ' & when rendering HTML
- **HttpOnly cookies**: JavaScript can't access session cookie
- **Frameworks**: React, Vue auto-escape by default (don't use `dangerouslySetInnerHTML`)

### CSRF Prevention

**Attack**: attacker's site makes authenticated request to victim's bank
**Fix**:
- **SameSite cookie attribute**:
  - `SameSite=Strict`: cookie not sent in cross-site requests
  - `SameSite=Lax`: cookie not sent in POST cross-site (default in modern browsers)
- **CSRF tokens**: random token in form, verified server-side
- **Double Submit Cookie**: CSRF token in cookie + header, compare server-side
- Not needed for APIs using `Authorization` header (not auto-sent cross-origin)

### Supply Chain Security

**SBOM (Software Bill of Materials)**:
- Machine-readable inventory of all components
- Formats: SPDX, CycloneDX
- Enables: CVE scanning, license compliance

**SLSA (Supply chain Levels for Software Artifacts)**:
- Framework for build integrity
- Level 1: scripted build
- Level 2: version controlled build
- Level 3: hardened build (isolation, ephemeral environments)
- Level 4: reproducible, two-party reviewed

**Sigstore**:
- Sign artifacts with ephemeral OIDC-tied keys
- Transparency log (Rekor): public append-only record of all signatures
- Tools: cosign (container image signing), gitsign (commit signing)

**Dependency confusion attack**: attacker publishes public package with same name as internal package
**Typosquatting**: similar package names (requests vs request)
**Fixes**: scope packages (`@company/pkg`), private package mirrors, checksums/digests

---

## 4. Infrastructure Security

### Secrets Management

**Never hardcode secrets**:
- Use environment variables (with caution — visible to all processes)
- Use secrets manager (Vault, AWS Secrets Manager, GCP Secret Manager)
- **Secret rotation**: automatically rotate DB passwords, API keys

**HashiCorp Vault — key features**:
- Dynamic secrets: generate ephemeral DB credentials per service, expire after use
- No static passwords stored anywhere
- Audit log: every secret access logged

### Container Security

**Image security**:
- Scan images for CVEs: Trivy, Snyk, Grype
- Use minimal base images (distroless, Alpine)
- Don't run as root: `USER nonroot` in Dockerfile
- Read-only root filesystem
- Sign images: cosign with Sigstore

**Runtime security**:
- **seccomp**: restrict syscalls a container can make
- **AppArmor / SELinux**: mandatory access control
- **Falco**: runtime threat detection (detects unexpected syscalls, file reads, network connections)
- **Pod Security Standards** (K8s): Baseline, Restricted policies

### Network Security

**WAF (Web Application Firewall)**:
- L7 filtering: block SQLi, XSS, known attack patterns
- AWS WAF, Cloudflare WAF, ModSecurity (open-source)
- Managed rule sets: OWASP CRS, AWS Managed Rules

**DDoS Protection**:
- **Volumetric**: absorb at edge (Cloudflare, AWS Shield Advanced)
- **Protocol attacks**: SYN flood — SYN cookies, rate limiting
- **Application layer**: fingerprint bots, CAPTCHA, JS challenge

**Network Segmentation**:
- Private subnets for databases (no direct internet access)
- Security groups: default deny, explicit allow per service
- VPC peering / PrivateLink for inter-service communication

---

## 5. Advanced Security

### Homomorphic Encryption

- Compute on encrypted data without decrypting
- `Encrypt(a + b) = Encrypt(a) + Encrypt(b)` (homomorphic over addition)
- **FHE (Fully Homomorphic Encryption)**: any computation — extremely slow (1000-10000× overhead)
- **Partially homomorphic**: one operation type (Paillier: addition, RSA: multiplication)
- **Somewhat homomorphic**: limited depth circuits
- **TFHE, CKKS, BFV**: practical FHE schemes
- Use cases: privacy-preserving ML inference, genomics, financial computation

### Secure Multi-Party Computation (MPC)

- Multiple parties jointly compute a function without revealing their private inputs
- **Garbled circuits**: compile function to boolean circuit, encrypt gates
- **Secret sharing**: split secret into N shares (Shamir's Secret Sharing), any K shares reconstruct
- **Oblivious RAM (ORAM)**: access memory without revealing access pattern
- Use cases: privacy-preserving auctions, joint analytics across organizations

### Zero-Knowledge Proofs (ZKP)

- Prove you know a secret without revealing the secret
- **Interactive ZKP**: prover and verifier exchange messages
- **Non-interactive (NIZK)**: single proof string (used in blockchains)
- **zk-SNARK** (Succinct Non-interactive ARgument of Knowledge): tiny proofs, fast verification
- **zk-STARK**: no trusted setup, quantum-resistant (larger proofs)
- Applications: private transactions (Zcash), identity verification, blockchain scaling (zk-rollups)

### Secure Enclaves

- Hardware-protected isolated execution environment
- **Intel SGX (Software Guard Extensions)**: encrypted memory region, code/data isolated from OS
- **ARM TrustZone**: secure world vs normal world on ARM
- **AWS Nitro Enclaves**: isolated EC2 instances with cryptographic attestation
- Use cases: key management, AI inference on sensitive data, credential processing

### Differential Privacy

- Add calibrated noise to query results to protect individual privacy
- **ε (epsilon)**: privacy budget — lower = more private but less accurate
- **Local DP**: noise added at collection (Apple, Google)
- **Central DP**: trusted aggregator adds noise to results
- Used in: iOS keyboard suggestions, Chrome histograms, US Census data

### Post-Quantum Cryptography

**Threat**: Shor's algorithm on quantum computer breaks RSA, ECC in polynomial time
**Timeline**: harvest now, decrypt later attacks happening today

**NIST PQC Standards (2024)**:
- **ML-KEM** (Kyber): key encapsulation, replaces RSA/ECDH
- **ML-DSA** (Dilithium): digital signatures, replaces RSA/ECDSA
- **SLH-DSA** (SPHINCS+): hash-based signatures (conservative, larger sigs)
- **FN-DSA** (FALCON): compact signatures, NTRU lattice-based

**Hybrid schemes**: combine classical + post-quantum (X25519+Kyber in TLS) for transition period

**Hash functions and symmetric encryption** (AES, SHA-2/3): remain secure against quantum (double key size for Grover's quadratic speedup resistance)
