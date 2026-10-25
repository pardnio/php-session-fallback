# php-session-fallback - Documentation

Last updated: 2026-10-25

> Back to [README](../README.md)

## Prerequisites

- PHP 8.0 or higher
- Composer
- Redis mode: the PHP `redis` extension (provides the `redis` session save handler)
- A Redis client object exposing `isConnected()`, such as `pardnchiu\RDB` from [`pardnchiu/redis-cli`](https://github.com/pardnio/php-redis-cli)
- File mode: `$_SERVER["DOCUMENT_ROOT"]/storage/sessions` creatable and writable

## Installation

### Using Composer

```bash
composer require pardnchiu/session
composer require pardnchiu/redis-cli
```

### From Source

```bash
git clone https://github.com/pardnio/php-session-fallback.git
cd php-session-fallback
composer install
```

The package maps `PD\` to `src/` via PSR-4.

## Configuration

### Environment Variables (Redis Mode)

| Variable | Required | Description |
|----------|----------|-------------|
| `REDIS_HOST` | Yes | Redis host; the `localhost` default in the source never applies because of cast precedence |
| `REDIS_PORT` | Yes | Redis port; the `6379` default never applies for the same reason |
| `REDIS_PASSWORD` | Yes | Redis password; set it to an empty string when there is none |

The save path is built as `tcp://<host>:<port>?auth=<password>&database=0&persistent=1`.

### ini Settings Applied by the Constructor

| Setting | Value |
|---------|-------|
| `session.cookie_lifetime` | `604800` (7 days) |
| `session.gc_maxlifetime` | `604800` (7 days) |
| `display_errors` / `display_startup_errors` | `1` |
| `error_reporting` | `E_ALL` |
| `session.save_handler` | `redis` or `files` |
| `session.save_path` | Redis URI or `DOCUMENT_ROOT/storage/sessions` |

Override `display_errors` after construction if production must not print errors.

## Usage

### Basic

The constructor calls `session_start()`, so create it before sending any output.

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

### Advanced: Login, Timeout, and Logout

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

In file mode the constructor returns without starting a session when the directory is unusable; check it with `session_status()`.

## API Reference

### Session

| Method | Signature | Description |
|--------|-----------|-------------|
| Constructor | `__construct($REDIS_CLIENT = null)` | Picks Redis or file storage from `$REDIS_CLIENT->isConnected()` and starts the session; an object must be passed |
| `set` | `set(string $key, mixed $value): void` | Writes `$_SESSION[$key]` |
| `get` | `get(string $key): mixed` | Reads `$_SESSION[$key]`, or `null` when missing |
| `delete` | `delete(string $key): void` | Removes one key |
| `destroy` | `destroy(): void` | Calls `session_destroy()` |
| `regenerateId` | `regenerateId(): void` | Calls `session_regenerate_id(true)`, deleting the old session |
| `getId` | `getId(): string` | Returns the current session ID |
| `getCreatedTime` | `getCreatedTime(): int` | Returns `$_SESSION["created_time"]`, initialized with `time()` on first call |

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
