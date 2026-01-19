# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build and Run Commands

```bash
# Install dependencies
npm install

# Run development server (frontend on port 5173)
npm run dev

# Build for production (outputs to ./web directory)
npm run build

# Lint the code
npm run lint

# Run Python unit tests
pytest tests/

# Run specific test file
pytest tests/<package>/test_<name>.py

# Run integration tests (requires deployed actions)
pytest tests/<package>/test_<name>_int.py
```

## Architecture

This is a **full-stack serverless application** combining:
- **Frontend**: React + TypeScript (Vite) with Radix UI components
- **Backend**: Python serverless functions (OpenServerless/OpenWhisk)
- **API Proxy**: Vite dev server proxies `/api/my/*` to serverless backend

### Frontend Structure (TypeScript/React)

```
src/
├── components/       # Reusable UI components (Radix UI)
│   └── ui/          # shadcn/ui components
├── pages/           # Route-level page components
├── hooks/           # Custom React hooks
└── App.tsx          # Root component with routing
```

**Key Frontend Patterns**:
- Uses React Router with HashRouter
- Tanstack Query for data fetching
- shadcn/ui component library with Radix UI primitives
- Tailwind CSS for styling
- Path alias `@/` resolves to `src/`

**API Calls**: Frontend calls backend at `/api/my/<package>/<action>.<ext>` (proxied in dev, deployed in production)

### Backend Structure (Python)

```
packages/<package>/<action>/
├── __main__.py      # Entry point with annotations (#--kind, #--web)
└── <action>.py      # Main function implementation

tests/<package>/
├── test_<action>.py      # Unit tests (direct function calls)
└── test_<action>_int.py  # Integration tests (HTTP requests)
```

**Serverless Function Pattern**:
- `__main__.py` contains annotations and calls `<action>.main(args)`
- All business logic goes in `<action>.py`, NOT in `__main__.py`
- Functions receive/return JSON objects (never primitives or arrays)
- Annotations (`#--kind`, `#--web`, `#--param`) configure deployment

**Example Function**:
```python
# packages/hello/world/__main__.py
#--kind python:default
#--web true
import world
def main(args):
  return { "body": world.main(args) }

# packages/hello/world/world.py
def main(args):
  input = args.get("input", "world")
  return { "output": f"Ciao, {input}" }
```

## Backend Development Workflow

### Creating New Backend Functions

```bash
# Create new function
ops tk new <package>/<action>

# Deploy single action
ops ide deploy <package>/<action>

# Deploy all actions
ops ide deploy
```

### Testing Workflow

1. **Unit tests**: Run directly without deployment
   ```bash
   pytest tests/<package>/test_<action>.py
   ```

2. **Integration tests**: Deploy first, then test HTTP endpoint
   ```bash
   ops ide deploy <package>/<action>
   pytest tests/<package>/test_<action>_int.py
   ```

**Environment**: Integration tests use `OPSDEV_HOST` from environment for API host.

### Backend Limitations

- **Only these Python libraries** are available (no pip/requirements.txt):
  - `requests`, `openai`, `psycopg`, `boto3`, `pymilvus`, `redis`
- Code must be in `packages/` and tests in `tests/`
- Functions are stateless with no shared code between actions

## Frontend Development

**Build Output**: Vite builds to `web/` directory (embedded for production deployment)

**Component Guidelines**:
- Use functional components with hooks
- Follow shadcn/ui patterns for UI components
- Use Tanstack Query for data fetching
- Routes defined in [src/App.tsx](src/App.tsx)

## Service Integrations

Backend functions can integrate with Redis, PostgreSQL, S3, or Milvus by:
1. Adding `#--param` annotations in `__main__.py` to inject environment variables
2. Accessing credentials via `args.get("VAR", os.getenv("VAR"))` pattern

See [.github/copilot-instructions.md](.github/copilot-instructions.md) for detailed service integration patterns.

## Key Conventions

- **Frontend**: TypeScript only, code in `src/`, static assets in `public/`
- **Backend**: Python only, code in `packages/`, tests in `tests/`
- **API URLs**: Backend actions accessible at `/api/my/<package>/<action>.<ext>`
- **Deployment**: Frontend builds to `web/`, backend deploys to OpenServerless
- **Authentication**: Web actions are publicly accessible (implement custom auth if needed)
