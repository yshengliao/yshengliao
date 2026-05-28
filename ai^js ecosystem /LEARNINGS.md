# ai*js — 工程經驗紀錄

按 release 累積的 retrospective。每個 cycle 加一節，舊節不動。寫給未來的我自己、未來的 collaborator、以及未來的 AI 代理人。

新進來的人請先讀 [README.md](README.md) 學「該怎麼做」，再讀本檔學「為什麼那樣做、以前怎麼跌過跤」。

---

## v0.2.0 cycle — 2026-05-28

三套件（[aibridgejs](aibridgejs/) 0.1.3 → 0.2.0、[aifsmjs](aifsmjs/) 0.1.2 → 0.2.0、[aiecsjs](aiecsjs/) 0.1.4 → 0.2.0）同步推進。本輪以**三路並行 CodeReview** 為核心方法論，並產出根目錄 `README.md` 為生態指引。

### 1. 三路並行 CodeReview 的價值

啟動時用 `/code-review`（correctness 主視角）、`/codex:review`（獨立 context 的第二意見）、`/security-review`（安全專責）**同一輪內並行送出**。三個 reviewer 在不同模型、不同 context 下各自走查同一份 diff。

**第一輪三份回報的不重疊度極高**：
- code-review 找 6 個 P1（多為 doc drift）+ 4 個 P3
- security-review 找 1 個 P0 (H1 prototype pollution) + 2 個 P2 (M1/M2) + 1 個 P3 (L1)
- Codex 找 6 個 P1（多為 correctness 真 bug，aiecsjs 觀察者三大隱患都是它抓到的）

只跑單一 reviewer 會嚴重漏網。**Codex 的價值在它的 context 完全獨立**，會看到主流程沒被注意的細節（例如 `removeComponent` 派送 observer 時 mask 還沒寫入、`destroyEntity` 不發送 query-targeted remove）。

**第二輪 re-review 同樣重要**：第一輪的修法**會引入新 bug**。本輪 Codex 在 round 2 抓到兩個我自己 round 1 修法引入的 P1：
- `aibridgejs` 的 `readyReject` 在舊 promise resolve 時無 identity guard 清掉新一輪的 reject handle
- `aiecsjs` Phase 2 destroy dispatch 讀 live `state.entityMask` 而非 pre-destroy snapshot

**結論：兩輪三路是基線，不是奢侈品。**

### 2. 修法常見陷阱（具體 pattern + 教訓）

每條都來自本輪的真實 bug。

**Prototype pollution via `Object.assign`**
- 反例：`Object.assign(inst, JSON.parse(untrustedBytes))` — `__proto__` 走 [[Set]] 觸發 setter，per-instance prototype 被替換
- 正例：`for (const k of Object.keys(src)) { if (k === '__proto__' || k === 'constructor' || k === 'prototype') continue; target[k] = src[k] }`
- 不要信賴 ESM/strict-mode 自動 fix — `JSON.parse('{"__proto__":...}')` 在 own property 上、無法用 strict mode 擋下
- 教訓：所有「user-input 物件合併到 instance」的路徑都要 audit。spread `{...a, ...b}` 走 `[[CreateDataProperty]]` 安全；`Object.assign` 走 `[[Set]]` 不安全

**Iterator 與 mutator 共用同一 collection**
- 反例：`for (const obs of state.observers) obs.handler(...)`，handler 內呼叫自己的 unsubscribe 會 `splice` 掉自己 — for-of iterator 跳過下一個兄弟
- 正例：`const snapshot = Array.from(state.observers); for (const obs of snapshot) { if (!state.observers.includes(obs)) continue; obs.handler(...) }`
- 已知副作用：每次 dispatch 變 O(N²)（snapshot 內每個元素查 includes）。下一個 cycle 改 `Set<>` 鏡像或 active flag 修
- 教訓：任何能在 callback 內被 mutate 的 collection 都要 snapshot 走訪

**「先 fire 後 mutate」vs「先 mutate 後 fire」**
- 反例：`aiecsjs.removeComponent` 先 `fireRemoveObservers`、再 `clearBit + migrateEntity`。query observer 讀 `state.entityMask` 時 bit 還在，query 仍 match，remove observer 不觸發
- 正例：mutate（mask 寫入）先，fire 後。aiecsjs 的 `addComponent` 一直是這個順序，`removeComponent` 偏偏相反
- 教訓：同類型的入口（add/remove）順序必須對稱。對稱性破壞是真 bug 的溫床

