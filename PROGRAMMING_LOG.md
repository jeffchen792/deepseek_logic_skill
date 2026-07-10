# Programming Experience Log

> 兩個專案（Portfolio + Calendar）踩過的坑，和學到的東西。

---

## 1. 前端效能診斷

### fontsReady gate 殺了 3D
`document.fonts.ready.then()` 在自託管 @font-face 字體可能永遠不觸發，導致 `{fontsReady && <Component />}` 永不 render。
**教訓**：條件渲染的 gate 要加 timeout fallback，或直接不 gate 不依賴外部資源的元件。

### R3F Canvas 吃滾輪事件
`<Canvas>` 內部對 canvas DOM 設了 `pointer-events: auto`，外層 div 的 `pointer-events: none` 無法穿透。
**教訓**：第三方元件內部可能覆寫你設的 CSS。`onCreated` 裡直接改 `state.gl.domElement.style.pointerEvents`。

### React 路由吃掉 UI state
`createPair()` 內部呼叫 `setUser()` → App.jsx 看到 user 非空 → redirect `/dashboard` → 邀請碼顯示步驟被跳過。
**教訓**：觸發路由跳轉的 state 變更，要在 UI 完成展示後才執行，不要放在 data-fetching function 裡面。

---

## 2. 後端安全

### RLS 無限遞迴
`CREATE POLICY ... USING (pair_id IN (SELECT pair_id FROM users WHERE id = auth.uid()))` 在 `users` 表上查 `users` → Postgres 對每行重新觸發 policy → 遞迴。
**正確修法**：`SECURITY DEFINER` function 繞過 RLS 遞迴檢查，因為它以自己的權限執行（不是 caller 的 RLS 上下文）。

### anon key ≠ 公開資料的許可證
Vite 的 `VITE_` 前綴 env var 會被打包進 JS bundle，任何人 F12 都能看到。anon key + `USING(true)` = 全世界可讀寫你的資料庫。
**教訓**：anon key 只能用來「識別你是誰」，不能用來「證明你有權限」。真正的授權要靠 RLS + auth.uid()。

---

## 3. CSS / 視覺

### z-index 與 stacking context
`position: fixed` 或 `opacity < 1` 的元素會建立新的 stacking context。放在裡面的 `z-index: 60` 對外面無效。
**教訓**：燈箱 / modal 用 `createPortal` 掛到 `<body>`，不要放在有 `-z-10` 的容器內。

### 負 z-index 的背景層
`-z-10` 的背景必須放在沒有不透明背景色的父層裡，否則父層的 background 會蓋住它。
**教訓**：背景層的父元素不要設 `bg-*` class。

---

## 4. 部署

### Vercel Deployment Protection
團隊帳號預設開 Vercel Authentication，所有 deploy preview 和 custom alias 都要登入才能看。
**教訓**：Deployment Protection → Vercel Authentication → Disabled。

### Vite 環境變數前綴
`VITE_` 前綴才會被打包進前端 bundle。`vercel env add` CLI 比在 dashboard 手動加快。
**教訓**：每個框架都有前綴規則（Next.js = `NEXT_PUBLIC_`，Vite = `VITE_`）。

---

## 5. 3D / WebGL

### 星場太暗 = 看起來像沒渲染
Hero 進度 p=0 時相機在 (0,0.5,10)，所有 3D 地標都在 -55 之後，只有 size=0.2 的 additive blending 星點 — 肉眼幾乎看不到。
**教訓**：首屏要有近景物件或加大星點。3D 場景「沒報錯」≠「使用者看得到」。

### Suspense 不捕捉 throw 的錯誤
`<Suspense fallback={null}>` 只處理 lazy loading，不處理 runtime error。EffectComposer 掛掉會是空白畫面 + console 紅字。
**教訓**：加 ErrorBoundary 包住 3D 場景。

---

## 6. 思考方法

### 讓報錯消失 ≠ 修好了
三次重複模式：
- 滾動卡死 → `events={() => undefined}` 關掉整個 render loop
- RLS 遞迴 → `USING(true)` 關掉整組安全規則
- 照片不顯示 → 調低 opacity 讓它融進背景

全部都是「關掉警報器」而不是「解決火災」。
**教訓**：動手改 code 之前，先走完五步診斷框架（見 `root-cause-first` skill）。

### 動態 import 的陷阱
如果 module A 已經被 static import 了，對 A 做 dynamic import 不會產生新的 chunk，只是多一次網路請求。
**教訓**：`import()` 只在 module 還沒被 static import 時有用。Vite 會印 `INEFFECTIVE_DYNAMIC_IMPORT` 警告。
