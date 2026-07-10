---
name: root-cause-first
description: "Systematic debugging: understand before acting. A thinking framework for diagnosing and fixing problems without breaking things."
version: 2.0.0
tags: [debugging, methodology, thinking, logic]
triggers:
  - debug
  - fix
  - broken
  - not working
  - error
  - why
---

# Root Cause First — 先想清楚，再動手

## 五步思考框架

遇到任何問題，按這個順序走完再寫 code：

```
1. 這東西本來是幹嘛的？
   └─ 理解機制的設計意圖，不是只看它現在的行為

2. 什麼觸發了它失敗？
   └─ 重現 → 縮小範圍 → 找到最小觸發條件

3. 為什麼這個觸發會讓它失敗？
   └─ 這是核心。不是「報了什麼錯」，是「為什麼這個輸入會造成這個結果」

4. 怎麼讓它在這個情況下正常工作？
   └─ 修正機制本身，不是繞過它、關掉它、或用另一條路取代它

5. 修完之後，有沒有弄壞別的東西？
   └─ 安全性、效能、其他功能、邊界情況
```

## 三個反例（來自本專案實際踩坑）

| 症狀 | 錯的做法 | 跳過了哪步 | 正確做法 |
|------|----------|-----------|----------|
| 滾動卡死 | `events={() => undefined}` | 第 3 步 — 沒問 canvas 為什麼會吃事件 | canvas pointer-events: none |
| RLS 500 | `USING(true)` | 第 3 步 — 沒問 policy 為什麼遞迴 | SECURITY DEFINER function |
| 照片不顯示 | 調低 opacity | 第 3 步 — 沒問 z-index 疊層順序 | 修正層級關係 |

共同模式：**把警報器關掉，當作問題不存在。**

## 決策樹

```
收到問題
  │
  ├─ 知道這個機制怎麼運作嗎？
  │   ├─ 不知道 → 先讀原始碼/文件，不要猜
  │   └─ 知道 → 往下
  │
  ├─ 能穩定重現嗎？
  │   ├─ 不能 → 加 log，收集更多資訊
  │   └─ 能 → 往下
  │
  ├─ 理解為什麼這個輸入會造成這個結果嗎？
  │   ├─ 不理解 → 縮小範圍，隔離變數
  │   └─ 理解 → 往下
  │
  ├─ 修法是「修正」還是「繞過」？
  │   ├─ 繞過 → 停下來，這不是修好
  │   └─ 修正 → 往下
  │
  └─ 旁邊的東西都還正常嗎？
      ├─ 不確定 → 加測試/手動驗證
      └─ 正常 → 收工，記下來
```

## 紅旗詞彙（看到這些修法立刻警覺）

`true`, `false`, `none`, `undefined`, `skip`, `disabled`, `() => null`, `return`, `!!`, `try {} catch {}`（空的 catch）

這些詞本身沒問題，但如果是用來「讓某個保護機制失效」，就是紅旗。