**兩階段派送的 state snapshot**
- 反例：destroyEntity Phase 1 firing 與 Phase 2 firing 都讀 live `state.entityMask`，Phase 1 handler reentrant mutate 會讓 Phase 2 miss
- 正例：Phase 1 開始前 `const preMask = new Uint32Array(...); for (...) preMask[i] = state.entityMask[base + i]`，Phase 2 用 preMask
- 教訓：multi-phase 派送，每個 phase 從 entry state 重新計算；不要依賴 phase 間的 live state

**Identity-guarded reject / cleanup**
- 反例：`readyReject = wrappedReject; ... wrappedReject = (reason) => { readyReject = null; reject(reason) }` — 舊 promise 的 resolve handler 不管當前 `readyReject` 是不是還指向自己就清 null
- 正例：`if (readyReject === wrappedReject) readyReject = null` — 只在 slot 還是自己時清
- 教訓：cleanup 函式更新共享 module-level 變數時必查 identity。多輪 lifecycle（reset → 新 promise）很容易踩

**Reentrant callback 與 mutable outer state**
- 反例：`send()` commit `snapshot = result.snapshot`，然後 callbacks 讀 `snapshot`。effect handler 同步 reenter `send()` 會 reassign `snapshot`，外層 callback 讀到 inner state
- 正例：`const committed = result.snapshot` 在 callbacks 前 capture；所有 callback context 用 `committed`
- 教訓：任何「commit 後跑使用者 callback」的點都先 capture local。即便 callback 看似不會 reenter，下個版本的某個 listener 會

**`instanceof Promise` vs `isThenable`**
- 反例：`if (result instanceof Promise) throw ...` — cross-realm Promise（iframe / worker / vm）`instanceof` 拿不到，user-defined PromiseLike 也漏
- 正例：`function isThenable(x) { return x !== null && (typeof x === 'object' || typeof x === 'function') && typeof x.then === 'function' }`
- 教訓：library 級 detection 永遠用 duck typing（callable `.then`），不用 `instanceof`

**`Function.prototype.name` 空字串陷阱**
- 反例：`const name = fn.name ?? '<inline>'` — 匿名 arrow function 的 `.name === ''`，`??` 不會 fallback empty string
- 正例：`const name = fn.name || '<inline>'`
- 教訓：`??` 只處理 nullish；`||` 處理 falsy（含 空字串、0、false）。對「字串可能是空」的情境用 `||`

### 3. Build / release tooling 陷阱

**`verify:llms = git diff --exit-code` 是 commit-後 gate**
- 反例（aifsmjs 原版）：`node scripts/build-llms-full.mjs && git diff --exit-code -- llms-full.txt` — working tree 有任何 uncommitted 改動就紅燈
- 正例：腳本內加 `--check` flag，build in-memory、與 disk 內容 byte-by-byte 比對、exit 1 if diff。pre-commit / commit-後 / CI 都正確
- 教訓：所有 lint-like gate（drift detection / format check）都該支援 pre-commit 模式

**Size budget 不是 zero-cost**
- 新功能（AsyncGuardError + thenable detection）讓 aifsmjs core 從 3481 B → 3618 B、replay 從 1583 B → 1685 B，雙雙撞 size-limit
- 加新 export / 加 throw / 加 try-catch 都會有 gzip 成本（即便 minify 後）
- 教訓：feature add 之前先預估 size delta，PR 內 `pnpm check:size` 必跑；budget 跟 feature 一起 raise，不要事後追

