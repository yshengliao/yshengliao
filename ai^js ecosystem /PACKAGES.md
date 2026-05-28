# `ai*js` — micro-runtime ecosystem ｜ 微執行時生態系

[![status](https://img.shields.io/badge/ecosystem-experimental-orange?style=flat-square)](https://github.com/yshengliao?tab=repositories)
[![license](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](#)
[![typescript](https://img.shields.io/badge/TypeScript-strict-3178C6?style=flat-square&logo=typescript&logoColor=white)](#)
[![runtime](https://img.shields.io/badge/runtime-Node%20%E2%80%A2%20Browser%20%E2%80%A2%20Worker-339933?style=flat-square)](#)
[![made in](https://img.shields.io/badge/made_in-Taiwan-007BC2?style=flat-square)](#)

A family of seven framework-agnostic TypeScript packages. Zero cross-package dependencies, AI-readable docs, and every release ships with SLSA Level 3 provenance via npm OIDC trusted publisher.

一組共七個框架無關的 TypeScript 套件。零跨套件依賴、面向 AI 可讀的文件規格、每個版本皆透過 npm OIDC trusted publisher 附帶 SLSA Level 3 provenance 簽章。

| # | Package | Role (EN) | 功能（繁中） | npm |
|---|---|---|---|---|
| 1 | **aifsmjs** | Strict finite-state machine — pure `step()` lifecycle, opt-in effects, PBT adapter | 嚴格有限狀態機；純 `step()` 生命週期、可選副作用、屬性測試友善 | [0.2.1](https://www.npmjs.com/package/aifsmjs) |
| 2 | **aiecsjs** | Archetype ECS — SoA TypedArray storage, SAB-ready snapshot transport | Archetype 風格 ECS；SoA TypedArray 儲存、SharedArrayBuffer-ready 快照傳輸 | [0.2.1](https://www.npmjs.com/package/aiecsjs) |
| 3 | **aibridgejs** | Cross-context RPC bridge — iframe / Flutter / mock | 跨執行情境 RPC bridge；iframe / Flutter / mock 三轉接層 | [0.2.1](https://www.npmjs.com/package/aibridgejs) |
| 4 | **aieventjs** | Tiny event emitter — wildcard `*` + `AbortSignal` + `dispose()` | 微型事件發射器；萬用字元 `*`、`AbortSignal`、可釋放 | [0.1.1](https://www.npmjs.com/package/aieventjs) |
| 5 | **aipooljs** | Fixed-size object pool — V8-friendly reset, double-release detection | 固定大小物件池；V8 友善 reset、重複釋放偵測 | [0.1.1](https://www.npmjs.com/package/aipooljs) |
| 6 | **aiquadtreejs** | 2D quadtree — per-frame rebuild collision broadphase | 2D 四元樹；每幀重建型碰撞 broadphase | [0.1.1](https://www.npmjs.com/package/aiquadtreejs) |
| 7 | **aiaudiojs** | Audio shell over Howler.js — `AbortSignal` + first-class `crossfade()` | Howler.js 薄殼；`AbortSignal` 一致化、第一級 `crossfade()` | [0.1.1](https://www.npmjs.com/package/aiaudiojs) |

See [README.md](./README.md) for the full architecture / conventions reference and [LEARNINGS.md](./LEARNINGS.md) for per-cycle retrospectives.

完整架構與規範指引見 [README.md](./README.md)；歷代 release cycle 的開發經驗紀錄見 [LEARNINGS.md](./LEARNINGS.md)。
