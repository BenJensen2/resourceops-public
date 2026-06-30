---
layout: default
title: ResourceOps — privacy policy
permalink: /privacy
---

# ResourceOps — Privacy Policy

**Last updated: 2026-06-29**

ResourceOps is a personal household finance application built and operated
by Ben Jensen for use by the Jensen household. It is not a commercial
service; it is not offered to the public.

## 1. Who runs this app

- **Operator:** Ben Jensen
- **Contact:** benjensen203@gmail.com
- **Users:** Members of the Jensen household only (Ben + Amichi Jensen).
  Account creation is invite-only and requires admin approval before any
  data is visible.

## 2. What data we collect

ResourceOps collects only the data needed to display a household member's
own financial picture:

- **Authentication identifiers** from Sign in with Apple — a stable, opaque
  user ID, and (one time only, on first sign-in) the email address and name
  the user authorizes Apple to share.
- **Bank account metadata + transactions**, when a user connects their bank
  via Plaid: institution name, account name and type, last 4 digits, current
  balance, and transaction history (date, amount, merchant, category).
- **User-entered data** the user types into the app: recurring items,
  budget categories, bill records, notes, and manual transactions.
- **Audit log** of every change made inside the app: who changed what,
  when, and from which screen.

We do **not** collect: precise device location, contacts, photos (except
photos a user explicitly attaches to a bill), microphone audio, advertising
identifiers, browsing history outside the app, or any data from people who
are not signed-in household members.

## 3. How we use the data

We use the collected data only to:
- Show the user their own financial picture (balances, transactions,
  forecasts, decisions).
- Compute derived views (cash-flow summary, category roll-ups, joint
  funding targets).
- Maintain an audit trail of edits so the user can see history.

We do **not** use the data for advertising, profiling for third parties,
training of AI models we do not own, sale, or sharing for any commercial
purpose.

## 4. Where the data lives

- **Application database:** a SQLite file on a private homelab server (a
  virtual machine on hardware owned by the operator), inside the operator's
  home network. The server is reachable only over a private VPN (Tailscale).
- **Bank credentials:** never stored by ResourceOps. When a user connects
  a bank through Plaid, the user authenticates directly with the bank in
  Plaid Link. ResourceOps receives only an opaque Plaid access token,
  which it stores in the same database described above.
- **No third-party processors** beyond the ones listed in §5.

## 5. Third parties we send data to

- **Plaid** — to retrieve the user's own bank account data, as the user
  authorizes per institution. Plaid's privacy policy applies to that
  retrieval (https://plaid.com/legal/).
- **Apple** — Sign in with Apple is verified against Apple's public JWKS;
  no profile data is sent to Apple beyond the identity token verification.
- **Anthropic** — for an optional AI-based payee categorization feature,
  the merchant *name* of an unmapped transaction and a list of category
  *names* the user has defined are sent to the Anthropic API to receive
  a suggested category. No identifying information, account numbers, or
  transaction amounts are sent.

We do **not** send data to ad networks, analytics services, marketing
platforms, or data brokers.

## 6. How long we keep it

- Transactions and items: indefinitely, so the user can see their own
  historical record. The user can delete any item or transaction at any
  time from within the app.
- Audit log: retained for the lifetime of the database so the user can
  review prior changes. The user can manually purge it.
- Sign-in sessions: 30 days, then re-authentication is required.

## 7. User rights

Because this is a household app (not a commercial service), a user can:
- **Read** all of their own data via the app at any time.
- **Edit or delete** any of their own records from the in-app editors.
- **Disconnect** a bank at any time from Settings → Connections, which
  removes the Plaid access token and stops further data retrieval.
- **Delete the account** via DELETE /api/auth/me (also available as a
  Settings action), which revokes all sessions and removes the user record.
- **Request a copy** of all their data by emailing the operator.
- **Request deletion** of their data by emailing the operator.

## 8. Security

- Server access is gated behind a private VPN; the server is not exposed
  to the public internet for application traffic.
- Sign in with Apple identity tokens are verified against Apple's JWKS.
- Plaid access tokens are treated as secrets and never logged.
- Backups are encrypted at rest.

## 9. Children

ResourceOps is used by adult members of the Jensen household. We do not
knowingly collect data from anyone under 13.

## 10. Changes to this policy

If this policy changes, the updated version will be published at the same
URL and the "Last updated" date at the top will be revised.

## 11. Contact

For any privacy question, data request, or deletion request, email
benjensen203@gmail.com.
