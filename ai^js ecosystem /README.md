# ai*js — micro-runtime ecosystem

[![status](https://img.shields.io/badge/ecosystem-active-brightgreen?style=flat-square)](https://github.com/yshengliao?tab=repositories)
[![license](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](#license)
[![typescript](https://img.shields.io/badge/TypeScript-strict-3178C6?style=flat-square&logo=typescript&logoColor=white)](#)
[![runtime](https://img.shields.io/badge/runtime-Node%20%E2%80%A2%20Browser%20%E2%80%A2%20Worker-339933?style=flat-square)](#)
[![made in](https://img.shields.io/badge/made_in-Taiwan-007BC2?style=flat-square)](#)

> 繁體中文 → [README_ZH.md](README_ZH.md)

A family of seven framework-agnostic TypeScript micro-runtimes. Pure-function cores, opt-in subpaths, **zero cross-package dependencies**. Runs in browser, Node, and Web Worker — each package is independently useful.

## Packages

| # | Package | Role | npm |
|---|---|---|---|
| 1 | **aifsmjs** | Strict finite-state machine — pure `step()` lifecycle, opt-in effects, PBT adapter | [0.5.0](https://www.npmjs.com/package/aifsmjs) |
| 2 | **aiecsjs** | Archetype ECS — SoA TypedArray storage, SharedArrayBuffer-ready snapshot transport | [0.5.0](https://www.npmjs.com/package/aiecsjs) |
| 3 | **aibridgejs** | Cross-context RPC bridge — iframe / Flutter / worker adapters, JSON envelope, `AbortSignal`-aware | [0.5.0](https://www.npmjs.com/package/aibridgejs) |
| 4 | **aieventjs** | Tiny event emitter — wildcard `*`, `AbortSignal`, `dispose()` | [0.5.0](https://www.npmjs.com/package/aieventjs) |
| 5 | **aipooljs** | Fixed-size object pool — V8-friendly reset, double-release detection | [0.5.0](https://www.npmjs.com/package/aipooljs) |
| 6 | **aiquadtreejs** | 2D quadtree — per-frame-rebuild broadphase, zero-alloc retrieve | [0.5.0](https://www.npmjs.com/package/aiquadtreejs) |
| 7 | **aiaudiojs** | Audio shell over Howler.js — `AbortSignal` + first-class `crossfade()` | [0.5.0](https://www.npmjs.com/package/aiaudiojs) |

Zero cross-package dependencies — adopt any package in isolation, or compose several by convention. Every release ships with [SLSA Level 3 provenance](https://slsa.dev/provenance/v1) via npm OIDC trusted publisher.

## Standards

- [Conventions](en/conventions.md) — naming, `package.json` shape, documentation contract, CI gates, package skeleton, composition.
- [Security & dependency baseline](en/security.md) — code rules, dependency policy, supply chain.

## License

MIT — see each package's `LICENSE`.
