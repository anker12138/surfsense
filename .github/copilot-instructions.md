# Copilot Instructions for SurfSense

This repository is a monorepo containing a FastAPI backend, a Next.js frontend, and a browser extension.

## 🏗️ Project Structure

- **`surfsense_backend/`**: Python/FastAPI backend service.
  - Uses PostgreSQL with `pgvector` for vector search.
  - Handles AI logic with LangChain/LangGraph.
  - Uses Celery/Redis for background tasks.
- **`surfsense_web/`**: Next.js 15 web application.
  - Uses React 19, TypeScript, and Tailwind CSS 4.
  - Uses Shadcn UI for components.
- **`surfsense_browser_extension/`**: Browser extension built with Plasmo.
  - Collects browsing history for SurfSense.

## 🛠️ Build, Test, and Lint Commands

### Backend (`surfsense_backend/`)
- **Package Manager**: Standard Python (managed via `pyproject.toml`).
- **Run Server**: `python main.py --reload` (enables hot reloading).
- **Run Celery Worker**: `celery -A app.celery_app worker --loglevel=info`
- **Run Celery Beat**: `celery -A app.celery_app beat --loglevel=info`
- **Lint/Format**: `ruff check .` / `ruff format .`
- **Database Migrations**: `alembic upgrade head` (configured in `alembic.ini`).

### Docker
- **Run All Services**: `docker-compose up --build`
- **Services**:
  - `backend`: FastAPI app (Port 8000)
  - `frontend`: Next.js app (Port 3000)
  - `db`: PostgreSQL with pgvector (Port 5432)
  - `redis`: Redis broker (Port 6379)
  - `pgadmin`: DB Management (Port 5050)
- **Note**: `celery_worker`, `celery_beat`, and `flower` are commented out in `docker-compose.yml` by default and may need to be uncommented for full background processing in Docker.

### Frontend (`surfsense_web/`)
- **Package Manager**: `pnpm`
- **Dev Server**: `pnpm dev` (Runs `next dev --turbopack`)
- **Build**: `pnpm build`
- **Lint**: `pnpm lint`
- **Format**: `pnpm format` (Uses Biome: `biome check --write ./`)
- **Database (Drizzle)**:
  - Generate: `pnpm db:generate`
  - Migrate: `pnpm db:migrate`
  - Studio: `pnpm db:studio`

### Browser Extension (`surfsense_browser_extension/`)
- **Package Manager**: `pnpm`
- **Dev Server**: `pnpm dev` (Runs `plasmo dev`)
- **Build**: `pnpm build`
- **Package**: `pnpm package`

## 🧩 High-Level Architecture

- **Hybrid Search**: Combines vector similarity (pgvector) and full-text search using Reciprocal Rank Fusion (RRF).
- **RAG Pipeline**:
  - Uses `Chonkie` for chunking.
  - Uses `LiteLLM` for model integration.
  - Implements hierarchical indices (2-tiered RAG).
- **Connectors**:
  - ETL pipeline integrations for various sources (Slack, Notion, Google, etc.).
  - Located in `surfsense_backend/app/connectors/`.
- **Authentication**: `FastAPI Users` with JWT/OAuth.
- **Task Queue**: Celery with Redis broker for async processing (document ingestion, podcast generation).

## 🔑 Key Conventions

### General
- **Git**: Use Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`).
- **Paths**: Always use relative paths or project-root relative paths when referencing files.

### Backend (Python)
- **Style**: Follow PEP 8. Use `ruff` for enforcement.
- **Type Hints**: Strongly encouraged for all function signatures.
- **Async**: Use `async/await` for all I/O bound operations (DB, API calls).
- **Dependency Injection**: Use FastAPI's dependency injection system (`Depends`).
- **Configuration**: Use `pydantic-settings` or `.env` via `python-dotenv`.

### Frontend (TypeScript/React)
- **Style**: Use Functional Components with Hooks.
- **UI Components**: Use Shadcn UI components from `@/components/ui`.
- **State Management**: Use `Nuqs` for URL search params state. Use `Jotai` for global client state.
- **Data Fetching**: Use `@tanstack/react-query` for server state.
- **Styling**: Use Tailwind CSS utility classes. Avoid inline styles.
- **Forms**: Use `react-hook-form` with `zod` schema validation.
- **Internationalization**: Use `next-intl`.

### Browser Extension
- **Framework**: Follow Plasmo framework patterns for content scripts and background service workers.
- **Messaging**: Use Plasmo's messaging API for communication between content scripts and background scripts.
