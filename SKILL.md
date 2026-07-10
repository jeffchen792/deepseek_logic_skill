---
name: root-cause-first
description: "Systematic debugging: observe → hypothesize → experiment → verify. Forces evidence-based fixes with mandatory output format. Never fix by disabling."
version: 3.0.0
tags: [debugging, methodology, thinking, logic, verification]
triggers:
  - debug
  - fix
  - broken
  - not working
  - error
  - why
  - 修
  - 壞了
  - 沒出來
  - 卡住
---

# Root Cause First — 先觀察，再假設，驗證了才動手

## 第零條：證據分級（整份 skill 的地基）

你腦中的每個判斷只可能是三種東西之一：

| 等級 | 定義 | 例子 |
|------|------|------|
| 事實 | 剛剛實際觀察到的輸出 | 「screenshot 顯示全黑」「`window.__frames` 是 undefined」 |
| 已驗證推論 | 由實驗結果直接支持的結論 | 「移除 X 後 loop 復活 → X 是死因」 |
| 猜測 | 聽起來合理但沒驗證過的解釋 | 「應該是 pointer-events 的問題」 |

**鐵則：修法只能建立在前兩級之上。** 每當你寫下「應該是」「可能是」「大概因為」，下一句必須是「所以我用 ___ 來驗證」。禁止基於猜測直接改 code。

## 強制輸出：除錯日誌

診斷任何 bug 時，**必須**邊做邊維護這張表（這不是建議，是輸出格式要求）：

```
| # | 觀察到的事實 | 假設 | 驗證實驗 | 結果 | 結論 |
|---|-------------|------|---------|------|------|
| 1 | 畫面全黑，console 無錯 | 可能沒 mount | 加 useEffect log | 有印出 MOUNTED | 排除：有 mount |
| 2 | 有 mount 但無畫面 | render loop 死了 | useFrame 加計數器 | 計數器恆為 undefined | 確認：loop 沒跑 |
```

規則：
- 「驗證實驗」欄必須是**實際執行過**的操作，不能填「檢查了程式碼」「重新閱讀邏輯」
- 提出修法前，表中至少要有一行「結論」欄寫著「確認」
- 一次實驗只改一個變因；實驗用的暫時碼標 `// TEMP DEBUG`，收工前全部移除

## 五步思考框架

```
1. 這東西本來是幹嘛的？
   └─ 理解機制的設計意圖，不是只看它現在的行為

2. 什麼觸發了它失敗？
   └─ 重現 → 縮小範圍 → 找到最小觸發條件

3. 為什麼這個觸發會讓它失敗？
   └─ 核心。不是「報了什麼錯」，是「為什麼這個輸入造成這個結果」

4. 怎麼讓它在這個情況下正常工作？
   └─ 修正機制本身，不是繞過它、關掉它、或用另一條路取代它

5. 修完之後，有沒有弄壞別的東西？
   └─ 見「回歸測試鐵則」
```

## 靜默失效清單 —「沒有報錯」≠「沒有錯」

當症狀是「壞了但沒有任何錯誤訊息」，先對照這張表，逐項排除：

| 靜默失效模式 | 怎麼偵測 |
|-------------|---------|
| uncaught error 沒進 console 監聽 | `window.addEventListener('error')` + `'unhandledrejection'` 自己抓 |
| React/Vue 整棵樹無聲 unmount | 檢查 `#root.children.length`；黑屏＋高度塌陷是典型症狀 |
| 動畫/render loop 無聲死亡 | 在 loop 裡放 `window.__frames` 計數器，兩秒後讀 |
| 元素有渲染但被蓋住/在畫面外 | 隱藏上層 DOM 直接看；檢查 z-index stacking context |
| 事件被吞（pointer-events、stopPropagation） | 注意：`elementsFromPoint` 會跳過 `pointer-events:none` 的元素 |
| 條件分支走進降級路徑 | 每條降級路徑都要有 `console.warn` 留痕，沒有就先補 |
| Promise 永不 resolve / listener 永不觸發 | 加 timeout log；別讓 `Suspense fallback={null}` 把「掛了」變空白 |
| 環境差異（OS 設定、瀏覽器、裝置） | 檢查 `prefers-reduced-motion`、`hardwareConcurrency`、視窗尺寸 |

## 修法分類器 — 動手前的最後一關

每個修法必須先自我歸類，並照規則走：

```
這個修法是哪一種？
├─ 修正：改正機制內部的錯誤邏輯 → 直接做
├─ 繞過：加旁路讓症狀不出現，機制照舊 → 必須先寫出：
│    (a) 原機制為什麼失敗（要有除錯日誌支持）
│    (b) 繞過後誰來承擔原機制的職責
└─ 移除/停用：把機制關掉 → 最高警戒。必須先回答：
     (a) 這個機制當初是防什麼的？（Chesterton's Fence：
         不知道柵欄為什麼在那裡之前，不准拆）
     (b) 拆掉後那個風險由什麼取代？
     (c) 明確寫出「我正在用功能換取症狀消失」讓使用者確認
```

紅旗詞彙（用這些讓保護機制失效時立刻停下）：
`USING (true)`、`events={() => undefined}`、`pointer-events: none`（為了讓報錯消失而加時）、
空的 `catch {}`、`// eslint-disable`、`!important`、`?.` 蓋掉 null 錯誤、
`setTimeout` 修 race condition、調 `opacity`/`z-index` 修「看不到」、砍掉整個功能換「不報錯」。

## 回歸測試鐵則

1. 修完 bug A，**重跑 bug A 的原始復現步驟**，親眼看到它不再發生
2. 如果這次修改「恢復」了任何之前被停用的東西，**重跑當初導致它被停用的那個 bug 的復現步驟**——舊 bug 復活是最常見的翻車
3. 同區域曾經修過的 bug，一併重測
4. 「驗證」的意思是觀察實際輸出（截圖、log、exit code、資料庫查詢結果），不是重讀程式碼覺得沒問題

## 三個反例（真實踩坑，全部同一個病：跳過第 3 步）

| 症狀 | 錯的做法 | 真正原因 | 正確做法 |
|------|----------|----------|----------|
| 捲到底滾動卡死 | `events={() => undefined}` | re-render 觸發 EffectComposer 崩潰、app 無聲 unmount | 讓 Canvas 子樹不再 re-render |
| RLS 報 500 遞迴 | `USING (true)` 全開 | policy 查自己所在的表造成遞迴 | SECURITY DEFINER function |
| 背景照片看不到 | 調低 opacity 硬上 | 負 z-index 子元素被父層不透明背景蓋住 | 修正疊層結構 |

共同模式：**把警報器關掉，當作火不存在。** 而且第一例還示範了連鎖災難：錯誤修法本身變成下一個 bug（render loop 全死），並且掩蓋了真兇——直到有人重跑原始復現步驟才被抓出來。

## 完工自查

- [ ] 除錯日誌表完整，修法對應到某行「確認」結論
- [ ] 所有 `// TEMP DEBUG` 碼已移除
- [ ] 本次 bug 的復現步驟重跑過，親眼確認
- [ ] 歷史 bug（尤其同區域）的復現步驟重跑過
- [ ] 修法分類不是「移除/停用」；若是，三個問題都已回答並經使用者確認
- [ ] 踩坑寫進專案的 TROUBLESHOOTING.md：症狀 → 誤診（如有）→ 真因 → 修法
