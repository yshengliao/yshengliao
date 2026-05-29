# ai*js — 微執行時生態系

[![status](https://img.shields.io/badge/ecosystem-active-brightgreen?style=flat-square)](https://github.com/yshengliao?tab=repositories)
[![license](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](#授權)
[![typescript](https://img.shields.io/badge/TypeScript-strict-3178C6?style=flat-square&logo=typescript&logoColor=white)](#)
[![runtime](https://img.shields.io/badge/runtime-Node%20%E2%80%A2%20Browser%20%E2%80%A2%20Worker-339933?style=flat-square)](#)
[![made in](https://img.shields.io/badge/made_in-Taiwan-007BC2?style=flat-square)](#)

> English → [README.md](README.md)

一組共七個框架無關的 TypeScript 微執行時套件。純函式核心、opt-in 子路徑、**零跨套件依賴**。可在瀏覽器、Node、Web Worker 執行 —— 每個套件皆可獨立使用。

## 套件

| # | 套件 | 功能 | npm |
|---|---|---|---|
| 1 | **aifsmjs** | 嚴格有限狀態機；純 `step()` 生命週期、可選副作用、屬性測試友善 | [0.5.0](https://www.npmjs.com/package/aifsmjs) |
| 2 | **aiecsjs** | Archetype 風格 ECS；SoA TypedArray 儲存、SharedArrayBuffer-ready 快照傳輸 | [0.5.0](https://www.npmjs.com/package/aiecsjs) |
| 3 | **aibridgejs** | 跨執行情境 RPC bridge；iframe / Flutter / worker 轉接層、JSON envelope、`AbortSignal` 一致 | [0.5.0](https://www.npmjs.com/package/aibridgejs) |
| 4 | **aieventjs** | 微型事件發射器；萬用字元 `*`、`AbortSignal`、可釋放 | [0.5.0](https://www.npmjs.com/package/aieventjs) |
| 5 | **aipooljs** | 固定大小物件池；V8 友善 reset、重複釋放偵測 | [0.5.0](https://www.npmjs.com/package/aipooljs) |
| 6 | **aiquadtreejs** | 2D 四元樹；每幀重建型碰撞 broadphase、zero-alloc retrieve | [0.5.0](https://www.npmjs.com/package/aiquadtreejs) |
| 7 | **aiaudiojs** | Howler.js 薄殼；`AbortSignal` 一致化、第一級 `crossfade()` | [0.5.0](https://www.npmjs.com/package/aiaudiojs) |

零跨套件依賴 —— 可單獨採用任一套件，或依慣例組合多個。每個版本皆透過 npm OIDC trusted publisher 附帶 [SLSA Level 3 provenance](https://slsa.dev/provenance/v1)。

## 規範

- [慣例 Conventions](zh/conventions.md) —— 命名、`package.json` 結構、文件契約、CI gate、套件骨架、組合方式。
- [安全與依賴基線](zh/security.md) —— 程式規則、依賴政策、供應鏈。

## 授權

MIT —— 見各套件的 `LICENSE`。