**Lint 設定不對稱**
- aifsmjs / aibridgejs 用 [Biome](https://biomejs.dev/)，aiecsjs 沒有
- 結果是 aiecsjs `src/internal/*` 累積 81 個 `noExplicitAny` warn 沒人發現
- 教訓：同生態系套件的 lint / format / verify gate 必須對稱；建新套件時 copy 既有的 [biome.json](aifsmjs/biome.json)、[verify-exports.mjs](aifsmjs/scripts/verify-exports.mjs)、[check-size.mjs](aifsmjs/scripts/check-size.mjs)、[build-llms-full.mjs](aifsmjs/scripts/build-llms-full.mjs)、[CONTRIBUTING.md](aifsmjs/CONTRIBUTING.md)

### 4. GitHub 發布工作流陷阱

**Lightweight tag 不被 `--follow-tags` 帶**
- `git tag v0.2.0`（未加 `-a`）建出來的是 lightweight tag
- `git push --follow-tags` 只推 **annotated** tag — lightweight tag 留在本地，commit 已推遠端但 tag 沒
- 解法：要嘛 `git tag -a v0.2.0 -m "..."` 建 annotated，要嘛顯式 `git push origin v0.2.0`
- 教訓：release tag 用 annotated；或乾脆永遠 `git push origin <tag>` 顯式

**Publish workflow 觸發條件三套件不對稱**
- `aifsmjs.publish.yml`：`on: push: tags: ["v*"]` → 推 tag 即啟動
- `aiecsjs.publish.yml`：同上 → 推 tag 即啟動
- `aibridgejs.publish.yml`：`on: release: published` → 需用 `gh release create v0.2.0 --notes ...` 才啟動
- 本輪 aibridgejs 第一次 push tag 後 Publish 沒跑，需要額外建 release 才補上
- 教訓：三套件 workflow 配置要對齊；下個 cycle 把 aibridgejs 改成 tag-triggered

**npm OIDC trusted publisher 是好東西**
- 三套件都採用：CI 用 `id-token: write` permission + npm CLI 11+ 的 `--provenance` flag，免長期 `NPM_TOKEN` secret
- 副效應：發布物自動附 sigstore provenance attestation，下游可驗證來源
- 設定一次性：npmjs.com Package Settings → Trusted publisher → 填 `<owner>/<repo>` + workflow filename

**Dependabot 報告在 push 後才顯現**
- `aifsmjs` / `aiecsjs` push v0.2.0 後 git 提示 2 個 moderate vulns（dev-only graph）
- gh CLI 不直接列 security alerts；要在 dashboard 確認
- 教訓：每次 release 後檢查一下 Dependabot tab

### 5. 文件 drift 的隱患

AI agent 讀文件的優先順序 ≠ 人類。

- 人類讀 `README.md`
- AI agent 優先 ground 在 `api.json`、`llms-full.txt`、`STABILITY.md` — **machine-readable surface drift 比 README drift 更危險**
- 本輪發現：`api.json` 的 observer signature 沒同步 `opts?: ObserverOptions`，AI 工具會生出舊版 signature 的程式碼

**雙語維護成本**
- aibridgejs 是唯一有 `llms_ZHTW.txt` 與 `llms-full_ZHTW.txt` 的套件（4.4 KB + 22 KB）
- LLM 預設 ground 在英文，中文 llms 對 AI 價值低
- 27 KB 塞在 npm tarball 裡 + 每次改 README 要雙語同步 = 純成本
- 砍掉是對的。`README_ZHTW.md` / `CHANGELOG_ZHTW.md` 保留（人類讀）

**STABILITY tag 必須跟 roadmap 一起更新**
- 本輪 `getEntityGeneration` / `packEntity` 從 stable 改 experimental — 因為實際就是 returns 0 / identity
- 但 README 的 roadmap 段、STABILITY 的 Roadmap 表都還寫「0.2.x: EntityRef」— 而 0.2.0 已發、EntityRef 沒做
- 教訓：roadmap 段是「未來會做什麼」，**已釋出的版本要立刻挪到「已做什麼」**；殘留 N 個未實作的 0.x 承諾會嚴重誤導下游

### 6. AI 評鑑材料的可信度

使用者開頭提供了一份「另一個 AI 給的評鑑 prompt」當參考。實際對照後：
- **約 60% 仍成立** — `aibridgejs.call()` 缺 generic、`aiecsjs` observer 缺 AbortSignal、命名不對稱、aifsmjs guard 無 runtime async 偵測
- **約 30% 過時** — `aiecsjs` Entity ID 文件矛盾在 0.1.1 已修；`aibridgejs` targetOrigin `'*'` 同步拋錯在 0.1.x 已做
- **約 10% 應商榷** — 例如建議加 `call()` 的 `validator: (x) => x is T` 參數，但實際 generic + boundary 驗證（Zod/Valibot）是更乾淨的解

**結論**：AI 評鑑文字當「checklist 起點」用，**每條都要對照當前原始程式碼確認**，不要全盤接受。直接讀 src 永遠比信舊 AI 報告快。

### 7. 流程數據

|  | 第一輪 | 第二輪 |
|---|---|---|
| code-review 找到 | P0=0, P1=5, P2=7, P3=4 | 1 殘留 + 4 P2 新發現 |
| security-review 找到 | H1=1, M1+M2=2, L1=1 | M2/L1 補洞建議 |
| Codex 找到 | P0=0, P1=6, P2=5, P3=3 | 2 個我自己引入的 P1 regression |
| 三輪間發現 | — | **修法引入 regression = 16%**（5 個 round-1 修中 1 個有問題；6 個觀察者類修中 1 個 phase 順序問題） |

**核心數字**：第一輪修了 23 項；第二輪 re-review 找到 5 個新議題（2 個 P1 + 3 個 P2 / P3）。**Re-review 命中率 22%** — 第一輪做完別就 ship，再走一輪很值得。

最終測試數：394 → 406（+12 全為 regression test）。所有 regression test 都對應一個具體 bug，不是 vanity test。

### 8. 給下個 cycle 的可行清單

按優先序：

1. 把 `aibridgejs.publish.yml` 改成 `push: tags: ["v*"]` 觸發，跟另兩套件對齊
2. 修 `aiecsjs.observers.ts` 的 O(N²) dispatch（snapshot + `includes`），改 active-flag 或 `Set<>` 鏡像
3. 補 `aiecsjs` `getEntityGeneration` / `packEntity` 的真正打包（ABA-safe `EntityRef`），把 experimental → stable
4. 清掉 `aiecsjs/src/internal/*` 的 81 個 `noExplicitAny` warn，或加 `// biome-ignore` 寫明理由
5. 修 `aifsmjs` 的 definition.ts ↔ runtime.ts 雙向 import（把 `initialSnapshot` 拆到第三模組）
6. 評估 aiecsjs 的 internal SAB transport（`aiecsjs/worker`）是否該強制 `trust` 參數
7. 建一個共用 `eslint-config-aijs` 或統一的 biome preset，三套件 lint 規則完全同步

### 9. 給未來 AI 代理人的提醒

如果你是接手這個 monorepo 的 AI（或 future 我自己讀這份）：

- **三路並行 review 是標準流程**，不是錦上添花。第一輪未必抓得到所有 bug，第二輪會
- **Codex 第二意見很關鍵** — Claude 主 context 看不到的問題它常常能看到，反之亦然
- **不要全信任 prior AI 評鑑** — 直接讀 src 比較快
- **doc drift 是首要 finding 類型** — `api.json`、`STABILITY.md`、`llms-full.txt`、`README` 之間必須對齊
- **新套件 scaffold 從 aifsmjs 拷貝** — 它目前是 most-conformant member
- **size budget 跟 feature 同步調整** — 別事後追
- **三個套件互不 import** — 整合走 documentation，不走 dependency
- **沒事不動 src/internal/** — 內部結構複雜（archetype migration、bitmask、SAB transport），改一行可能踩到三個 invariants

—— end of v0.2.0 cycle ——

---

## v0.3.0 cycle — 預備區（2026-05-28 起）

v0.2.x 三套件（aibridgejs / aifsmjs / aiecsjs）穩定後，使用者端要做 Svelte 5 runes + PixiJS 小遊戲，也可能單純做 SvelteKit / Nuxt 純網頁。本段記錄 v0.3 cycle 入場前對四個候選套件的評估與決議依據，供下一輪實作期重看。

### 1. 候選套件總覽

| 套件 | 決議 | 狀態 |
|---|---|---|
| [aipooljs](aipooljs/) | 自建 | 本輪建立 scaffold（README + ZHTW + LICENSE + CHANGELOG + src/index.ts throw stub） |
| [aiquadtreejs](aiquadtreejs/) | 自建 | 本輪建立 scaffold（同上） |
| aieventjs | 自建（**不** fork mitt） | v0.3 cycle 後續實作 |
| aiaudiojs | 自建（peerDependency Howler.js） | v0.3 cycle 後續實作 |

四個都決議自建，但理由與包裝策略各異 — aieventjs / aiaudiojs 各對標一個成熟外部套件，評估結論寫在下面兩節。

### 2. aieventjs vs mitt — 評估摘要

對標 `mitt@3.0.1`（developit/mitt），評估結論：**自寫 aieventjs，不 fork mitt**。

- **License** [確定]：MIT 純淨（`Copyright (c) 2021 Jason Miller`），fork+rename 合法，唯一義務是保留 copyright + permission notice。
- **Size** [確定]：mitt@3.0.1 實測 gzip **282 bytes**（Bundlephobia），minified 488 bytes。README 仍宣稱「<200 bytes」是過時宣傳。
- **Maintenance** [確定]：最新 release **3.0.1 @ 2023-07-04**，主分支同日 commit，停滯近 3 年。Open issues 26、stars 11,870。社群想要的 PR — unsubscribe return、AbortSignal、`sideEffects: false`、nodenext 相容 — 全部未 merge。維護者沉默。
- **缺口 vs ai*js convention** [確定]：mitt 缺 `unsubscribe`-returning `on()`、`once()`、`AbortSignal`、`dispose()` + `EventBusDisposedError`、`sideEffects: false`、`noUncheckedIndexedAccess` 等級 strict TS、tsup pipeline。`on()` 須手動 `off(type, handler)` 解綁，與 ai*js 家族 `() => void` unsubscribe pattern 不一致。
- **Fork 經濟性** [推測]：純邏輯 ~35 行，fork 等同重寫；upstream 已死無法回饋 PR；繼承 copyright notice 反而是多餘負擔。
- **結論**：自寫。估 **100–140 行 TS**，gzip **450–550 bytes**（含 dispose / signal / once / JSDoc）。**保留 wildcard `*` handler** 作為 mitt 既有用戶遷移誘因（mitt 招牌特色，跳過可省 ~80 bytes 但失去心智模型相容性 — 不建議跳）。

### 3. aiaudiojs vs Howler.js — 評估摘要

對標 `howler@2.2.4`（goldfire/howler.js），評估結論：**自建薄殼，Howler.js 設為 peerDependency**。

- **License** [確定]：MIT 純淨（`Copyright (c) 2013-2020 James Simpson and GoldFire Studios, Inc.`），允許 (a) 直接用、(b) fork & rebrand、(c) peerDependency 包裝重新導出 — 三者皆無摩擦。
- **Size** [確定]：howler@2.2.4 預設入口 `dist/howler.js` 36.5 KB min / **9.7 KB gzip**；`howler.core.min.js` 26.3 KB（純 Web Audio，無 HTML5 fallback）；`howler.spatial.min.js` 9.1 KB（spatial plugin）。0 runtime deps。
- **Maintenance** [確定]：最新 release **2.2.4 @ 2023-09-19**；master 最新 commit 2025-11-23 但僅 README 編輯，功能性 commit 自 2023-09 停滯。Open issues 363、stars 25,300。半休眠 — API 成熟穩定，但 iOS regression 不再積極跟進。
- **iOS Safari 邊角狀況** [部分確定]：unlock happy-path 穩定；iOS 17.4+ / 18 邊角情境未解（#1744 VoiceOver / Audio Ducking、#1711 HTML5 live stream 永久 buffering、#1668 高頻播放 HTML5 pool 耗盡、#1702 背景化後 interrupted 無法恢復）。**這些是 WebKit 病，自寫一樣中** — 重寫救不了。
- **包裝可行性三選項** [推測]：
  - **A. App 直接 deps Howler.js，無 aiaudiojs**：effort 0；gzip 9.7 KB；ai*js 一致性破洞、使用者重複寫 boilerplate。
  - **B. peerDependency 薄殼（選此）**：effort 1–2 週；殼層自身 ~1.5–2 KB gzip（總計 ~11.5 KB）；維護低（年 1–2 次跟版 + iOS hook 補丁）；API 對齊 `dispose()` / AbortSignal / `on(..., { signal })` / 一級 `crossfade()`。
  - **C. 從零 Web Audio**：effort 4–8 週（含 iOS 真機回歸）；gzip 6–10 KB；維護高 — 重踩 Howler 13 年 iOS unlock fixes、收益有限。
- **結論**：**B**。公開子集 `createAudio` / `load` / `play` / `pause` / `stop` / `fade` / `crossfade` / `setVolume` / `setRate` / `setSpatial` / `on,once,off` 帶 signal / `dispose` / `disposeAll`，保留 `getNativeHowl()` escape hatch 給需要 Howler 進階 API 的呼叫方。

### 4. aipooljs / aiquadtreejs 背景

兩者皆無近似既有套件可直接包裝：

- **aipooljs**：物件池無公認生態系套件；hand-rolled 版本多在 game-engine 內部，沒有獨立 npm package 既有可用 + 維護中 + 與 ai*js convention（`dispose()` / `AbortSignal`-friendly / strict TS）一致的。
- **aiquadtreejs**：`@timohausmann/quadtree-ts` 是接近的選項（~2 KB、MIT、retrieve Set 去重已 O(n)），但缺與 aiecsjs entity ID + AABB 的 zero-copy 整合介面；自建可在 0.3+ 直接消費 `Uint32Array` view，省掉每 frame 物件 alloc。

**自建為唯一可行路徑**。本輪先落地 scaffold + API surface，實作留給下一輪。

### 5. 給下一輪 cycle 的進度旗標

- [x] aipooljs scaffold 落地（README + ZHTW + LICENSE + CHANGELOG + src/index.ts throw stub）
- [x] aiquadtreejs scaffold 落地（同上）
- [x] 根目錄 README.md 新增 v0.3.0 candidates 章節
- [x] aipooljs 完整 scaffold（package.json / tsconfig / tsup / vitest / biome / scripts / test 占位 / CI workflow，publish workflow disabled，0.0.1 bumped）
- [x] aiquadtreejs 完整 scaffold（同上）
- [x] aieventjs 完整 scaffold 建立（含 README / ZHTW / LICENSE / CHANGELOG + 14 配置檔，src 為 mitt-like throw stub）
- [x] aiaudiojs 完整 scaffold 建立（同上，含 Howler.js `^2.2.4` peerDep 宣告）
- [x] `/codex:review` 一次性覆核四套件，抓 7 P0 + 3 P1 全修畢（method-vs-property、`v0.1.0`→`v0.0.1` 字串、`@example` 自包含、`EmitterOptions` 出現在 API sketch、ecosystem footer 補齊）
- [x] 四套件 `.gitignore` 補齊 + `git init -b main` + commit + annotated `v0.0.1` tag + push 至 GitHub remote
- [x] 四套件 GitHub Actions `ci.yml` 全綠（24-30 秒一次），`publish.yml` 因為只接 `workflow_dispatch` 沒被誤觸
- [x] aieventjs 0.0.1 → 0.1.0 實作 + tests + publish — [v0.1.0 on npm](https://www.npmjs.com/package/aieventjs/v/0.1.0)
- [x] aipooljs 0.0.1 → 0.1.0 同上 — [v0.1.0 on npm](https://www.npmjs.com/package/aipooljs/v/0.1.0)
- [x] aiquadtreejs 0.0.1 → 0.1.0 — git tag landed; first npm release shipped as v0.1.1 (see below)
- [x] aiaudiojs 0.0.1 → 0.1.0 — same as aiquadtreejs (0.1.0 git-only milestone, first npm release is v0.1.1)
- [x] 每個套件跑 Codex review；aiaudiojs 加跑 security review (passed clean)
- [x] **v0.1.1 patch — OIDC trusted publisher + provenance attestation pipeline 跑通**：四套件 [`v0.1.1` on npm](https://www.npmjs.com/package/aieventjs/v/0.1.1) 都掛 [SLSA Level 3 provenance](https://slsa.dev/provenance/v1)（GitHub Actions tag-push trigger → OIDC token → `npm publish --provenance` 一步完成，零 OTP 互動）
- [ ] 整合 demo：PixiJS 小遊戲整合 aiecsjs + aifsmjs + aipooljs + aiquadtreejs + aiaudiojs，置於 `/Volumes/MiniBackup/sys/examples/pixijs-shmup/` 或專屬 repo

### 6. 給下一輪 AI 代理人的提醒

- **mitt fork 已評估過、結論是自寫** — 不要再花 token 重評估，直接寫
- **Howler.js 是 MIT、可包薄殼** — 不要再走「從零 Web Audio」這條路，iOS unlock 邊角是 WebKit 病不是 Howler 病
- **aipooljs / aiquadtreejs 的 API surface 已凍結** — 兩個 README 與 `src/index.ts` 是契約；實作期不要改 export 形狀，否則 README 與實作會 drift
- **新套件 scaffold 仍從 aifsmjs 拷貝** — 是 most-conformant member；本輪兩個新套件已對齊五-badge shields + tagline + ecosystem footer + 精簡 6 章節 README + ZHTW mirror

### 7. Subagent + Codex 二鐘流程的實證紀錄（v0.3 cycle）

四套件 0.0.1 → 0.1.0 + 0.1.1 publish 全程的工作流數據，作為「降 token 負擔」策略的實證。

**Pipeline 確定為**：
- **Phase A — Spec**: **Opus 主 context** 自寫 spec（200-310 行 markdown）
  - 第一次嘗試用 Codex 跑 aieventjs spec — 卡住 14+ 分鐘無 Assistant message captured 事件後 cancel。
  - 教訓寫進 memory：[[codex-prompt-weight]] — Codex 跑長 output prompt 容易 hang；只給輕量 review prompt。
- **Phase B — Impl**: **`Agent(subagent_type="general-purpose", model="sonnet")`** 寫 src + tests + CHANGELOG + config bumps
  - 自包含 prompt（讀 spec from /tmp）、要求 summary 回主 context
  - 平均 5-15 分鐘出完整實作
- **Phase C — Review**: **Codex** 短 prompt（≤150 行 finding 上限）
  - 平均 3 分鐘出完整 P0/P1/P2/P3 finding list
  - 每套件都抓到至少 1 個真 P1（doc drift / correctness / security gap）
- **Phase D — Publish**: Opus 主 context 跑 `pnpm prepublishOnly` → commit + tag + push

**四套件 Phase 數據**：

| 套件 | Phase A | Phase B 規模 | Phase C finding | Final gzip / budget |
|---|---|---|---|---|
| aieventjs | Opus spec 310 行 | Sonnet 345 行 + 52 tests | 2 P1（wildcard snapshot order / README allocation claim） | 754 B / 800 B |
| aipooljs | Opus spec 270 行 | Sonnet 192 行 + 31 tests | 2 P1（drain O() claim / size budget drift） | 557 B / 700 B |
| aiquadtreejs | Opus spec 320 行 | Sonnet 280 行 + 33 tests | 1 P1（zero-extent at midpoint dropped）+ 2 P3 doc | 967 B / 2000 B |
| aiaudiojs | Opus spec 340 行 | Sonnet 460 行 + 33 tests (Howler mock + happy-dom) | 4 P1 + 1 P2（pause/stop ck guard, README 多處 drift, abort listener leak）+ security clean | 1423 B / 2000 B |

**所有 finding 都修畢、所有 gate 全綠**。Phase C 抓出來的 7 個 P1 都是真 bug 或真 drift — Codex 的「獨立第二意見」價值在這次 cycle 完全證實。

**publish workflow 跑通**：
- 0.1.0 全部本機 `pnpm publish --no-git-checks --access public --otp=<6>` 跑（OTP 互動）
- 0.1.1 patch 改 `publish.yml` 加 `on: push: tags: ["v*"]` + `--provenance` → tag push 觸發 GitHub Actions OIDC trusted publisher → 自動 publish + SLSA Level 3 provenance attestation，零 OTP 互動。四套件都成功。

**給下一輪的提醒**：
- **長 spec 不要丟給 Codex** — 之前踩過 14 分鐘 hang。Opus 自寫快過 codex
- **Codex review 限定 ≤150 行 output** — prompt 內明示「Keep your report under 150 lines」很有效
- **Sonnet Agent 適合 src + tests 規模 200-500 行的任務** — 比直接讓 Opus 寫快很多 + token 省 50-70%；但 Agent 完成後 Opus 必須自跑 prepublishOnly 驗證可重現（曾在 aieventjs 抓到 llms-full.txt drift）
- **OIDC trusted publisher 設好後，不再需要本機 OTP 跑 npm publish** — tag push 即發 + provenance attestation

—— end of v0.3.0 cycle ——
