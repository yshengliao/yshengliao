# ysl ｜ Taiwan-based software engineer

[![status](https://img.shields.io/badge/ecosystem-experimental-orange?style=flat-square)](https://github.com/yshengliao?tab=repositories)
[![license](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](#)
[![typescript](https://img.shields.io/badge/TypeScript-strict-3178C6?style=flat-square&logo=typescript&logoColor=white)](#)
[![runtime](https://img.shields.io/badge/runtime-Node%20%E2%80%A2%20Browser%20%E2%80%A2%20Worker-339933?style=flat-square)](#)
[![made in](https://img.shields.io/badge/made_in-Taiwan-007BC2?style=flat-square)](#)

> Software engineer building TypeScript micro-runtimes for gaming, blockchain, and RNG.
> 軟體工程師，駐臺灣；專注遊戲產業、區塊鏈、亂數產生器，並維護一組可組合的 TypeScript 微執行時套件。

---

## `ai*js` — micro-runtime ecosystem ｜ 微執行時生態系

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

Internal architecture notes, learnings, and conventions are archived under [`ai^js ecosystem/`](./ai%5Ejs%20ecosystem%20/) in this repo.

本生態系的內部架構紀錄、開發經驗、規範指引存放於本 repo 的 [`ai^js ecosystem/`](./ai%5Ejs%20ecosystem%20/) 目錄。

---

## Tech stack ｜ 技術棧

TypeScript (strict + `noUncheckedIndexedAccess`), Flutter / Dart, Go, Java / Spring Boot, Python.

TypeScript（嚴格模式 + `noUncheckedIndexedAccess`）、Flutter / Dart、Go、Java / Spring Boot、Python。

## Contact

- ysl — <ysl@sheng.page>
