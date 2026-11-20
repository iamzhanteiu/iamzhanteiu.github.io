---
title: "The Button Is Hidden, the Endpoint Is Not: Testing Role Matrices for BFLA"
date: 2026-11-20
draft: false
tags: ["bfla", "access-control", "privilege-escalation", "api-security", "methodology"]
summary: "Narrow operator roles in large consoles are a rich source of broken function level authorization. A repeatable way to build a role matrix, replay actions across roles, and report the gaps."
cwe: "CWE-285"
cover:
  image: "/images/role-matrix-bfla-hidden-endpoints.svg"
  alt: "The console hides Delete device from operators, but a direct DELETE request succeeds"
  relative: false
---

## TL;DR

Large admin consoles offer narrow roles, such as an operator who may run remote sessions but not delete devices.
The console hides the buttons that role should not use.
When the backend does not check the role again, the operator can call the hidden endpoint directly and delete, disable or reconfigure things the role was never granted.
This is broken function level authorization (BFLA), listed as API5 in the OWASP API Security Top 10.

The examples below are illustrative and do not describe a specific vendor.

## Why narrow roles are fragile

Coarse roles such as *admin* and *read-only* are easy to reason about.
Narrow roles are not.
Each one is a cut through hundreds of endpoints, and each endpoint needs its own decision.

Common causes of drift:

- The role was added after most endpoints existed, and only UI-visible actions were reviewed.
- One service checks permissions per action, while an older service only checks "is this user in the tenant?".
- Read and write share a route prefix, and the check only covers `GET`.
- The permission is checked in the frontend before showing a button, and the backend trusts the frontend.

## Build the role matrix first

Before testing anything, write down what each role should do.
The vendor's documentation is the source, because it turns "I think this is wrong" into "this contradicts your access model".

| Action | Admin | Operator | Read-only |
| --- | --- | --- | --- |
| List devices | yes | yes | yes |
| Start remote session | yes | yes | no |
| Delete device | yes | **no** | no |
| Delete agent | yes | **no** | no |
| Edit device policy | yes | **no** | no |

The bold cells are the test cases.

## Collect the endpoints

You need the requests the console makes for each action, including actions your low role cannot reach in the UI.

1. **Record as admin.** Perform every action in the matrix with an admin account and capture the requests in a proxy.
2. **Mine the frontend bundle.** JavaScript bundles often contain route templates for features hidden from some roles.
3. **Note the ID sources.** For each write, find a read the low role *is* allowed to call that returns the IDs you need, such as a device list. That keeps the test realistic: the attacker uses only data their role can see.

## Replay across roles

1. Log in as the low role in a second browser profile or session.
2. Replay each admin write with the low role's session, keeping the body and IDs.
3. Classify each response:

| Response | Meaning |
| --- | --- |
| `403` with a permission error | Check present |
| `404` on a known object | Probably a check that hides existence |
| `2xx` | Candidate finding |
| `400` validation error | Inconclusive: fix the body and retry |

4. For every `2xx`, confirm the side effect from the admin session: the device is gone, the policy changed.

Tools that replay traffic with a second session, such as authorization-testing extensions for intercepting proxies, speed this up.
They do not replace step 4.

## Keep it safe

Destructive verbs are the most valuable findings and the easiest way to cause harm.

- Test only on objects you created for the purpose.
- Prefer reversible actions first, such as disable before delete.
- Stop at one proof per endpoint. Deleting ten devices does not make the report stronger.

## Reporting so it gets fixed

Programs sometimes rate these lower when the impact stays inside the attacker's own tenant.
Make the case concrete:

- Quote the vendor's role definition that the action violates.
- Show the business impact: an operator who deletes agents or policies can switch off monitoring for the whole tenant.
- Group related endpoints only if the program asks for it. Otherwise, report each distinct action, because each needs its own fix.

## Fixing the class

- **Check permissions in the backend on every action**, keyed by an explicit permission such as `device.delete`, not by role names.
- **Generate the check from one source.** A single permission map used by both the UI and the API stops them from drifting apart.
- **Deny by default.** A new endpoint without a declared permission should fail closed.
- **Test the matrix automatically.** For each role and each write, a test asserts the expected allow or deny.

## Checklist

- [ ] Did I write the role matrix from the vendor's documentation?
- [ ] Did I capture admin requests for every action in the matrix?
- [ ] Did I get target IDs from reads the low role is allowed to make?
- [ ] Did I confirm every `2xx` by checking the side effect?
- [ ] Did I test only on objects I own, and stop at one proof per endpoint?

## Takeaway

A hidden button is a UX decision, not an access control.
Map what each role should do, replay what it should not, and the gaps show up quickly.
