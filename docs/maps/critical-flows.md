# Critical Flows

Status: current

These sequences identify the lifecycle owner and handoff points for workflows
that cross multiple Modules. They are navigation maps, not substitutes for
contracts or implementation.

## Account import

Manual JSON/credential import writes through the account API and
`AccountService`. CPA and Sub2API use one shared remote-job lifecycle.

```mermaid
sequenceDiagram
    participant UI as Accounts/Settings runtime
    participant API as accountImportsApi/accountsApi
    participant Route as api/accounts.py
    participant Source as CPAImportService or Sub2APIImportService
    participant Job as RemoteAccountImportJob
    participant Accounts as AccountService
    participant DB as Application Database

    UI->>API: Start import with target Account Group
    API->>Route: Validated request
    alt Manual import
        Route->>Accounts: Save one batch
    else CPA or Sub2API
        Route->>Source: start_import(...)
        Source->>Job: reserve and start worker
        Job->>Source: Fetch remote account payloads
        Job->>Accounts: finish with one normalized batch
        UI->>API: Poll import job
    end
    Accounts->>DB: Account Repository transaction
    DB-->>Accounts: Stored batch result
    Accounts-->>Route: Backend action result
    Route-->>API: Stable response projection
    API-->>UI: Result or polled progress
```

`RemoteImportJobCoordinator` enforces the shared active-job rule. Both remote
sources finish through `RemoteAccountImportJob.finish`; source adapters do not
implement a second save lifecycle. The UI owns polling and presentation only.

### Sub2API 凭据恢复

Sub2API 导入只持久化 AT、ID Token 和远端来源绑定，不保存 RT。AT 已过期或被上游拒绝时，`AccountService` 通过 `Sub2APIImportService` 绑定的恢复 Adapter 按来源账号重新读取凭据；失败次数逐次持久化，累计三次后永久停止，手工重新导入同一来源账号会轮换 AT 并重置状态。

```mermaid
sequenceDiagram
    participant Request as 上游请求
    participant Accounts as AccountService
    participant Source as Sub2API Adapter
    participant DB as Application Database
    Request->>Accounts: AT 过期或被拒绝
    Accounts->>Source: 按 server_id/account_id 获取最新凭据
    alt 获取成功
        Source-->>Accounts: AT + ID Token
        Accounts->>DB: 原子轮换凭据并清空 RT
        Accounts-->>Request: 使用新 AT 重试
    else 获取失败
        Accounts->>DB: 持久化失败次数
        Accounts-->>Request: 三次后永久停止自动恢复
    end
```

## Studio Image Task

```mermaid
sequenceDiagram
    participant Studio as Studio image runtime
    participant API as imageTasksApi
    participant Route as api/image_tasks.py
    participant Task as ImageTaskService
    participant Upstream as Image generation services
    participant Assets as ImageStorageService
    participant Store as data/image_tasks.json

    Studio->>API: Create generation or edit task
    API->>Route: Owner-scoped request
    Route->>Task: Create task
    Task->>Store: Persist queued/running projection
    Task->>Upstream: Execute attempts and account switches
    Upstream->>Assets: Publish successful Image Assets
    Task->>Store: Persist terminal result
    Studio->>API: Poll owner-scoped task list
    API-->>Studio: Final task projection and asset URLs
```

`ImageTaskService` owns task admission, transitions, attempts, resumption, and
terminal state. `ImageStorageService` exclusively owns Image Asset catalogue and
storage mutations. Studio owns browser conversation presentation and polling,
not task truth.

## Call Record, live monitor, and metrics

```mermaid
sequenceDiagram
    participant Route as Public API route
    participant Call as LoggedCall
    participant Live as RealtimeMonitorService
    participant Logs as CallRecordService
    participant Metrics as DashboardMetricsService
    participant DB as Application Database

    Route->>Call: Wrap invocation
    opt Instrumented image request
        Call->>Live: start and stage events
    end
    Call->>Call: Resolve final result and diagnostics
    opt Instrumented image request
        Call->>Live: finish Active Request
    end
    Call->>Logs: Add final Call Record
    Logs->>DB: CallRecordRepository
    Metrics->>Logs: Incremental synchronization
    Metrics->>DB: DashboardMetricsRepository
```

The Call Record is durable final truth. Active Requests and live operations are
process-memory observations and may disappear on restart. The Dashboard Metric
Projection reads only Call Records after its stored sequence during normal
synchronization, updates affected hourly rows atomically, and rebuilds only when
an explicit repository operation rotates the Call Record generation or an
aggregate table is structurally incompatible. User-facing log deletion and
automatic retention preserve the cursor, so the 30-day history remains
independent of the shorter Call Record retention window. If only the projection
state table is recreated, startup checkpoints the current cursor without
deleting compatible hourly aggregates.

## Dashboard projection

```mermaid
sequenceDiagram
    participant Page as Dashboard/useDashboardPage
    participant API as statsApi
    participant Route as GET /api/dashboard
    participant View as build_dashboard_view
    participant Metrics as DashboardMetricsService
    participant Accounts as AccountService
    participant Runtime as RuntimeEnvironment/RealtimeMonitor

    Page->>API: Fetch dashboard immediately or on timer
    API->>Route: Authenticated request
    Route->>View: Build one response
    View->>Metrics: Read stored multi-range snapshot
    View->>Accounts: Read Account Pool stats
    View->>Runtime: Read runtime and live-operation snapshots
    View-->>Page: Stable Dashboard projection
```

The backend owns metric definitions, available ranges, units, health, and
runtime values. Account cards and Active Request counts read current state;
24-hour call cards and all chart ranges come from the same 30-day hourly
projection; runtime-environment samples are not persisted. The page selects one
returned range and owns refresh timing, retained snapshots, chart construction,
animation, layout, and responsive behavior.

## Managed-container online update

```mermaid
sequenceDiagram
    participant Shell as AppShell update overlay
    participant API as versionApi
    participant Status as UpdateStatusService
    participant Task as UpdateService
    participant Release as GitHub Release
    participant Runtime as Managed runtime directory

    Shell->>API: Check update status
    API->>Status: Read latest release projection
    Status->>Release: Fetch release metadata/changelog
    Shell->>API: Confirm and start update
    API->>Task: Start one update task
    Task->>Release: Resolve and download validated asset
    Task->>Runtime: Install files and synchronize dependencies
    Task-->>Shell: Persisted progress via polling
    Task->>Runtime: Exit for container restart
```

`UpdateService` owns the complete update task and rejects unsupported runtime
modes. The frontend only confirms, starts, polls, and renders progress. Update
task state is persisted in `data/update_task.json`; the container runtime owns
restarting the process after exit.
