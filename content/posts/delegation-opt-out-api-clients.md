---
title: "When \"No Access\" Means \"No UI\": Delegation Opt-Outs and API Clients"
date: 2026-11-20
draft: false
tags: ["multi-tenant", "access-control", "api-clients", "account-takeover", "methodology"]
summary: "Platforms that let a child tenant opt out of parent management often enforce the opt-out in the console but not on machine credentials. How to test the gap, and how to close it."
cwe: "CWE-285"
cover:
  image: "/images/delegation-opt-out-api-clients.svg"
  alt: "The console path runs the self-managed mode check, the API client path does not"
  relative: false
---

## TL;DR

Some multi-tenant platforms let a child organization switch to a self-managed mode, where the parent reseller loses access to its users and data.
That promise is often enforced where people look, in the web console, and not where automation lives, on API client tokens.
If a parent admin can still call "set user password" with a client-credentials token, the opt-out is decoration, and the parent can take over any account in the child tenant.

The examples below are illustrative and do not describe a specific vendor.

## The delegation model

Reseller-style platforms usually grant a parent tenant management rights over its children by default.
To win customers who do not want that, they add a switch on the child tenant, with names such as *self-managed* or *no parent access*.

The switch changes the meaning of the relationship:

| Mode | Parent can see child | Parent can manage child users |
| --- | --- | --- |
| Managed (default) | Yes | Yes |
| Self-managed | Limited, usually billing only | No |

That table is a security promise to the child tenant.
Every endpoint that touches the child's users has to read the mode before it acts.

## Where the promise leaks

There are usually at least two ways into the same backend:

1. **Interactive sessions** from the web console, which call the API with a user cookie or token.
2. **API clients**, which are machine credentials created by an admin and exchanged for a token with the OAuth 2.0 `client_credentials` grant.

Teams often add the mode check where the console flow runs: in the UI, in a backend-for-frontend layer, or in middleware tied to the interactive session.
API client tokens carry the same tenant and roles, but they may enter through a different gateway route or policy.
The check never runs for them.

The result is an inconsistent model: the console hides the child's users, while a script with the parent's client token can still change them.

## The high-impact verbs

Not every leaked action matters equally.
Look first for writes that change who controls an account:

- set or reset a password
- change the login email
- add or remove MFA factors
- change roles, or add a new admin user
- create API clients inside the child tenant

Any of these turns a parent with "no access" into the owner of the child tenant.

## How to test it

1. **Build both sides.** Create a parent tenant and a child tenant you own. Put the child in self-managed mode and confirm in the console that the parent can no longer see the child's users.
2. **Create a parent API client.** Exchange it for a token with the `client_credentials` grant.
3. **Collect target IDs legitimately.** Use IDs you recorded while setting up the child, before the switch. Do not rely on enumeration you are not allowed to do.
4. **Replay account-control writes.** Call each high-impact verb against a child user with the parent's client token, then with the parent's console session. Compare the results.
5. **Prove the effect.** Log in as the child user with the new password, or check that the email changed. A `200 OK` alone is not proof.
6. **Read the vendor's access model.** If the documentation says the parent must not manage users in self-managed tenants, the finding is a broken promise, not a design debate. Quote that line in the report.

## Why the fix is not "add a check here"

Patching the one endpoint leaves the same gap on its neighbours.
The mode is a property of the *target tenant*, so the check belongs where every request learns its target:

- **Enforce in one place.** Resolve the target tenant server-side and evaluate the delegation mode in shared authorization code that every route uses, regardless of how the caller authenticated.
- **Treat credentials equally.** An API client token and a console session for the same admin must get the same answer. Add tests that run the same request with both and expect identical decisions.
- **Deny by default.** When the mode is self-managed, refuse every write from a parent unless the endpoint is on a short, reviewed allowlist such as billing.
- **Re-check on mode change.** When a child switches to self-managed, revoke parent tokens and API clients scoped to it, and log the switch.

## Checklist

- [ ] Does the platform have a mode where a parent loses access to a child?
- [ ] Did I test with both an interactive session and an API client token?
- [ ] Did I try every account-control verb, not only reads?
- [ ] Did I confirm the effect by logging in or checking the changed field?
- [ ] Did I quote the vendor's own access model in the report?

## Takeaway

An access mode is only as strong as the least-checked path into the backend.
When a product offers "no access" to a parent, test it with the credentials automation uses, because that is where the check is most likely missing.
