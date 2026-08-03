# security

Admission control and workload identity. Kyverno policies, Policy Reporter, and
the Zero Trust mesh posture.

**Waves:** 3 and 7 · Absorbed from the former `platform-security` repository (ADR 0010).

## Ownership

| Path | Owner | Contains |
|---|---|---|
| `kyverno/` | `@security` | Admission policies |
| `policy-reporter/` | `@security` | Policy violation reporting |
| `zerotrust/` | `@zerotrust` | STRICT mTLS, AuthorizationPolicy, SPIFFE |

## Why this tree matters more than its size suggests

The platform runs **one cluster with one OIDC provider**, and environments are
namespaces rather than accounts. Everything that would normally be enforced by an
account boundary is enforced here instead.

The sharpest edge: an IRSA trust policy without a `namespace:serviceaccount`
condition is assumable from **any** namespace, including `dev`. A pod in a
developer namespace can then assume a production role. Nothing about the cluster
prevents this — a Kyverno policy does.

That is why admission control is Wave 3, before any workload lands, rather than
a hardening pass afterwards.

## Policy lifecycle

Policies never go straight to `Enforce`.

```
audit  →  observe for one full sprint  →  fix violations  →  enforce
```

1. Land the policy in `Audit` mode.
2. Watch Policy Reporter for a sprint. Real violations, not theoretical ones.
3. Fix what it finds, or amend the policy if it is wrong.
4. Promote to `Enforce` in a separate pull request, so the promotion is
   independently revertible.

A policy promoted to `Enforce` on the same day it was written will break
someone's deploy at an inconvenient hour, and the response will be to add an
exclusion — which is worse than not having the policy.

## Baseline policy set

| Policy | Mode target | Why |
|---|---|---|
| Pod security baseline | Enforce | No privileged containers, no host namespaces |
| Required labels | Enforce | `u25c.io/workstream`, `u25c.io/owner` — FinOps showback depends on it |
| Resource requests and limits | Enforce | An unbounded workload is how Karpenter provisions a surprise |
| IRSA namespace conditioning | Enforce | See above; this is the important one |
| Image registry allowlist | Audit first | Only ECR and approved upstreams |

## Exemptions

An exemption is a pull request against `kyverno/exceptions/`, reviewed by
`@security`, with an expiry date and the issue that will remove it. An exemption
without an expiry is a policy change made quietly.
