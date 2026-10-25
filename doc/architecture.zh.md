# php-session-fallback - 架構

最後更新：2026-10-25

> 返回 [README](./README.zh.md)

## 概覽

```mermaid
graph TB
    A[呼叫端] --> B[PD\Session 建構子]
    R[Redis 客戶端 如 pardnchiu\RDB] --> B
    ENV[REDIS_HOST / PORT / PASSWORD] --> B
    B --> C{isConnected?}
    C -->|是| D[PHP redis save handler]
    C -->|否| E[DOCUMENT_ROOT/storage/sessions]
    D --> F[session_start]
    E --> F
    F --> G[$_SESSION 存取方法]
```

## Module: Session

`PD\Session` 在建構時完成儲存選擇與啟動，其餘方法皆為 `$_SESSION` 與 PHP 原生 session 函式的薄包裝。

```mermaid
graph TB
    subgraph Session
        INI[套用 ini：7 天存活 / 錯誤顯示] --> SW{isConnected?}
        SW -->|是| RU[組 tcp:// URI 並設 save_handler=redis]
        SW -->|否| DIR{目錄可建立且可寫?}
        DIR -->|否| RET[直接返回 不啟動 Session]
        DIR -->|是| FU[設 save_handler=files]
        RU --> ST[session_start]
        FU --> ST
        ST --> API[set / get / delete]
        ST --> SEC[regenerateId / destroy]
        ST --> META[getId / getCreatedTime]
    end
```

## State Machine

```mermaid
stateDiagram-v2
    [*] --> 未啟動
    未啟動 --> 啟用中: 建構子 session_start
    未啟動 --> 未啟動: 檔案目錄不可用
    啟用中 --> 啟用中: regenerateId（新 ID，刪舊資料）
    啟用中 --> 已銷毀: destroy
    已銷毀 --> [*]
```

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
