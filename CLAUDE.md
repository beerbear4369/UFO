# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UFO² (The Desktop AgentOS)** is a multi-agent framework for automating Windows tasks through natural language. It combines hierarchical agent orchestration (HostAgent → AppAgents), hybrid Windows integration (UIA/Win32/WinCOM), retrieval-augmented generation (RAG), and speculative execution to achieve autonomous, multi-application task completion.

**Key Papers:**
- UFO² (2025): https://arxiv.org/abs/2504.14603
- UFO v1 (2024): https://arxiv.org/abs/2402.07939

## Environment Setup

**Requirements:**
- Windows OS >= 10
- Python >= 3.10 (tested with 3.10, 3.11)
- Virtual environment located at `.venv/`

**Activation:**
```powershell
# PowerShell
.venv\Scripts\Activate.ps1

# Command Prompt
.venv\Scripts\activate.bat

# Bash/Git Bash
source .venv/Scripts/activate
```

**Installation:**
```powershell
pip install -r requirements.txt
```

**Configuration:**
Copy `ufo/config/config.yaml.template` to `ufo/config/config.yaml` and configure LLM credentials for both HOST_AGENT and APP_AGENT sections.

## Running UFO

**Basic execution:**
```powershell
# Interactive mode
python -m ufo --task <task_name>

# Direct request
python -m ufo --task <task_name> -r "<your request>"

# Execute pre-recorded plan
python -m ufo --task <task_name> --mode follower

# Batch processing
python -m ufo --task <task_name> --mode batch_normal

# OpenAI Operator mode
python -m ufo --task <task_name> --mode operator
```

**Logs location:** `./ufo/logs/<task_name>/`
- `screenshots/` - Annotated UI captures per step
- `steps_*.json` - Execution trace with timing
- `requests_*.json` - LLM request/response history with costs
- `evaluations_*.json` - Task success evaluations

## Architecture Overview

### Multi-Agent Hierarchy

```
Entry Point (python -m ufo)
    ↓
SessionFactory → Session/Round/Processor Pattern
    ↓
HostAgent (Orchestrator)
    ├─ Parses natural language requests
    ├─ Launches applications
    ├─ Creates and coordinates AppAgents
    └─ Manages global finite state machine (FSM)
    ↓
AppAgent(s) (One per application)
    ├─ ReAct Loop: Perceive → Retrieve → Reason → Act
    ├─ Multimodal perception (UI tree + screenshots)
    ├─ RAG integration (4 knowledge sources)
    └─ Executes via Puppeteer
    ↓
Puppeteer (Command Orchestrator)
    ├─ Speculative batch validation (reduces LLM calls by 51%)
    ├─ ReceiverManager routes to:
    │   ├─ ControlReceiver (UIA clicks)
    │   ├─ AppAPIReceiver (Word/Excel/PowerPoint native APIs)
    │   ├─ WinCOMReceiver (Windows COM)
    │   └─ BashReceiver (shell commands)
```

### Core Components

**State Management:**
- `AgentStateManager` - Singleton registry with decorator-based lazy-loaded states
- `AgentState` - Base class for all FSM states
- States define transitions via `next_state()` and `next_agent()`

**Memory Systems:**
- `Blackboard` - Shared memory for inter-agent communication
- `Context` - Enum-based named storage passed through all processors (tracks costs, logs, app info, trajectories)
- `Memory` - Per-agent episodic memory

**Processing Pipeline (BaseProcessor):**
1. `print_step_info()` - Log step number
2. `capture_screenshot()` - PhotographerFacade → annotated images
3. `get_ui_control_info()` - ControlInspectorFacade → UI tree JSON
4. `get_prompt_message()` - Construct LLM prompt with RAG
5. `get_response()` - LLM call
6. `update_cost()` - Token tracking
7. `parse_response()` - JSON schema parsing
8. `execute_action()` - Puppeteer command execution
9. `update_memory()` - Update Blackboard + Memory
10. `log_execution()` - Write step logs

### Windows Integration

**Hybrid Control Detection** (`ufo/automator/ui_control/`):
- **UIA (Primary):** Structured accessibility tree with text/semantic/icon filtering
- **Win32 (Fallback):** Legacy Windows API for older applications
- **OmniParser (Vision):** LLM-based grounding for custom controls

