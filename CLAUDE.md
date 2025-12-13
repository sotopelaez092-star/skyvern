# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Skyvern is a browser automation platform that uses LLMs and computer vision to interact with websites. It automates browser-based workflows using Vision LLMs and Playwright, replacing brittle XPath-based automation with AI-powered navigation.

**Key Capabilities:**
- Operates on websites it has never seen before
- Resistant to website layout changes
- Applies single workflows across multiple websites
- Uses LLM reasoning for complex interactions

## Development Commands

### Python Backend Commands
- **Install dependencies**: `uv sync` (use `uv sync --group dev` for development)
- **Run Skyvern service**: `skyvern run all` (starts both backend and UI)
- **Run backend only**: `skyvern run server`
- **Run UI only**: `skyvern run ui`
- **Check status**: `skyvern status`
- **Stop services**: `skyvern stop all`
- **Quickstart**: `skyvern quickstart` (for first-time setup with DB migrations)
- **Initialize LLM**: `skyvern init llm` (configure LLM provider)
- **Initialize browser**: `skyvern init browser` (setup Playwright)

### Code Quality & Testing
- **Lint**: `ruff check` and `ruff format`
- **Type checking**: `mypy skyvern`
- **Run all tests**: `pytest tests/`
- **Run unit tests**: `pytest tests/unit_tests/`
- **Pre-commit hooks**: `pre-commit run --all-files`

### Frontend Commands (in skyvern-frontend/)
- **Install dependencies**: `npm install`
- **Development**: `npm run dev`
- **Build**: `npm run build`
- **Lint**: `npm run lint`
- **Format**: `npm run format`

### Database Management
- **Run migrations**: `alembic upgrade head`
- **Create migration**: `alembic revision --autogenerate -m "description"`

## Architecture Overview

### Directory Structure

```
/skyvern/                    # Main Python package
  /forge/                    # Core API and execution engine (FastAPI)
    /sdk/                    # SDK components
      /api/llm/              # LLM abstraction layer (multi-provider)
      /db/                   # Database client and models
      /routes/               # API endpoints
      /workflow/models/      # Workflow definitions (blocks, parameters)
    agent.py                 # Core agent orchestration
    api_app.py               # FastAPI application setup
    forge_app.py             # ForgeApp service container
  /webeye/                   # Browser automation layer
    /actions/                # Action execution (click, type, etc.)
    /scraper/                # DOM scraping and page analysis
    browser_factory.py       # Playwright browser creation
    browser_manager.py       # Browser lifecycle management
  /services/                 # Business logic services
    task_v2_service.py       # Current task execution (recommended)
    workflow_service.py      # Workflow CRUD and management
    browser_session_service.py
  /schemas/                  # Pydantic data models
  /cli/                      # Command-line interface
  /client/                   # Generated Python SDK client
  /library/                  # Library utilities for SDK users
  /utils/                    # Utility functions
  /errors/                   # Custom exception definitions
  config.py                  # Settings (Pydantic BaseSettings)
/skyvern-frontend/           # React/TypeScript UI
  /src/
    /routes/                 # Page components
    /components/             # Reusable UI components
    /store/                  # Zustand state management
    /api/                    # API client (Axios, React Query)
/skyvern-ts/                 # TypeScript SDK client
/tests/                      # Test suite
  /unit_tests/               # Unit tests
/alembic/                    # Database migrations
/integrations/               # Third-party integrations
  /langchain/                # LangChain agent integration
  /llama_index/              # LlamaIndex integration
  /mcp/                      # Model Context Protocol
  /n8n/, /make/              # Workflow automation integrations
```

### Core Components

#### ForgeApp Service Container (`skyvern/forge/forge_app.py`)
Central dependency injection container providing singletons:
- `DATABASE` - AgentDB database client
- `STORAGE` - Artifact storage (local or S3)
- `BROWSER_MANAGER` - Browser automation
- `LLM_API_HANDLER` - Main LLM provider
- `SECONDARY_LLM_API_HANDLER` - Lightweight LLM for simple tasks
- `WORKFLOW_SERVICE` - Workflow orchestration
- `CREDENTIAL_VAULT_SERVICES` - Credential management

