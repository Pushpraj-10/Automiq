
# Automiq Services

This repo contains three services that run together:

- Frontend (Next.js UI)
- Backend (control plane API)
- Executor (execution plane + workers)

See [SYSTEM.md](SYSTEM.md) for the architecture diagram.

## Service map

### Frontend (Next.js)

- Location: [frontend](frontend)
- Purpose: UI + API client
- Talks to: Backend over HTTP
- Env: `NEXT_PUBLIC_API_BASE_URL`

### Backend (Control Plane)

- Location: [backend](backend)
- Purpose: API, auth, workflows, dispatch
- Talks to: Executor over gRPC
- Stores: PostgreSQL

### Executor (Execution Plane)

- Location: [executor](executor)
- Purpose: queue + step execution
- Talks to: Backend over gRPC callbacks
- Stores: MongoDB

## Local dev quick start

1) Fill env files:

- [backend/.env.example](backend/.env.example) -> backend/.env
- [executor/.env.example](executor/.env.example) -> executor/.env
- [frontend/.env.example](frontend/.env.example) -> frontend/.env

2) Start backend:

```bash
cd backend
pnpm install
pnpm prisma generate
pnpm prisma migrate dev
pnpm dev
```

3) Start executor:

```bash
cd executor
pnpm install
pnpm dev
```

4) Start frontend:

```bash
cd frontend
pnpm install
pnpm dev
```

## gRPC ports (defaults)

- Executor gRPC bind: 0.0.0.0:50051
- Backend gRPC bind: 0.0.0.0:50052

These defaults are configurable in each service env.