**Configuration:** `CONTROL_BACKEND: [uia, win32, omniparser]` in config_dev.yaml

**Puppeteer Command Pattern:**
- `AppPuppeteer` manages command queue and receiver selection
- `ReceiverManager` factory dispatches to appropriate backend
- Supports speculative execution: batch predict + live validate

### RAG (Knowledge Substrate)

Four knowledge sources retrieved on-the-fly during AppAgent reasoning:

| Source | Storage | Indexer |
|--------|---------|---------|
| **Offline Docs** | `vectordb/docs/{app}/` FAISS | `learner/` module |
| **Online Search** | Live Bing API | N/A |
| **Experience** | `vectordb/experience/` | Self-learning from trajectories |
| **Demonstrations** | `vectordb/demonstration/` YAML | `record_processor/` |

**Enabling RAG:** Set `RAG_OFFLINE_DOCS`, `RAG_ONLINE_SEARCH`, `RAG_EXPERIENCE`, `RAG_DEMONSTRATION` to `true` in config_dev.yaml

**Creating offline docs:**
```powershell
# Index help documents for an application
python learner/indexer.py --app <app_name>
```

**Processing demonstrations:**
```powershell
# Convert Steps Recorder output to structured YAML
python record_processor/record_processor.py --input <path_to_xml>
```

### LLM Integration

**Supported models** (`ufo/llm/`):
- OpenAI (GPT-4o, GPT-4-turbo, GPT-3.5-turbo)
- Azure OpenAI
- Claude (Anthropic)
- Gemini (Google)
- Qwen
- Ollama (local models)
- Custom models via `model_worker/` template

**Configuration pattern:**
```yaml
API_TYPE: "openai" | "aoai" | "claude" | "gemini" | "qwen" | "ollama" | "custom"
VISUAL_MODE: true  # Enable vision models
API_BASE: "<endpoint>"
API_KEY: "<key>"
API_MODEL: "<model_name>"
```

**Vision support:** Controlled by `VISUAL_MODE` config, used for screenshot understanding and control detection.

**Cost tracking:** Automatic via `config/config_prices.yaml` with per-request token calculation.

## Directory Structure

| Directory | Purpose |
|-----------|---------|
| `ufo/` | Main agent system |
| `ufo/agents/` | Multi-agent framework (HostAgent, AppAgent, EvaluationAgent, states, memory) |
| `ufo/automator/` | Windows OS integration (Puppeteer, UI control, screenshots, native APIs) |
| `ufo/module/` | Orchestration (Session/Round/Processor pattern, Context, SessionFactory) |
| `ufo/rag/` | Knowledge substrate (4 retriever types) |
| `ufo/llm/` | LLM abstraction layer (multi-model support) |
| `ufo/prompter/` | Prompt construction for each agent type |
| `ufo/config/` | Configuration files (config.yaml, config_dev.yaml, config_prices.yaml) |
| `dataflow/` | LAM training pipeline (template→prefill→filter) |
| `learner/` | Knowledge indexing (help docs → FAISS) |
| `record_processor/` | Demonstration parsing (Steps Recorder → actions) |
| `vectordb/` | Persistent storage (FAISS indexes, demonstrations) |
| `model_worker/` | Custom LLM integration template |
| `documents/` | Documentation source files |

## Key Files for Understanding

**Entry points:**
- `ufo/ufo.py` - Main entry point
- `ufo/__main__.py` - Module entry (`python -m ufo`)

**Session orchestration:**
- `ufo/module/basic.py` - Session/Round/BaseProcessor pattern
- `ufo/module/sessions/session.py` - SessionFactory + concrete session types

**Agents:**
- `ufo/agents/agent/host_agent.py` - HostAgent (orchestrator)
- `ufo/agents/agent/app_agent.py` - AppAgent (executor)
- `ufo/agents/processors/basic.py` - Processing pipeline template

**State management:**
- `ufo/agents/states/basic.py` - State pattern infrastructure
- `ufo/agents/memory/blackboard.py` - Inter-agent communication
- `ufo/module/context.py` - Shared state management

**Automation:**
- `ufo/automator/puppeteer.py` - Command orchestration
- `ufo/automator/ui_control/controller.py` - UI interaction
- `ufo/automator/ui_control/inspector.py` - UI tree inspection

**RAG:**
- `ufo/rag/retriever.py` - RAG factory and retrievers
- `learner/indexer.py` - Knowledge base creation
- `record_processor/record_processor.py` - Demonstration processing

