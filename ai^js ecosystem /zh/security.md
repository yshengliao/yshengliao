# ai*js — 安全與依賴基線

每個套件每次釋出都須滿足；由 code review 把關。

> English → [../en/security.md](../en/security.md) ｜ 回[索引](../README_ZH.md)

## 程式

- **不用 `eval`、`new Function` 或其他動態程式執行。** CSP 友善；家族會進 iframe、WebView、CSP 受限 host。
- **零 hardcoded secret** —— 連測試 fixture 都用環境變數。
- **邊界驗證。** 每個來自套件外的輸入（postMessage payload、`JSON.parse`、快照 adoption）都做結構驗證。binary 格式帶 magic + version header。
- **prototype-pollution 意識。** 不用 `Object.assign({}, untrustedJson)`；改用顯式欄位取出或 `structuredClone`，合併不可信物件時跳過 `__proto__` / `constructor` / `prototype` key。
- **transport 嚴格 origin / source 驗證。** bridge 的 iframe adapter 在 construction 時同步拒絕 `targetOrigin: '*'`，每個進站訊息都驗 `event.origin` 與 `event.source`。

## 依賴政策

- **預設零 runtime 依賴。** 唯一允許的例外是 *optional* `peerDependency`，讓對應 adapter 在未用時可被 tree-shake。
- **runtime 無 cross-package import。** 套件依慣例組合，不靠依賴邊。
- **devDependencies 追蹤 advisory。** transitive 安全修補即時跟進並記在 CHANGELOG。

## 供應鏈

- **npm OIDC trusted publisher** + provenance attestation —— 無長期 `NPM_TOKEN`。七套件皆此方式發布；每個版本帶 [SLSA Level 3 provenance](https://slsa.dev/provenance/v1)。
- **發布 gate。** `prepublishOnly` 跑完整品質鏈才放行；gate 失敗會擋下 tag 觸發的 publish workflow。
- **可重現建置。** 給定相同 lockfile，`tsup` 為 deterministic。