#### Agent System (`skyvern/forge/agent.py`)
LLM-powered decision-making engine that:
1. Takes screenshots of current page state
2. Scrapes DOM into structured element tree
3. Sends to LLM with task context and visual data
4. Receives action instructions
5. Executes actions via browser automation
6. Iterates until task completion

#### Browser Automation (`skyvern/webeye/`)
- `browser_factory.py` - Creates Playwright browser instances
- `actions/handler.py` - Executes browser actions (click, type, etc.)
- `actions/action_types.py` - ActionType enum: CLICK, INPUT_TEXT, UPLOAD_FILE, SELECT_OPTION, CHECKBOX, HOVER, WAIT, EXTRACT, SCROLL, KEYPRESS, DRAG
- `scraper/scraper.py` - DOM scraping orchestrator
- `scraper/scraped_page.py` - Represents scraped page with element tree

#### LLM Integration (`skyvern/forge/sdk/api/llm/`)
Multi-provider abstraction supporting:
- OpenAI (GPT-4, GPT-4o, o3, o4-mini)
- Anthropic (Claude 3.5, 3.7, 4)
- Azure OpenAI
- Google Vertex AI / Gemini
- AWS Bedrock
- Ollama (local models)
- OpenRouter
- OpenAI-compatible endpoints

Key files:
- `api_handler_factory.py` - Creates LLM handlers
- `config_registry.py` - Model configurations
- `api_handler.py` - Base handler interface

### Workflow System

#### Block Types (`skyvern/forge/sdk/workflow/models/block.py`)
- **Task Blocks**: TaskBlock, TaskV2Block, NavigationBlock, ActionBlock, ExtractionBlock, LoginBlock, ValidationBlock
- **Control Flow**: ForLoopBlock, ConditionalBlock, WaitBlock
- **Code**: CodeBlock, TextPromptBlock
- **File Operations**: FileDownloadBlock, FileUploadBlock, FileParserBlock, PDFParserBlock
- **External**: HttpRequestBlock, SendEmailBlock, DownloadToS3Block, UploadToS3Block

#### Parameters (`skyvern/forge/sdk/workflow/models/parameter.py`)
- WORKFLOW - Input parameters
- CONTEXT - Reference to another block's output
- OUTPUT - Final workflow output
- AWS_SECRET, AZURE_VAULT_CREDENTIAL - Cloud secrets
- BITWARDEN_*, ONEPASSWORD - Password manager credentials

### Data Flow

**Task Execution:**
1. Create task via API → Store in database → Queue for execution
2. Create browser session → Navigate to URL
3. Loop: Screenshot → Scrape DOM → LLM decision → Execute actions
4. Save results → Webhook notification → Cleanup

**Workflow Execution:**
1. Create WorkflowRun → Validate definition → Initialize context
2. For each block: Resolve parameters → Execute → Store outputs
3. Return final outputs → Webhook notification

## Development Notes

### Environment Setup
- Requires Python 3.11+ (works with 3.12, not ready for 3.13)
- Requires Node.js and npm
- Uses UV for Python dependency management
- PostgreSQL database (via Docker or local install)
- Browser dependencies installed via Playwright

### Key Environment Variables

**LLM Configuration:**
- `LLM_KEY` - Active model (e.g., `OPENAI_GPT4O`, `ANTHROPIC_CLAUDE3.7_SONNET`)
- `SECONDARY_LLM_KEY` - Lightweight model for simple tasks
- Provider-specific: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `AZURE_API_KEY`, etc.

