# ai*js — security & dependency baseline

Every package satisfies these on every release. Enforced by code review.

> 繁體中文 → [../zh/security.md](../zh/security.md) ｜ Back to [index](../README.md)

## Code

- **No `eval`, `new Function`, or other dynamic code execution.** CSP-friendly; the family ships into iframes, WebViews, and CSP-restricted hosts.
- **No hardcoded secrets** — even test fixtures use environment variables.
- **Boundary validation.** Every input from outside the package (postMessage payloads, `JSON.parse`, snapshot adoption) is structurally validated. Binary formats carry magic + version headers.
- **Prototype-pollution awareness.** No `Object.assign({}, untrustedJson)`; prefer explicit field extraction or `structuredClone`, and skip `__proto__` / `constructor` / `prototype` keys when merging untrusted objects.
- **Strict origin / source validation in transports.** The bridge's iframe adapter rejects `targetOrigin: '*'` synchronously at construction, and validates both `event.origin` and `event.source` on every inbound message.

## Dependency policy

- **Zero runtime dependencies by default.** The only allowed exception is an *optional* `peerDependency`, so the relevant adapter can be tree-shaken when unused.
- **No cross-package imports at runtime.** Packages are composed by convention, never by a dependency edge.
- **devDependencies tracked for advisories.** Transitive security fixes are patched promptly and noted in the CHANGELOG.

## Supply chain

- **npm OIDC trusted publisher** with provenance attestation — no long-lived `NPM_TOKEN`. All seven packages publish this way; every release carries [SLSA Level 3 provenance](https://slsa.dev/provenance/v1).
- **Publish gate.** `prepublishOnly` runs the full quality chain before release; a failing gate stops the tag-triggered publish workflow.
- **Reproducible builds.** `tsup` is deterministic given the same lockfile.
