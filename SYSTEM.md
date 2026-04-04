        SYSTEM ARCHITECTURE (CURRENT)

                     ┌───────────────────────────────┐
                     │        Client (UI/API)        │
                     └──────────────┬────────────────┘
                                    │ HTTP
                                    ▼
                 ┌─────────────────────────────────────────────────────┐
                 │                 BACKEND (CONTROL PLANE)             │
                 │                     Express + TS                    │
                 │                                                     │
                 │  Modules:                                           │
                 │  - auth / users                                     │
                 │  - billing (Stripe)                                 │
                 │  - workflow + actions definition                    │
                 │  - dispatcher (enqueue + execution status)          │
                 │                                                     │
                 │  Responsibilities:                                  │
                 │  - Own workflow/action definitions                  │
                 │  - Create Execution records                         │
                 │  - Dispatch execution requests to executor (gRPC)   │
                 │  - Receive execution + step updates from executor   │
                 └──────────────┬───────────────────────┬──────────────┘
                                │                       │
                                │ gRPC (ExecutorService)│
                                │ EnqueueExecution      │
                                ▼                       │
                 ┌─────────────────────────────────────────────────────┐
                 │                EXECUTOR (EXECUTION PLANE)           │
                 │                     Express + TS                    │
                 │                                                     │
                 │  Components:                                        │
                 │  - gRPC server: accepts enqueue requests            │
                 │  - queue persistence: MongoDB collections           │
                 │    (Execution, WorkflowVersion, ExecutionQueue,     │
                 │     ExecutionStep)                                  │
                 │  - batch worker: claims queued jobs, executes steps │
                 │  - action handlers: http, webhook, email, delay     │
                 │                                                     │
                 │  Responsibilities:                                  │
                 │  - Own queue + step execution lifecycle             │
                 │  - Execute jobs in batches                          │
                 │  - Report status back to backend via gRPC callbacks │
                 └──────────────┬───────────────────────┬──────────────┘
                                │                       ▲
                                │ MongoDB               │ gRPC (DispatcherService)
                                ▼                       │ UpdateExecutionStatus
                    ┌────────────────────┐              │ UpdateExecutionStepStatus
                    │      MongoDB       │──────────────┘
                    │  (executor store)  │
                    └────────────────────┘

                    ┌────────────────────┐
                    │    PostgreSQL      │
                    │  (backend store)   │
                    │ users/workflows/   │
                    │ actions/execution  │
                    └────────────────────┘

        NOTES
        - Backend and executor are separate processes.
        - Communication between backend and executor is gRPC in both directions.
        - Backend remains source of truth for workflow/action definitions and high-level execution status.
        - Executor remains source of truth for queue/job and step execution internals.