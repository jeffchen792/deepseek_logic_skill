# Programming Experience Log

> Portfolio + Calendar 兩個專案踩過的坑，按症狀分類，方便日後查閱。

---

## 症狀：畫面空白 / 元件不 render

### 條件 gate 殺了自己
```jsx
// ❌ 自託管 @font-face 可能永遠不 resolve
document.fonts.ready.then(() => setReady(true));
return ready && <Component />;

// ✅ 加 timeout fallback，或不 gate 不依賴外部資源的元件
setTimeout(() => setReady(true), 3000);
```
**偵測**：元件沒 render 但沒報錯 → 檢查所有 conditional render 的 gate。

### 空狀態的條件渲染死結
```jsx
// ❌ 只有已設定日期的人才看得到設定按鈕 → 沒設定的人永遠找不到入口
{user?.pairedAt && <EditButton />}

// ✅ 永遠顯示入口，沒設定時用不同文字
<EditButton>{user?.pairedAt ? `${days}天` : "設定紀念日"}</EditButton>
```
**偵測**：功能「存在但找不到」→ 問「第一次進來條件會是 true 還是 false」。

### R3F `useFrame` 不執行
`<Canvas events={() => undefined}>` 會讓整個 render loop 停擺。不是「關掉事件」而已。
**偵測**：3D 場景不更新 → 在 `useFrame` 裡加 `console.count()` 看有沒有在跑。

### Suspense 不接 runtime error
`<Suspense fallback={null}>` 只處理 lazy load，不處理 throw。
**偵測**：畫面空白 + console 紅字 → 加 `<ErrorBoundary>`。

---

## 症狀：改 A 結果 B 壞了

### 加新功能時拆掉舊功能的條件分支
加「星空」tab 時，把月曆/列表的條件渲染合併成一個區塊，導致兩個 view 永遠顯示相同內容。
**偵測**：點 tab 畫面沒變 → 檢查每個 tab 對應的 JSX 是否被合併或條件被移除。

---

## 症狀：兩個裝置看到不同資料

### localStorage 不是共享的
`pairedAt` 只存 localStorage → A 改了 B 看不到，兩人各說各話。
**解法**：多人共享的資料要放後端（Supabase），localStorage 只放個人偏好。
**偵測**：A 操作後 B 沒變化 → 檢查資料存在 localStorage 還是資料庫。

---

## 症狀：東西在但看不到

### 3D 物件在視錐外
相機在 (0, 0.5, 10)，所有地標在 z < -55。沒報錯，渲染正常，但什麼都看不到。
**偵測**：console 無紅字但畫面空白 → 把 camera far 調到 1000，看物件有沒有出現。

### z-index 被 stacking context 困住
`opacity < 1` 或 `transform` 的元素會建立新 stacking context，內部 z-index 對外面無效。
**偵測**：devtools → Layers 面板，看每個元素的 stacking context 邊界。

### 負 z-index 背景被父層 bg 蓋住
父層有 `bg-*` class 時，`-z-10` 的子層會被不透明背景完全遮住。
**偵測**：父層 element 的 computed background-color 不是 `transparent`。

---

## 症狀：API 500 / 權限錯誤

### RLS policy 自我參照造成無限遞迴
```sql
-- ❌ users 表的 policy 查 users 表 → 遞迴
CREATE POLICY "x" ON users USING (pair_id IN (SELECT pair_id FROM users WHERE ...));

-- ✅ SECURITY DEFINER function 以自身權限執行，繞過 RLS
CREATE FUNCTION my_pair_id() RETURNS UUID LANGUAGE sql SECURITY DEFINER STABLE
AS $$ SELECT pair_id FROM users WHERE id = auth.uid() LIMIT 1; $$;
CREATE POLICY "x" ON users USING (pair_id = my_pair_id());
```
**偵測**：Supabase 500 + `infinite recursion detected in policy`。

### anon key 洩漏
`VITE_*` env var 會被打包進瀏覽器 JS bundle，F12 → Sources 就能看到。
```
anon key + USING(true) RLS = 全世界可讀寫你的資料庫
anon key + auth.uid() RLS = 只有登入者能讀自己的資料
```
**偵測**：打開 production build 的 JS，搜 `supabase.co`，看 key 在不在裡面。

---

## 症狀：React state 時序錯誤

### setState 觸發路由跳轉，跳過 UI 步驟
```jsx
// ❌ createPair 裡直接 setUser → App 看到 user 非空 → redirect → 邀請碼顯示步驟被跳過
async function createPair(name) {
  const user = await api.createUser(name);
  setUser(user);  // 這行觸發 redirect
  return user.pairCode;  // 這行永遠不會被 UI 看到
}
```
**偵測**：某個 UI 步驟被跳過，但 data flow 看起來正確 → 檢查是否有 setState 觸發了 redirect。

---

## 症狀：重複的過度防禦

三個獨立的 bug，同一個 pattern：
```
isLowPower()    → 2 核心就跳過 3D
fontsReady gate → 字體沒載完就不 render
skip3D watchdog → 前 4 秒偵測到慢幀就永久降級
```
全部都是「保護機制太激進，把正常情境也擋掉了」。
**偵測**：看到 `if (condition) return fallback` 的 pattern → 問：這個 fallback 會不會在不該觸發的時候觸發？

---

## 快速查表

| 看到什麼 | 先做什麼檢查 |
|----------|-------------|
| 畫面空白 + 無報錯 | 檢查 conditional gate、視錐、z-index |
| 畫面空白 + 有報錯 | 看是不是 Suspense 吞了 ErrorBoundary 該接的 |
| 功能存在但找不到入口 | 檢查條件渲染的空狀態（第一次會是 true 嗎） |
| A 操作 B 看不到 | 資料存 localStorage 還是後端 |
| 點 tab 畫面沒變 | 檢查條件分支是否被合併/移除 |
| 3D 不動 | `useFrame` 裡加 counter 看有沒有執行 |
| 滾輪無效 | `getComputedStyle(canvas).pointerEvents` |
| Supabase 500 | 看 error.message 有沒有 "recursion" |
| UI 步驟被跳過 | 搜 `setUser` / `navigate` 是否在 async function 裡面 |
| 功能正常但不該這樣修 | 見 `root-cause-first` skill 的五步框架 |