**Browser:**
- `BROWSER_TYPE` - "chromium-headful" or "chromium-headless"
- `BROWSER_WIDTH`, `BROWSER_HEIGHT` - Viewport (default 1920x1080)
- `BROWSER_ACTION_TIMEOUT_MS` - Action timeout (5000ms)

**Database:**
- `DATABASE_STRING` - PostgreSQL connection string

**Storage:**
- `SKYVERN_STORAGE_TYPE` - "local" or "s3"
- `ARTIFACT_STORAGE_PATH` - Local artifact directory

**Task Execution:**
- `MAX_STEPS_PER_RUN` - Max steps per task (10)
- `MAX_STEPS_PER_TASK_V2` - Max steps for TaskV2 (25)
- `DEBUG_MODE` - Enable debug logging

### Testing Strategy
- Unit tests in `tests/unit_tests/`
- Use `pytest` with async support (`pytest-asyncio`)
- Mock external services with `moto` (AWS) and `aiosqlite`
- Test fixtures for common test data

### Code Style

**Python:**
- Ruff for linting and formatting (configured in pyproject.toml)
- Line length: 120 characters
- Target Python 3.11
- Type hints required throughout
- Async/await for all I/O operations
- Structured logging with `structlog`
- Pydantic for data validation

**TypeScript (Frontend):**
- ESLint + Prettier
- Strict TypeScript configuration
- React hooks over class components
- Zustand for state management
- React Query for server state

**Import Organization:**
1. Future imports
2. Standard library
3. Third-party packages
4. Local imports
(Sorted alphabetically within sections)

### Architectural Patterns

- **Async-First**: All I/O operations are async
- **Dependency Injection**: ForgeApp singleton container
- **Protocol-Based Interfaces**: BrowserState, BrowserManager as protocols
- **Service Layer**: Separation of concerns with dedicated service classes
- **Factory Pattern**: LLMAPIHandlerFactory, StorageFactory

### Error Handling
- Custom exception hierarchy based on `SkyvernException`
- Automatic retry on transient errors
- Max retries per action/step configurable
- Structured error logging

### Security Practices
- Credentials stored in vault (Bitwarden, Azure Key Vault, 1Password)
- Never store secrets in code or logs
- JWT tokens for API access
- Pydantic validation for all inputs
- Parameterized queries via SQLAlchemy

## Common Patterns

### Adding a New Workflow Block
1. Define block class in `skyvern/forge/sdk/workflow/models/block.py`
2. Add block type to `BlockType` enum
3. Implement `execute()` method
4. Register in block execution logic in `skyvern/services/block_service.py`
5. Add frontend support in `skyvern-frontend/src/routes/workflows/`

### Adding a New LLM Provider
1. Add configuration in `skyvern/forge/sdk/api/llm/config_registry.py`
2. Implement handler if needed in `api_handler.py`
3. Add environment variables in `skyvern/config.py`
4. Update `skyvern/cli/llm_setup.py` for CLI support

### Adding a New API Endpoint
1. Create route in `skyvern/forge/sdk/routes/`
2. Add to router in `skyvern/forge/sdk/routes/routers.py`
3. Define request/response models in `skyvern/schemas/`
4. Implement service logic in `skyvern/services/`

## Key Files Reference

| Component | Key Files |
|-----------|-----------|
| Agent Core | `forge/agent.py`, `forge/agent_functions.py` |
| LLM Layer | `forge/sdk/api/llm/api_handler_factory.py` |
| Browser | `webeye/browser_manager.py`, `webeye/actions/handler.py` |
| Workflows | `forge/sdk/workflow/models/block.py`, `forge/sdk/workflow/models/parameter.py` |
| Database | `forge/sdk/db/client.py`, `forge/sdk/db/models.py` |
| API | `forge/api_app.py`, `forge/sdk/routes/` |
| Config | `config.py`, `.env.example` |
| CLI | `cli/commands.py`, `cli/run_commands.py` |
| Frontend | `skyvern-frontend/src/` |
| Tests | `tests/unit_tests/` |
