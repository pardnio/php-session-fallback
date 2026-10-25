# php-session-fallback - 技術文件

最後更新：2026-10-25

> 返回 [README](./README.zh.md)

## 前置需求

- PHP 8.0 以上
- Composer
- Redis 模式：PHP `redis` 擴充（提供 `redis` session save handler）
- 提供 `isConnected()` 的 Redis 客戶端物件，例如 [`pardnchiu/redis-cli`](https://github.com/pardnio/php-redis-cli) 的 `pardnchiu\RDB`
- 檔案模式：`$_SERVER["DOCUMENT_ROOT"]/storage/sessions` 可建立且可寫入

## 安裝

### 使用 Composer

```bash
composer require pardnchiu/session
composer require pardnchiu/redis-cli
```

### 從原始碼

```bash
git clone https://github.com/pardnio/php-session-fallback.git
cd php-session-fallback
composer install
```

套件以 PSR-4 將 `PD\` 對應至 `src/`。

## 設定

### 環境變數（Redis 模式）

| 變數 | 必填 | 說明 |
|------|------|------|
| `REDIS_HOST` | 是 | Redis 主機；原始碼的 `localhost` 預設值因型別轉換優先序不會生效 |
| `REDIS_PORT` | 是 | Redis 埠號；`6379` 預設值同上不會生效 |
| `REDIS_PASSWORD` | 是 | Redis 密碼，無密碼時設為空字串 |

save path 組成為 `tcp://<host>:<port>?auth=<password>&database=0&persistent=1`。

### 建構子會套用的 ini 設定

| 設定 | 值 |
|------|----|
| `session.cookie_lifetime` | `604800`（7 天） |
| `session.gc_maxlifetime` | `604800`（7 天） |
| `display_errors`／`display_startup_errors` | `1` |
| `error_reporting` | `E_ALL` |
| `session.save_handler` | `redis` 或 `files` |
| `session.save_path` | Redis URI 或 `DOCUMENT_ROOT/storage/sessions` |

正式環境若不希望輸出錯誤，需在建構之後自行覆寫 `display_errors`。

## 使用方式

### 基礎

建構子會呼叫 `session_start()`，必須在輸出任何內容之前建立。

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use PD\Session;
use pardnchiu\RDB;

$session = new Session(new RDB());

$session->set("user_id", 123);
$userId = $session->get("user_id");

$session->delete("user_id");
```

### 進階：登入、逾時與登出

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use PD\Session;
use pardnchiu\RDB;

$session = new Session(new RDB());
ini_set("display_errors", "0");

if (session_status() !== PHP_SESSION_ACTIVE) {
    http_response_code(503);
    error_log('[Unavailable] session: storage/sessions not writable');
    exit;
}

function login(Session $session, int $userId): void
{
    $session->regenerateId();
    $session->set("user_id", $userId);
}

if (time() - $session->getCreatedTime() > 86400) {
    $session->destroy();
    header("Location: /login");
    exit;
}
```

檔案模式下目錄不可用時建構子會直接返回而不啟動 Session，可用 `session_status()` 判斷。

## API 參考

### Session

| 方法 | 簽章 | 說明 |
|------|------|------|
| 建構子 | `__construct($REDIS_CLIENT = null)` | 依 `$REDIS_CLIENT->isConnected()` 選擇 Redis 或檔案儲存並啟動 Session；必須傳入物件 |
| `set` | `set(string $key, mixed $value): void` | 寫入 `$_SESSION[$key]` |
| `get` | `get(string $key): mixed` | 讀取 `$_SESSION[$key]`，不存在回傳 `null` |
| `delete` | `delete(string $key): void` | 移除單一鍵 |
| `destroy` | `destroy(): void` | 呼叫 `session_destroy()` |
| `regenerateId` | `regenerateId(): void` | 呼叫 `session_regenerate_id(true)`，同時刪除舊 Session |
| `getId` | `getId(): string` | 回傳目前 Session ID |
| `getCreatedTime` | `getCreatedTime(): int` | 回傳 `$_SESSION["created_time"]`，首次呼叫時以 `time()` 建立 |

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
