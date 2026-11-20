---
title: "Hierarchy-Blind Writes: Why Tenant Trees Break Authorization"
date: 2026-11-20
draft: false
tags: ["multi-tenant", "idor", "access-control", "account-takeover", "methodology"]
summary: "Capability checks and scope checks are different controls. How missing scope checks in hierarchical multi-tenant APIs turn an email update into a tenant takeover, and how to hunt and fix the bug class."
cwe: "CWE-639"
cover:
  image: "/images/scope-checks-in-tenant-trees.svg"
  alt: "A low-privileged user rebinds an admin email in another reseller's subtree because the server never checks the target's scope"
  relative: false
---

## TL;DR

Hierarchical multi-tenant platforms have to answer two questions on every write: *may this role do this action?* and *is the target inside the caller's part of the tree?*
APIs often answer the first and skip the second.
When the action is "update a user's email", that gap is a full account takeover, and against an administrator it becomes a tenant takeover.

This post looks at the bug class, how to hunt it, and how to fix it at the design level.
The examples are illustrative and do not describe any specific vendor.

## Background: authorization in a tenant tree

Many B2B SaaS products, especially ones sold through resellers or managed service providers, model customers as a tree instead of a flat list:

```text
Reseller A                  Reseller B
├── Organization A1         ├── Organization B1
│   └── Team A1-x           │   └── Team B1-x
└── Organization A2         └── ...
```

Every write in such a system has to answer two separate questions:

1. **Capability:** may this role perform this action at all? For example, "an organization admin may edit users".
2. **Scope:** is the *target object* inside the subtree this caller is allowed to touch?

Teams usually get the first question right, because it maps to visible UI permissions.
The second is easier to miss, because the UI never offers a target outside your subtree.
The API, however, accepts whatever identifier the client sends.

## Why email is the most valuable field

An account's email is the anchor of its identity.
Password reset, MFA recovery, invitations and most notifications trust it.
In most products, "change another user's email" is the same as "take over another user's account".

Two common design choices make it worse:

- The confirmation link goes to the **new** address. The attacker controls that inbox, so the confirmation step proves nothing about the account owner.
- The **old** address gets no notice, so the owner has no chance to react.

## What the bug looks like

A typical vulnerable request carries the caller's tenant in a header and the target in the body:

```http
POST /api/users/email HTTP/1.1
Host: app.example.com
Authorization: Bearer <token of a low-privileged user in Organization A1>
X-Tenant-Context: <Organization A1>
Content-Type: application/json

{
  "user_id": "<admin user in Organization B1>",
  "email": "attacker@example.net"
}
```

The server checks that the token is valid, that the caller has the "edit user" capability, and that the caller belongs to the tenant in `X-Tenant-Context`.
All three checks pass, because they only describe the caller.
Nothing loads the target user and asks which tenant it lives in.

The attack chain is short:

1. Rebind the target admin's email to an attacker inbox.
2. Confirm the change from that inbox.
3. Request a password reset for the admin, which now arrives at the attacker inbox.
4. Log in as the admin. If the target sits at reseller level, every organization below it is now reachable.

## Hunting approach

This class rarely shows up by fuzzing single requests.
It shows up when you build a tree you control and test every write across its branches.

1. **Build the tree.** Create at least two independent top-level tenants, each with child tenants and users at each level. Record every tenant ID and user ID. Use only accounts you own.
2. **Map writes that take a target in the body.** Path parameters get reviewed often. Body fields such as `user_id`, `tenant_id` or `login` get reviewed less.
3. **Separate context from target.** When the caller's tenant travels in a header and the target in the body, send a target that disagrees with the header and watch which one the server trusts.
4. **Cross every boundary.** Send each write from the lowest role against targets in: the same tenant, the parent, a sibling, and a different top-level subtree.
5. **Check side effects, not status codes.** A `204 No Content` is a hint. The confirmation email in your test inbox, or the changed setting, is the proof.
6. **Read the denials.** When a request fails, the error tells you which check fired. "Permission denied" means a capability check. An error about tenant membership or ancestry means a scope check exists for that endpoint.

## Root cause pattern

The underlying mistake is checking the caller in isolation.
A correct check is a relation between two objects:

```text
allowed(caller, action, target) =
    has_capability(caller.role, action)
    AND caller.tenant IN ancestors_or_self(target.tenant)
```

The vulnerable version drops the second line, or replaces `target.tenant` with a tenant ID the client sent.

## Fixing the class, not the endpoint

- **Resolve the target server-side.** Load the target object, read its tenant from the database, and check that the caller's tenant is that tenant or one of its ancestors. Never trust a tenant ID sent alongside the target.
- **Centralise the scope check.** One function such as `assert_in_subtree(caller, target)`, called by every write, is easier to audit than checks copied per endpoint. A deny-by-default middleware that refuses writes without a resolved target is better still.
- **Treat email change as a sensitive action.** Require step-up authentication, notify the old address with a one-click revert, and block password resets for a short window after an email change.
- **Test the matrix.** Add automated tests that call each write endpoint from each role against targets in sibling and foreign subtrees and expect a denial. New endpoints then fail the build until someone thinks about scope.

## Checklist for hunters

- [ ] Do I control at least two independent branches of the tenant tree?
- [ ] Have I listed every write that takes a user or tenant ID in the body?
- [ ] Did I test each write from the lowest role against a foreign subtree?
- [ ] Does the server trust the context header, the body, or the object it loads?
- [ ] Did I confirm the real side effect (email received, setting changed)?

## Takeaways

Capability checks and scope checks are different controls.
A missing scope check turns any "edit user" feature into a cross-tenant weapon, and email is the field where that hurts most.
In tree-shaped tenancy, the most useful question for builders and hunters alike is: *whose subtree does this target live in, and who decided that?*
