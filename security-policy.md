---
layout: default
title: ResourceOps — security policy
permalink: /security-policy
---

# Security Policy

**Last updated: 2026-06-30**

This document is the umbrella information-security policy for ResourceOps —
a personal household-finance application operated by Ben Jensen for use by
the Jensen household (two adults). The application is single-tenant and is
not offered to third parties or the general public.

This document satisfies Plaid's Security Questionnaire requirements for
governance (Q2), access controls (Q3), infrastructure and network security
(Q6), data encryption (Q7), vulnerability management (Q8), privacy (Q9–Q11),
and incident response. Section numbers below correspond to those SecQ topics.

---

## §1 Governance and ownership

**Information-security owner:** Ben Jensen, Software Developer,
benjensen203@gmail.com. All security decisions, policy updates, and incident
responses flow through this single operator.

**Policy review cadence:** this document is updated whenever the application's
architecture changes (new auth mechanism, new third-party service, new
infrastructure) and at minimum annually. The Git commit history of the public
repository (`BenJensen2/resourceops-public`) provides an auditable review trail.

**Risk posture:** the application stores the financial data of exactly two
individuals (the operator and their spouse) on privately-owned infrastructure.
There is no external user base, no public registration, and no commercial
service relationship. Security controls are sized accordingly — defense-in-depth
for a high-value personal data set without the overhead of enterprise compliance
frameworks that assume a large staff or public-facing systems.

---

## §2 Access controls

### 2a · Application-layer RBAC

The Fastify backend enforces role-based access control at the data layer:

- **`users.is_admin`** — boolean flag; only admins may access membership-management
  and integration-configuration endpoints. Exactly one user (`is_admin = 1`) today.
- **`users.status`** — `pending` | `active` | `revoked`. New sign-ins land in
  `pending` until the admin promotes them. `revoked` blocks all requests at the
  `requireAuth` middleware before any handler executes.
- **`persons.parent_managed`** — household admins may act on behalf of a child
  member for limited operations.

### 2b · Zero-trust network architecture

The application server is **not exposed to the public internet** on its
application port. All access — web browser and native iOS — flows through
**Tailscale**, a WireGuard-based mesh VPN with per-device cryptographic identity.
Joining the tailnet requires SSO + MFA; each device receives its own short-lived
private key. No IP-based allowlisting, no firewall exceptions to the public
internet.

### 2c · Identity and session management

Identity is centralized through **Sign in with Apple** — the single IdP:

- Users authenticate with their Apple ID, which Apple enforces with device-bound
  biometrics (Face ID / Touch ID) + Apple-account MFA.
- The application verifies each identity token against Apple's JWKS on issuance.
- Sessions expire after 30 days; the bearer is validated against the `sessions`
  table on every request.
- Revoking the application from an Apple ID invalidates future tokens
  automatically.

### 2d · Non-human authentication

All service-to-service calls use cryptographic tokens over TLS 1.2+; no shared
passwords:

- **Plaid** — OAuth-style access tokens (`/item/public_token/exchange`). Stored
  AES-256-GCM-encrypted in the database (§4). Never logged.
- **Apple** — RS256 JWTs verified against Apple's JWKS.
- **Anthropic** — API key over TLS 1.2+ against `api.anthropic.com`, in
  SOPS-encrypted environment file only.

### 2e · Operator credentials

| System | Protection |
|---|---|
| SSH to application host | Ed25519 keys only; no password auth. Private key in macOS Keychain. |
| GitHub | SSH + passkey-protected account. |
| Tailscale | SSO + MFA; per-device rotating keys. |
| Plaid dashboard | Unique strong password in Bitwarden + MFA. |
| SOPS secrets | age encryption; recipient private keys on workstation (FileVault) + server only. |

### 2f · Access review

With two active users, access state is reviewed on every login (audit log) and
immediately on any household change. Every write to the application generates an
audit log entry (actor, action, entity, timestamp) surfaced at Analyze → Audit log.

### 2g · Termination and revocation

Setting `users.status = 'revoked'` immediately blocks all API access. The
operator also rotates Plaid credentials, invalidates SOPS recipients, and
removes SSH keys as applicable.

---

## §3 Infrastructure and network security

### 3a · Transport encryption (Q6)

All traffic between clients and the application server traverses **Tailscale
(WireGuard)**. WireGuard uses Curve25519 for key exchange, ChaCha20-Poly1305
for symmetric encryption — cryptographically equivalent to TLS 1.3 or better.

All outbound HTTPS to Plaid, Apple, and Anthropic is over TLS 1.2 minimum,
validated by Node.js's built-in TLS stack.

### 3b · Host isolation

The application server (Debian 12 Linux VM) runs on Proxmox hosted on
privately-owned hardware inside the operator's home network. It has no inbound
public-internet firewall rules beyond what the host router's NAT provides by
default. SSH is available on the tailnet only (no public SSH port).

### 3c · Container isolation

The application runs in a Docker container (`resourceops-resources-app`) with
`restart: unless-stopped`. The data volume (`./data`) is bind-mounted; the
container runs as a non-root user. No privileged mode. No host-network mode.

---

## §4 Data encryption at rest (Q7)

### 4a · Sensitive consumer data (PII) — field-level AES-256-GCM

Plaid **access tokens** — the most sensitive Plaid-originated values; they
provide ongoing access to the user's bank — are encrypted with AES-256-GCM
before being written to the SQLite database:

- Key: 32 bytes, generated with `openssl rand`, stored as `PLAID_AT_REST_KEY`
  in the SOPS-encrypted environment file.
- Wire format on disk: `gcm:<iv_b64>:<ciphertext_b64>:<tag_b64>`.
- Decryption occurs only inside the application process at API-call time.
- If the key is missing or malformed at startup, the server refuses to start.

### 4b · Application secrets

All API keys and shared secrets (`PLAID_CLIENT_ID`, `PLAID_SECRET`,
`PLAID_AT_REST_KEY`, `YNAB_TOKEN`, `ANTHROPIC_API_KEY`, etc.) are stored in
a **SOPS-encrypted** environment file using `age` recipients. The age private
keys are held only on the operator's workstation (FileVault full-disk encrypted)
and the application host. The encrypted file is committed to a private Git
repository; the plaintext `.env` exists only on the application host at mode
`0600 ben:ben`, never in source control.

### 4c · Backups

Nightly automated backups (`backup-sqlite-app.sh` via cron at 02:55 UTC) run
the following pipeline:
1. `VACUUM INTO` — consistent point-in-time snapshot.
2. `PRAGMA integrity_check` — rejects corrupt dumps before they leave the host.
3. `gzip` — compression.
4. `age encrypt` — AES-256 via two age recipients (operator workstation + VM
   probe key). The backup blob is unreadable without the private key.
5. `rsync` to NAS over SSH.
6. Embedded restore probe — the cron pulls the encrypted blob back, decrypts
   with the VM probe key, runs a row-count smoke query, and writes
   `.last-restore-ok.json`. The probe fails the cron (and triggers an email) if
   the restore doesn't match a baseline. Backup verification is automatic.

### 4d · Live database permissions

The SQLite database file is owned `root:ben`, mode `640`. It is readable by the
container (which runs as `ben`) and root only. Write access inside the container
is via the bind-mount; the data directory is writable by the service account.

---

## §5 Vulnerability management (Q8)

### 5a · Operating system patching

`unattended-upgrades` is **enabled and active** on the Debian 12 host. Security
patches are applied automatically within 24–48 hours of publication. The service
runs daily and sends email on failure.

### 5b · End-of-life (EOL) monitoring

The host OS is Debian 12 (Bookworm), with LTS support until June 2028. The Node
runtime is Node 22 LTS (support through April 2027). Docker is on the stable
release channel. Dependencies are tracked in `package.json`; `npm audit` is run
before each production build.

### 5c · Application dependency scanning

`npm audit` runs as part of the TypeScript build step on every deploy. The iOS
app uses Swift Package Manager; SPM dependency updates are reviewed before each
build.

---

## §6 Privacy (Q9–Q11)

### 6a · Privacy policy (Q9)

The full privacy policy is available at:
**https://benjensen2.github.io/resourceops-public/privacy**

It is linked from within the application (iOS: More → Settings → Privacy policy;
web: Settings footer).

### 6b · Consent (Q10)

Users consent to data collection at two explicit moments:

1. **Sign in with Apple** — Apple's own consent dialog (name + email share
   choices) is the first screen any new user sees.
2. **Plaid Link** — Plaid's consent flow (institution → account selection →
   data-sharing disclosure) is required before any bank data is retrieved.

No data is collected before both consent gates are cleared.

### 6c · Data retention and deletion (Q11)

Retention is defined in the [Privacy Policy §6](https://benjensen2.github.io/resourceops-public/privacy).
Key points:

- Transactions and items: indefinite (user's own historical record).
- Sessions: 30 days.
- Audit log: lifetime of database; purgeable by user.

Deletion paths:

- **Self-service in-app:** users can delete any item, transaction, or account
  from the editors.
- **Bank disconnect:** Settings → Connections → Remove calls Plaid's
  `/item/remove`, revokes the access token, and stops future data retrieval.
- **Account deletion:** `DELETE /api/auth/me` revokes all sessions and removes
  the user record. Available in-app under Settings.
- **Email request:** benjensen203@gmail.com — operator fulfills within 72 hours.

---

## §7 Incident response

### Detection

The application audit log records every write. Server logs (Fastify request log)
are local to the host. The nightly backup probe emails on failure, providing a
daily data-integrity check.

### Response steps (single-operator process)

1. **Contain** — revoke affected Plaid access token(s) via `/item/remove` on
   the Plaid dashboard. Rotate `PLAID_CLIENT_ID`/`PLAID_SECRET` immediately.
2. **Investigate** — review audit log in-app (Analyze → Audit log) for
   unauthorized access.
3. **Eradicate** — if host is compromised, rebuild VM from the last clean
   Proxmox snapshot. Restore database from the last verified backup blob.
   Re-issue all secrets via SOPS.
4. **Notify** — inform affected household members within 24 hours of confirming
   a breach. Notify Plaid per their Production Terms within 24 hours.
5. **Document** — write a post-incident summary in the application audit log
   within 72 hours.

### Recovery time / point objectives

- **RPO (max data loss):** ~24 hours (nightly backup cadence).
- **RTO (time to restore):** ~4 hours from a clean Proxmox snapshot + backup
  restore.

---

## §8 Contact

For any security question, vulnerability report, or incident notification,
contact **Ben Jensen** at **benjensen203@gmail.com**.

This is a personal application; there is no security team, bug bounty, or formal
responsible-disclosure program. Reports are reviewed and responded to by the
operator directly.