**Configuration:**
- `ufo/config/config.py` - Configuration singleton
- `ufo/config/config.yaml.template` - Configuration template

## Design Patterns Used

| Pattern | Usage |
|---------|-------|
| **State Machine** | Agent FSM via AgentStateManager + AgentState |
| **Factory** | AgentFactory, RetrieverFactory, ReceiverFactory, SessionFactory |
| **Singleton** | Config, StateManager, SessionFactory |
| **Blackboard** | Inter-agent shared memory |
| **Command** | Puppeteer automation (CommandBasic + ReceiverBasic) |
| **Template Method** | BaseProcessor (subclasses customize steps) |
| **Strategy** | Multiple RAG sources, control backends, LLM models |

## Task Execution Modes

1. **Normal (`--mode normal`)**: Interactive - user enters request, full agent execution with tracing
2. **Follower (`--mode follower`)**: Executes pre-recorded JSON plan for reproducible automation
3. **Batch (`--mode batch_normal`)**: Processes multiple task files with completion tracking
4. **Operator (`--mode operator`)**: Uses OpenAI's CUA Operator as AppAgent backend

## Important Configuration Options

**Feature flags in `config_dev.yaml`:**
- `CONTROL_BACKEND: [uia, win32, omniparser]` - Hybrid detection methods
- `BATCH_ACTIONS: true` - Enable speculative execution (51% fewer LLM calls)
- `RAG_*: true` - Enable specific RAG knowledge sources
- `EVA_SESSION: true` - Enable post-task evaluation
- `PRINT_LOG: true` - Console logging verbosity

**Performance tuning:**
- `MAX_STEP` - Maximum steps per AppAgent before escalation
- `MAX_TRAJECTORY_LENGTH` - Context window for trajectory memory
- `TOP_K_*` - Number of RAG results to retrieve per source

## AppAgent Creation

UFO² supports custom AppAgents for specific applications. See `documents/docs/creating_app_agent/` for guides:

1. **Basic AppAgent:** Extend `AppAgent` class in `ufo/agents/agent/app_agent.py`
2. **Custom Receiver:** Implement `ReceiverBasic` for native API integration
3. **Custom Prompts:** Override `AppAgentPrompter` in `ufo/prompter/`
4. **Registration:** Add to `AgentFactory` in `ufo/agents/agent/basic.py`

**Example native API receivers:**
- `ufo/automator/app_apis/word/` - Microsoft Word COM automation
- `ufo/automator/app_apis/excel/` - Microsoft Excel API
- `ufo/automator/app_apis/powerpnt/` - PowerPoint automation

## Benchmarking

UFO² is evaluated on:
- **Windows Agent Arena (WAA):** 154 real Windows tasks across 15 applications
- **OSWorld (Windows):** 49 cross-application tasks

Integration repositories available separately. See documentation at https://microsoft.github.io/UFO/

## Repository Context

- **Original upstream:** `microsoft/UFO` (https://github.com/microsoft/UFO)
- **Current organization:** `kukutech-io/ProcAgent` (https://github.com/kukutech-io/ProcAgent)
- **Git remotes:**
  - `origin` → kukutech-io/ProcAgent (your fork)
  - `upstream` → microsoft/UFO (original source)

## Development Workflow

1. **Activate virtual environment:** `source .venv/Scripts/activate`
2. **Install dependencies:** `pip install -r requirements.txt`
3. **Configure LLMs:** Copy and edit `ufo/config/config.yaml` from template
4. **Run task:** `python -m ufo --task test_task -r "your request"`
5. **Review logs:** Check `./ufo/logs/test_task/` for execution details

## Debugging

**Enable verbose logging:**
- Set `PRINT_LOG: true` in `config_dev.yaml`
- Check `ufo/logs/<task_name>/steps_*.json` for detailed execution trace

**Common issues:**
- **UIA detection failures:** Try adding `win32` or `omniparser` to `CONTROL_BACKEND`
- **LLM API errors:** Verify credentials in `config.yaml` and check `requests_*.json`
- **Action execution failures:** Review screenshots in `ufo/logs/<task_name>/screenshots/`

**Cost monitoring:**
- Execution costs logged per request in `requests_*.json`
- Pricing configured in `config/config_prices.yaml`
