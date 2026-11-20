---
title: "One Path, Two Parsers: 403 Bypasses from Proxy and Backend Disagreement"
date: 2026-11-20
draft: false
tags: ["403-bypass", "path-normalization", "reverse-proxy", "information-disclosure", "methodology"]
summary: "Why a semicolon and a tab can turn a 403 into a 200, how to reason about parser differentials instead of spraying payloads, and how to fix the root cause."
cwe: "CWE-436"
cover:
  image: "/images/path-parsing-differentials-403-bypass.svg"
  alt: "The proxy blocks /app/metrics but forwards /app;%09/metrics, which the app server normalizes back to /app/metrics"
  relative: false
---

## TL;DR

Many deployments block sensitive paths such as `/metrics` at a reverse proxy or gateway, then pass everything else to an application server.
If the proxy and the backend parse the same URL differently, a request can look harmless to the proxy and still reach the blocked handler.
A path like `/app;%09/metrics` is a typical example: the proxy sees a path that matches no deny rule, while the backend strips the extra characters and routes it to `/app/metrics`.

The examples below are illustrative and do not describe a specific vendor.

## Why this keeps happening

An access rule on a path is a promise that *every* component in the chain agrees on what the path is.
In practice, at least three components parse it:

1. A CDN or load balancer.
2. A reverse proxy or API gateway, where the deny rule often lives.
3. The application server and its router.

Each one may decode, normalize and match paths with its own rules.
A 403 bypass is the gap between the component that decides and the component that serves.

## The usual differentials

| Input | Some components see | Others see |
| --- | --- | --- |
| `;` path parameters (`/a;x/b`) | Literal segment `a;x` | Segment `a`, parameter dropped |
| Encoded whitespace (`%09`, `%20`) | Literal character in the segment | Character trimmed or ignored |
| Encoded slash (`%2f`) | One segment `a%2fb` | Two segments `a/b` |
| Dot segments (`/a/./b`, `/a/x/../b`) | Unnormalized path | `/a/b` |
| Case (`/METRICS`) | No match on `/metrics` | Case-insensitive route match |
| Trailing characters (`/metrics/`, `/metrics.json`) | No match on exact rule | Same handler |

Semicolon path parameters come from the URI spec and are honoured by several Java servlet containers.
Many proxies treat them as plain characters.
Combine that with a router that trims whitespace inside a segment, and `/app;%09/metrics` matches no proxy rule while still landing on the metrics handler.

## Testing without spraying

Payload lists work, but reasoning gets you there faster and gives a better report.

1. **Confirm the block is in front.** Compare the 403 response with the backend's normal errors. A different server header, body style or error page often means the proxy answered, not the app.
2. **Find what the backend tolerates.** On an allowed path, test whether `/allowed;x`, `/allowed%09` or `/ALLOWED` still reach the same handler. Each variant that works is a normalization the backend performs.
3. **Apply that normalization to the blocked path.** Insert the tolerated characters into the segment *before* the blocked one, and into the blocked segment itself.
4. **Change the response, not just the status.** A 200 with the real content of the endpoint is the finding. A 200 with a generic page is not.
5. **Report the impact.** Metrics endpoints often expose internal hostnames, route names, versions, queue sizes and error counters. Map what you saw to what an attacker gains, without publishing the data.

## Fixing the root cause

Adding the next payload to a deny list only moves the gap.

- **Normalize once, at the edge.** Have the proxy reject or canonicalize ambiguous input before it applies rules: reject `;` in paths unless needed, reject encoded control characters, and decode then normalize dot segments.
- **Authorize in the application too.** Sensitive handlers such as metrics, health detail and admin routes should require authentication inside the app, not only at the proxy.
- **Bind to a separate interface.** Serve metrics on an internal port or network that the public proxy never forwards to.
- **Test the differential.** Add regression tests that send the variants in the table above through the real proxy and expect a denial.

## Checklist

- [ ] Do I know which component returns the 403?
- [ ] Which normalizations does the backend perform on an allowed path?
- [ ] Did I apply them before and inside the blocked segment?
- [ ] Is the 200 response the real handler's content?
- [ ] Is the sensitive endpoint also protected inside the application?

## Takeaway

A path-based rule is a contract between parsers.
Find the normalization the backend performs and the proxy does not, and you have found the bypass.
For defenders, the durable fix is one canonical form at the edge plus authorization in the handler.
