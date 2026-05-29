# ai*js — 慣例

每個 `ai*js` 套件遵循的契約。建立新 sibling 時以此為檢查清單。

> English → [../en/conventions.md](../en/conventions.md) ｜ 回[索引](../README_ZH.md)

## 命名契約

| 介面 | 慣例 | 理由 |
|---|---|---|
| 工廠函式 | `createX(opts?)`，不用 `class` constructor | 函式式介面；易傳遞、mock、tree-shake。 |
| 釋放 | `dispose()` —— **idempotent**；dispose 後所有公開方法同步拋 `XDisposedError` | 家族一致的 teardown。 |
| 重置 | `reset()` —— 回到初始狀態但不釋放內部 buffer | HMR 友善；與 dispose 是不同概念。 |
| 訂閱 | `on(event, handler, opts?: { signal?, once? }) => () => void` | 回傳 unsubscribe 函式；尊重 `AbortSignal`。 |
| 長時間 / async | `(args, { signal?: AbortSignal })` | 每個長生命週期操作都收 signal。 |
| 領域中立 | 公開 API 與 source 不得出現領域名詞（`player`、`level`、`score`、`sprite`、`collision`…） | 應用領域用語只屬於使用範例，不進 API 介面。 |

## `package.json` 結構

```jsonc
{
  "name": "aiXXXjs",
  "version": "0.5.0",
  "type": "module",
  "sideEffects": false,
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".":         { "types": "./dist/index.d.ts",   "import": "./dist/index.js",   "require": "./dist/index.cjs"   },
    "./feature": { "types": "./dist/feature.d.ts", "import": "./dist/feature.js", "require": "./dist/feature.cjs" }
  },
  "files": ["dist", "README.md", "README_ZHTW.md", "LICENSE", "llms.txt", "llms-full.txt"],
  "engines": { "node": ">=18.0.0" },
  "license": "MIT"
}
```

- `tsup` 雙 ESM + CJS 輸出。
- `sideEffects: false` 讓 bundler 能丟棄未用子路徑。
- 每個 opt-in 能力一個 subpath export；root 保留 stable core。
- **零 `dependencies`。** runtime 零依賴。唯一允許的例外是 *optional* `peerDependency`（如 PBT adapter 的 `fast-check`、audio shell 的 `howler`）。更重的歸 user-land。
- **無 cross-package import。** `aiXXXjs` 不得 `import 'aiYYYjs'`。整合是文件議題，不是依賴。
- `aiecsjs` 的 `files[]` 另含 `api.json`（machine-readable export manifest）。

## 文件契約（AI-readable）

每個套件出貨：

| 檔案 | 用途 |
|---|---|
| `README.md`（英文，canonical） | 快速上手、心智模型、API 參考、FAQ。 |
| `README_ZHTW.md` | 繁體中文鏡像。 |
| `CHANGELOG.md` | Keep-a-Changelog + SemVer。 |
| `STABILITY.md` | 每個 export 的 stability tag + `since` 版本。 |
| `llms.txt` | [llmstxt.org](https://llmstxt.org/) 探索索引。 |
| `llms-full.txt` | 給 LLM 消費的單檔完整 API 參考；由 `scripts/build-llms-full.mjs` 產生。 |
| `api.json` | machine-readable export manifest（有提供者）。 |
| `CONTRIBUTING.md` | 什麼容易進、什麼需討論。 |
| `LICENSE` | MIT。 |

Stability tag 跨家族共用同一套詞彙：**stable**、**experimental**、**internal**、**deprecated**。同一 tag，到哪都同義。

## CI 與品質 gate

`prepublishOnly` 依序跑：

```
typecheck → lint → coverage → build → verify:exports → check:size
```

| Gate | 工具 | 門檻 |
|---|---|---|
| Typecheck | `tsc --noEmit`（`strict + noUncheckedIndexedAccess + exactOptionalPropertyTypes`） | 零錯誤。 |
| Lint | [Biome](https://biomejs.dev/) | 零錯誤。 |
| 測試 | Vitest 行為測試 | ≥95% statements / lines / functions、≥90% branches（部分套件強制 100）。 |
| Build | `tsup` 雙 ESM/CJS + `.d.ts` | — |
| Exports | `scripts/verify-exports.mjs` | 每個 `exports` 條目對得上實際 `dist/` 檔。 |
| Size | `scripts/check-size.mjs` | 每子路徑 gzip 預算；超標擋發布。 |

## 套件骨架

```
aiXXXjs/
├── src/
│   ├── index.ts              # root re-exports
│   └── <feature>/index.ts    # subpath entries
├── test/                     # vitest 行為測試
├── examples/<NN>-<scenario>/ # 可執行範例
├── scripts/
│   ├── verify-exports.mjs    # gate: exports vs dist/
│   ├── check-size.mjs        # gate: 每子路徑 gzip 預算
│   └── build-llms-full.mjs   # generator: README + src → llms-full.txt
├── README.md / README_ZHTW.md
├── CHANGELOG.md
├── STABILITY.md
├── CONTRIBUTING.md
├── llms.txt / llms-full.txt
├── biome.json
├── tsup.config.ts
├── tsconfig.json
├── vitest.config.ts
├── LICENSE
└── package.json
```

`aifsmjs` 是最符合規範的成員 —— 建新套件時以它的 repository 為範本。

## 組合（依慣例，非依賴）

各套件處理正交的關注點、**互不依賴**。透過在邊界傳遞純資料來組合：

- 狀態機掌管高層流程狀態；ECS world 掌管每幀 entity 資料。狀態機觸發 transition → 應用層生成或更新 entity；ECS 觸發條件 → 應用層回饋事件給狀態機。
- 透過 bridge 把 ECS 快照串到沙箱情境（iframe / worker）—— 先序列化成純 JSON；envelope 會丟掉 `Date`、`Map`、`Set`、class prototype。
- bridge 是對 host（父框架、WebView、worker）的副作用通道；emitter 是行程內的 fan-out。

這些組合都不新增 `dependencies` 條目 —— 每個套件維持可獨立發布。
