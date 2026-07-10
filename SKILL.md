---
name: diagnosis-first
description: "Diagnose root cause before fixing — never make symptoms disappear by disabling mechanisms."
version: 1.0.0
tags: [debugging, methodology, safety, discipline]
triggers:
  - fix this error
  - it stopped working after
  - debug this
  - why is this broken
---

# Diagnosis First — 先診斷，再修復

## 核心原則

**任何修法如果是「關掉一個機制」而不是「修正它」，先解釋清楚為什麼原本的機制會失敗。**

## 三個實例

| 專案 | 症狀 | 錯的修法 | 為什麼錯 | 正確診斷 |
|------|------|----------|----------|----------|
| Portfolio 3D | 滾動卡死 | `events={() => undefined}` 關掉 R3F render loop | 沒問為什麼 canvas 會吃滾輪事件 | R3F canvas 的 pointer-events: auto 攔截了 wheel，應該設 pointer-events: none |
| Calendar 照片 | 照片不顯示 | 調低 opacity 融進黑底 | 沒問為什麼照片在畫面下方 | z-index 疊層順序：calendar grid 蓋在照片上面 |
| Calendar RLS | 500 遞迴錯誤 | `USING(true)` 關掉整組 RLS | 沒問為什麼 policy 會遞迴 | `users` 表的 policy 查了自己，需要 SECURITY DEFINER function 繞過 |

## 診斷流程

```
收到報錯/異常
    │
    ├─ 1. 重現 → 確認不是偶發
    ├─ 2. 定位 → 哪個機制失敗了？（不是哪行報錯）
    ├─ 3. 理解 → 這個機制設計來做什麼？為什麼現在失效？
    ├─ 4. 修正 → 讓機制正常工作，不是繞過它
    └─ 5. 驗證 → 確認其他保護都沒被動到
```

## 紅旗（立刻停下來診斷）

- 修法裡有 `true`、`none`、`() => undefined`、`disabled`、`skip`
- 修法是在「關掉」或「繞過」一個既有保護
- 錯誤消失了，但不確定為什麼消失
- 同一段程式碼被反覆改了三種不同方向

## 鐵則

1. **讓報錯消失 ≠ 修好了**。報錯是症狀，不是病因。
2. **拔掉保護機制之前，先問它本來在擋什麼。**
3. **如果解釋不清楚為什麼會失敗，就不要改。**
4. **修正後，確認沒有副作用（其他功能、安全性、效能）。**
