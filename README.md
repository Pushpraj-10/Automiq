
# Automiq Platform

Automiq is a multi-service system with a clear split between control plane and execution plane.

- Frontend: Next.js application for workflow authoring, dispatching, and monitoring.
- Backend: control plane API for auth, workflows, billing, events, and dispatch orchestration.
- Executor: execution plane worker service that runs workflow steps and reports progress.

Architecture overview and interaction diagram: [SYSTEM.md](SYSTEM.md)

## Table of contents

- [Architecture](#architecture)
- [Repository layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [Local development](#local-development)
- [Service command reference](#service-command-reference)
- [Environment reference](#environment-reference)
- [Dispatch and execution lifecycle](#dispatch-and-execution-lifecycle)
- [gRPC contracts](#grpc-contracts)
- [Realtime channels](#realtime-channels)
- [Testing and quality](#testing-and-quality)
- [Production hardening checklist](#production-hardening-checklist)
- [Troubleshooting](#troubleshooting)
- [Additional documentation](#additional-documentation)

## Architecture

| Service | Role | Primary tech | Persistence | Main protocols |
| --- | --- | --- | --- | --- |
| frontend | User-facing product UI | Next.js + React + Redux Toolkit | none (browser state) | HTTP + Socket.IO |
| backend | Control plane (auth, workflows, dispatch orchestration) | Express + TypeScript + Prisma | PostgreSQL | HTTP + gRPC + Socket.IO |
| executor | Execution plane (queue, worker, step execution) | Express + TypeScript | MongoDB | gRPC + HTTP |

### Data ownership model

- Backend is source of truth for users, workflows, actions, API keys, high-level execution records, and persisted callback step snapshots.
- Executor is source of truth for queued jobs and execution internals while work is running.
- Frontend is a client only and should not own durable execution state.

## Repository layout

```text
.
|- frontend/        # Next.js web app
|- backend/         # Control plane API + Prisma/Postgres
|- executor/        # Execution plane + Mongo worker
|- proto/           # Shared gRPC contract(s)
|- SYSTEM.md        # High-level architecture diagram
`- README.md        # This file
```

## Prerequisites

- Node.js 20+ recommended
- pnpm 9+ recommended (or npm equivalent commands)
- PostgreSQL running locally or remotely for backend
- MongoDB running locally or remotely for executor

## Local development

### 1) Create env files

Copy each example file:

- [backend/.env.example](backend/.env.example) -> backend/.env
- [executor/.env.example](executor/.env.example) -> executor/.env
- [frontend/.env.example](frontend/.env.example) -> frontend/.env

### 2) Install dependencies

```bash
cd backend && pnpm install
cd ../executor && pnpm install
cd ../frontend && pnpm install
```

### 3) Prepare backend database

```bash
cd backend
pnpm prisma generate
pnpm prisma migrate dev
```

### 4) Start services (use 3 terminals)

Terminal 1:

```bash
cd backend
pnpm dev
```

Terminal 2:

```bash
cd executor
pnpm dev
```

Terminal 3:

```bash
cd frontend
pnpm dev
```

### 5) Verify startup

- Frontend app: http://localhost:3000
- Backend base URL for frontend: http://localhost:4000 (from env example)
- Executor health: http://localhost:4001/health

Note: backend runtime falls back to port 3000 if `PORT` is not set, but local env example sets `PORT=4000` so frontend can use `NEXT_PUBLIC_API_BASE_URL=http://localhost:4000`.

## Service command reference

### Backend

Path: [backend](backend)

| Command | Purpose |
| --- | --- |
| pnpm dev | Start API in watch mode |
| pnpm build | Compile TypeScript |
| pnpm start | Run compiled server |
| pnpm type-check | TypeScript validation |
| pnpm test | Run all tests |
| pnpm test:integration | Run integration tests only |

### Executor

Path: [executor](executor)

| Command | Purpose |
| --- | --- |
| pnpm dev | Start executor in watch mode |
| pnpm build | Compile TypeScript |
| pnpm start | Run compiled server |
| pnpm type-check | TypeScript validation |
| pnpm test | Run all tests |
| pnpm test:integration | Run integration tests only |

### Frontend

Path: [frontend](frontend)

| Command | Purpose |
| --- | --- |
| pnpm dev | Start Next.js dev server |
| pnpm build | Build frontend |
| pnpm start | Run production build |
| pnpm lint | Lint source |
| pnpm type-check | TypeScript validation |

## Environment reference

### Frontend (core)

| Variable | Example | Purpose |
| --- | --- | --- |
| NEXT_PUBLIC_API_BASE_URL | http://localhost:4000 | Backend HTTP base URL |
| NEXT_PUBLIC_SOCKET_URL | http://localhost:4000 | Backend Socket.IO URL |
| NEXT_PUBLIC_EDITOR_SOCKET_ENABLED | true | Enable editor realtime syncing |

File: [frontend/.env.example](frontend/.env.example)

### Backend (core + gRPC)

| Variable | Purpose |
| --- | --- |
| DATABASE_URL | PostgreSQL connection string |
| PORT | Backend HTTP port |
| JWT_SECRET | API auth signing secret |
| EXECUTOR_SHARED_SECRET | Shared auth secret used for backend <-> executor gRPC calls |
| EXECUTOR_GRPC_ADDRESS | Address of executor gRPC server |
| DISPATCHER_GRPC_BIND | Bind address for backend callback gRPC server |
| EXECUTOR_GRPC_TLS_ENABLED | Enable TLS for backend gRPC client to executor |
| EXECUTOR_GRPC_CLIENT_CERT_PATH / EXECUTOR_GRPC_CLIENT_KEY_PATH | Optional mTLS client identity for backend -> executor |
| EXECUTOR_GRPC_RETRY_ATTEMPTS / EXECUTOR_GRPC_RETRY_BASE_MS | Retry policy for backend -> executor gRPC |
| EXECUTOR_GRPC_CIRCUIT_BREAKER_FAILURE_THRESHOLD / EXECUTOR_GRPC_CIRCUIT_BREAKER_RESET_MS | Resilience controls for backend -> executor gRPC |
| DISPATCHER_GRPC_TLS_ENABLED / DISPATCHER_GRPC_MTLS_ENABLED | TLS settings for backend callback gRPC server |
| DISPATCHER_GRPC_SERVER_CERT_PATH / DISPATCHER_GRPC_SERVER_KEY_PATH / DISPATCHER_GRPC_CA_CERT_PATH | Server cert, key, and CA cert for backend callback gRPC |

File: [backend/.env.example](backend/.env.example)

### Executor (core + gRPC + worker)

| Variable | Purpose |
| --- | --- |
| EXECUTOR_PORT | Executor HTTP port |
| MONGODB_URI | MongoDB connection string |
| EXECUTOR_GRPC_BIND | Bind address for executor gRPC server |
| BACKEND_GRPC_ADDRESS | Backend callback gRPC server address |
| EXECUTOR_SHARED_SECRET | Shared auth secret for gRPC callbacks |
| EXECUTOR_GRPC_TLS_ENABLED / EXECUTOR_GRPC_MTLS_ENABLED | TLS settings for executor gRPC server |
| EXECUTOR_GRPC_SERVER_CERT_PATH / EXECUTOR_GRPC_SERVER_KEY_PATH / EXECUTOR_GRPC_CA_CERT_PATH | Cert, key, and CA cert for executor gRPC server |
| BACKEND_GRPC_TLS_ENABLED | Enable TLS for executor callback client |
| BACKEND_GRPC_CLIENT_CERT_PATH / BACKEND_GRPC_CLIENT_KEY_PATH / BACKEND_GRPC_CA_CERT_PATH | Optional mTLS client identity for executor -> backend |
| BACKEND_GRPC_RETRY_ATTEMPTS / BACKEND_GRPC_RETRY_BASE_MS | Retry policy for executor callback gRPC |
| BACKEND_GRPC_CIRCUIT_BREAKER_FAILURE_THRESHOLD / BACKEND_GRPC_CIRCUIT_BREAKER_RESET_MS | Resilience controls for executor callback gRPC |
| WORKER_POLL_INTERVAL_MS / WORKER_BATCH_SIZE / QUEUE_LOCK_TIMEOUT_MS | Worker queue tuning |
| SENDGRID_* / SMTP_* | Optional email action transport settings |

File: [executor/.env.example](executor/.env.example)

## Dispatch and execution lifecycle

1. User edits and publishes a workflow in frontend.
2. Frontend calls backend APIs to save workflow definition and steps.
3. Frontend triggers dispatch enqueue.
4. Backend creates an execution record and calls executor via gRPC `EnqueueExecution`.
5. Executor stores execution and step work items in MongoDB.
6. Executor worker claims queued jobs and executes steps in order.
7. Executor sends callbacks to backend via gRPC:
	 - `UpdateExecutionStatus`
	 - `UpdateExecutionStepStatus`
8. Backend persists execution and step callback snapshots and emits realtime events.
9. Frontend receives status/step updates over socket and updates UI.

## gRPC contracts

- Proto file: [proto/dispatcher_executor.proto](proto/dispatcher_executor.proto)
- Backend -> Executor service: `ExecutorService`
- Executor -> Backend callback service: `DispatcherService`

Default local gRPC bind endpoints:

- Executor gRPC bind: `0.0.0.0:50051`
- Backend callback gRPC bind: `0.0.0.0:50052`

Both are configurable via env.

## Realtime channels

Backend socket gateway supports two realtime domains:

- Execution stream: execution room join + status/step events
- Editor collaboration: join, patch, ack, and conflict resolution events

Socket gateway implementation: [backend/src/modules/socket/socket.gateway.ts](backend/src/modules/socket/socket.gateway.ts)

## Testing and quality

Recommended local validation sequence:

```bash
cd backend && pnpm type-check && pnpm test
cd ../executor && pnpm type-check && pnpm test
cd ../frontend && pnpm type-check && pnpm lint
```

The backend test suite includes cross-service gRPC integration coverage for dispatch and callback flows.

## Production hardening checklist

- Use unique secrets for `JWT_SECRET` and `EXECUTOR_SHARED_SECRET` per environment.
- Enable TLS (and optionally mTLS) for both gRPC directions.
- Configure retry and circuit-breaker values based on expected network behavior.
- Set strict CORS origins in backend and socket config.
- Restrict database network access and enforce TLS on database connections.
- Add centralized logging and alerting for dispatch failures and callback failures.
- Back up PostgreSQL and MongoDB with tested restore procedures.

## Troubleshooting

### Frontend cannot call backend

- Check `NEXT_PUBLIC_API_BASE_URL` in [frontend/.env.example](frontend/.env.example).
- Confirm backend is running on the expected `PORT`.

### Dispatch enqueue fails or callbacks are unauthorized

- Ensure backend and executor share the same `EXECUTOR_SHARED_SECRET`.
- Confirm both gRPC addresses are reachable from each service process.

### gRPC TLS or mTLS errors

- Verify cert/key/CA file paths exist and are readable.
- For mTLS, provide both client cert and client key together.
- Ensure server cert CN/SAN matches the target address used by the client.

### Database startup issues

- Backend: verify `DATABASE_URL` and run Prisma migration commands.
- Executor: verify `MONGODB_URI` and database user permissions.

## Additional documentation

- System architecture: [SYSTEM.md](SYSTEM.md)
- Backend API reference: [backend/API.md](backend/API.md)
- Backend service notes: [backend/README.md](backend/README.md)
- Executor service notes: [executor/README.md](executor/README.md)
- Frontend implementation docs:
	- [frontend/README.md](frontend/README.md)
	- [frontend/FLOW.md](frontend/FLOW.md)
	- [frontend/PLAN.md](frontend/PLAN.md)

