# php-session-fallback - Architecture

Last updated: 2026-10-25

> Back to [README](../README.md)

## Overview

```mermaid
graph TB
    A[Caller] --> B[PD\Session constructor]
    R[Redis client e.g. pardnchiu\RDB] --> B
    ENV[REDIS_HOST / PORT / PASSWORD] --> B
    B --> C{isConnected?}
    C -->|Yes| D[PHP redis save handler]
    C -->|No| E[DOCUMENT_ROOT/storage/sessions]
    D --> F[session_start]
    E --> F
    F --> G[$_SESSION accessors]
```

## Module: Session

`PD\Session` picks storage and starts the session during construction; every other method is a thin wrapper over `$_SESSION` and native PHP session functions.

```mermaid
graph TB
    subgraph Session
        INI[Apply ini: 7-day lifetime / error display] --> SW{isConnected?}
        SW -->|Yes| RU[Build tcp:// URI and set save_handler=redis]
        SW -->|No| DIR{Directory creatable and writable?}
        DIR -->|No| RET[Return without starting session]
        DIR -->|Yes| FU[Set save_handler=files]
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
    [*] --> Inactive
    Inactive --> Active: constructor session_start
    Inactive --> Inactive: file directory unusable
    Active --> Active: regenerateId (new ID, old data deleted)
    Active --> Destroyed: destroy
    Destroyed --> [*]
```

***

©️ 2024 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
