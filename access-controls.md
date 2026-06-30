---
layout: default
title: ResourceOps — access controls policy
permalink: /access-controls
---

# Access Controls Policy

**Last updated: 2026-06-29**

This document describes the access controls in place for ResourceOps —
a personal household-finance application operated by Ben Jensen for use
by the Jensen household (two adults). The application is single-tenant
and is not offered to third parties.

## Scope

This policy governs access to:
- The application server (Linux virtual machine hosting the Fastify
  backend and SQLite database).
- The application database (SQLite file containing user data, Plaid
  access tokens, and transaction history).
- The application user interface (web app and native iOS app).
- The operator's secret-management chain (SOPS-encrypted secrets, Plaid
  dashboard, source-code repositories, deploy tooling).

## 1 · Defined and documented access control policy

This document is the policy. It is reviewed when the application's
architecture changes (new auth mechanism, new third-party service, etc.)
and at least annually.

The policy is published at the same stable URL as the privacy policy
and is updated in lockstep with the application's actual controls.

## 2 · Role-based access control (RBAC)

The application enforces RBAC at the data layer:

- **`users.is_admin`** — boolean flag granting access to admin-only
  endpoints (membership management, integration configuration). Today
  exactly one user holds `is_admin = 1`.
- **`users.status`** — `pending` | `active` | `revoked`. Only `active`
  users can access protected routes. New sign-ins land in `pending`
  until an admin promotes them. `revoked` users are blocked at the
  request middleware before any handler executes.
- **`persons.parent_managed`** — when set, household admins can act on
  a child member's behalf for limited operations.

These checks are enforced in `requireAuth` / `requireAdmin` middleware
that wraps every non-public route.

## 3 · Zero-trust access architecture

The application's network and authentication design follows zero-trust
principles:

- **No implicit network trust.** The application server is not exposed
  to the public internet on its application port. All access flows
  through Tailscale, a WireGuard-based mesh VPN with per-device
  cryptographic identity. Joining the tailnet requires SSO + MFA at
  the IdP layer; each device receives its own short-lived key.
- **No implicit user trust.** Every API request must present a valid
  Sign-in-with-Apple session bearer. The bearer is verified against
  Apple's JWKS on issuance and against the application's `sessions`
  table on every request. Expired or revoked sessions are rejected at
  the middleware layer.
- **No long-lived ambient credentials.** Sessions expire after 30 days;
  Plaid access tokens are revoked individually when the user
  disconnects a bank (Settings → Connections → Remove).
- **Audit on every write.** A global audit hook records actor, action,
  entity, timestamp, and source page on every successful mutation.
  Surfaced in-application at Analyze → Audit log.

## 4 · Centralized identity and access management

Identity is centralized through **Sign in with Apple** — the single
identity provider for the application:

- Users authenticate with their Apple ID (which Apple enforces with
  device-bound biometrics + Apple-account MFA).
- The application receives a signed identity token and verifies it
  against Apple's JWKS.
- The application never stores an Apple password or any factor used
  to obtain the Apple identity token.
- Account deactivation through Apple (revoking the application from
  the Apple ID) invalidates future identity tokens automatically;
  existing sessions are revoked via `DELETE /api/auth/me` or admin
  action against `users.status`.

## 5 · OAuth tokens and TLS certificates for non-human authentication

All non-human authentication uses cryptographic tokens or certificates;
no passwords are used for service-to-service traffic.

- **Plaid** — long-lived access tokens (OAuth-style) returned from
  `/item/public_token/exchange`. Stored encrypted-at-rest in the
  application database. Never logged. Each request to Plaid is
  authenticated by `PLAID-CLIENT-ID` + `PLAID-SECRET` headers over
  TLS 1.2+ against `production.plaid.com` (or `sandbox.plaid.com`
  during development).
- **Apple** — identity tokens (JWT, RS256) verified against Apple's
  JWKS at `appleid.apple.com/auth/keys`. The application requires the
  token's `aud` claim to match the registered bundle identifier and
  rejects tokens with an `iss` other than `https://appleid.apple.com`.
- **Anthropic** — API key (`ANTHROPIC_API_KEY`) over TLS 1.2+ against
  `api.anthropic.com`. Stored in a SOPS-encrypted environment file
  (see §7). Never logged. Used only for the optional payee-categorization
  feature; no Plaid data is sent — only the merchant *name* of an
  uncategorized transaction.
- **Source-code hosting / package registries** — Personal access tokens
  with least-privilege scopes, stored in the operating-system keychain
  on the operator's workstation. Never committed to source control.

## 6 · Operator-side credential management

The single operator's credentials are protected as follows:

- **SSH** — Ed25519 keys only; no password authentication on any server.
  Private keys are stored in the operator's macOS Keychain. Public keys
  are managed centrally via the deploy toolkit.
- **GitHub** — SSH for `git push`, OAuth-issued personal access tokens
  for `gh` CLI operations, all behind a passkey-protected GitHub
  account.
- **Tailscale** — joined via SSO + MFA. Per-device keys auto-rotated.
- **Plaid dashboard** — protected by a strong unique password held in
  a password manager + MFA.

## 7 · Secrets at rest

Application secrets (Plaid keys, YNAB token, Anthropic key, Sign-in-with-
Apple audience, cross-app shared tokens) are stored in a SOPS-encrypted
environment file. Encryption uses age recipient keys; the private keys
are held only on:

1. The operator's workstation (macOS, FileVault full-disk encryption).
2. The application server (encrypted Linux LVM, locked-down `/root`).

Plaintext exists only as a `.env` file on the application host with
ownership `ben:ben` and mode `0600`. Re-deployment regenerates this
file from the encrypted source; the plaintext file is never committed
to source control and never leaves the host.

## 8 · Access review

Because the user count is two (both adults in the operator's household),
access state is reviewed continuously:

- Every active session is visible to the admin via `/api/auth/users`
  (and the in-app Settings → Admin · Users surface).
- The audit log surfaces every login + every privileged action.
- Any change in household membership triggers an immediate manual
  access review by the operator.

## 9 · Termination / revocation

A user's access can be revoked immediately by setting
`users.status = 'revoked'`, which:

1. Blocks all future requests via the `requireAuth` middleware.
2. Causes the next session lookup to return no row.
3. Surfaces the change in the audit log.

The operator additionally rotates Plaid `client_id`/`secret`,
invalidates SOPS recipient keys, and removes the affected user's
public SSH key from server `authorized_keys` if applicable.

## 10 · Contact

For questions about this policy or to report a concern,
email **benjensen203@gmail.com**.
