# 01 · End-to-End Overview

從 LINE 訊息進入到 Supabase 寫入完成的完整流程，呈現 human-in-the-loop 與 knowledge feedback 兩個核心迴圈。

---

## Diagram

![End-to-end overview](../images/01-overview.svg)

---

## 流程說明

### Stage 1 · Inbound

客戶從 LINE 發訊息進入。LINE Messaging API 透過 webhook 把訊息推送到 n8n。對外曝露走 Cloudflared tunnel — 因為 LINE webhook 必須打到固定可到達的 endpoint，但內網沒有固定 IP，所以用 Cloudflared 解決。

### Stage 2 · Preprocessing

n8n 接收後做幾件事：
- **追蹤碼 extraction**：用 regex 從訊息文字抓出物流追蹤碼
- **客戶上下文 aggregation**：用追蹤碼或 LINE userId（fallback）查出該客戶的歷史訊息
- **Contact count**：算出 72 小時內這個客戶發過幾次訊息
- **Prompt assembly**：把所有上下文組裝成給 LLM 的 prompt

### Stage 3 · LLM Classification

送到 Groq 上的 llama-3.3-70b。Prompt 要求 LLM 回傳 6 個 structured fields：

| Field | Purpose |
|---|---|
| `intent` | 訊息意圖分類（查詢、客訴、投訴、其他） |
| `confidence` | 0–1 之間的信心分數 |
| `tone` | 客戶語氣（neutral / frustrated / angry） |
| `force_human` | bool，是否強制轉人工 |
| `priority` | low / medium / high |
| `suggested_reply` | 建議回覆內容 |

回傳後做 JSON schema validation，不合 schema 視為 classification 失敗。

### Stage 4 · 8-Layer Decision

進入決策層，依序通過 8 個 gate（詳見 [02-decision-tree.md](02-decision-tree.md)）。每個 gate 失敗時導向不同處理路徑。

### Stage 5 · Output

三個出口：
- **自動回覆**：通過所有 gate 後直接推送到 LINE
- **轉人工**：通知客服 panel，客戶會收到「客服稍後回覆」的訊息
- **緊急升級**：通知主管 panel，標記為高優先

### Stage 6 · Logging

所有路徑收斂到 Supabase 寫入。logging 內容包含：
- 原始訊息與時間戳
- LLM 回傳的 6 個 fields
- 走過哪些 gate、結果如何
- 最終出口路徑
- 客服處理結果（如果走人工）

### Stage 7 · Knowledge Feedback

人工處理過的訊息會被標記為 training candidate。定期 review 後可以：
- 補充 prompt 的 few-shot examples
- 調整 confidence threshold
- 新增 force_human 規則

---

## Two Loops

這個系統有兩個核心迴圈：

**Human-in-the-loop**：LLM 不是黑盒子做決定，而是把不確定的 case 交給人類。Confidence < 0.5、tone = angry、contact_count ≥ 3、reply generation 失敗等 — 這些 signal 都會觸發人類介入。

**Knowledge feedback loop**：人類處理過的訊息變成下次系統判斷的依據。這個迴圈讓系統會隨時間變準，而不是部署後就停止學習。

---

## 為什麼要把 logging 放在三個出口的下游而非 LLM 的下游？

直覺上你會想在 LLM 回傳後立刻 log，因為那是最完整的 metadata。但這個設計選擇是刻意的：

**只有在「決策結果產生」之後 log，才能保留完整的因果鏈**。如果在 LLM 後 log，再在出口 log 一次，就會有兩筆紀錄，分析時要 join；如果合併成一筆，那就應該等出口確定後再寫。

這是工管思維 — **資料的單位應該對應業務的單位**。一筆 logging row = 一次完整的客戶互動處理，而不是 LLM 的一次推論。
