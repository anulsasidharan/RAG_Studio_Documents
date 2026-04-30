# RAG Studio Architecture

## System Overview

```
Frontend (Next.js) → API Gateway (FastAPI) → Core Services
                                           → Agent System
                                           → Databases
```

## Components

### Frontend Layer
- Next.js 14 with App Router
- TypeScript + Tailwind CSS
- Zustand state management

### Backend Layer
- FastAPI (Python)
- LangGraph (Agents)
- LangChain (RAG)

### Data Layer
- PostgreSQL (metadata)
- Redis (cache)
- Qdrant (vectors)

## Data Flow

1. User → Frontend
2. Frontend → API
3. API → Services
4. Services → Databases
5. Results → Frontend

See [detailed architecture](docs/architecture/overview.md)
