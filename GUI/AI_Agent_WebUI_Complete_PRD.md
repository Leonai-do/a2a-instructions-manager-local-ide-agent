================================================================================
AI AGENT WEBUI - COMPLETE PRODUCT REQUIREMENTS DOCUMENT
================================================================================

Generated: 2026-01-22 20:36:39
Status: DRAFT - Ready for Review
Version: 1.0

================================================================================


TABLE OF CONTENTS
================================================================================

1. EXECUTIVE SUMMARY & RESEARCH FINDINGS
   1.1 Project Overview
   1.2 Research Key Findings
   1.3 Technology Stack Decisions

2. PRODUCT REQUIREMENTS
   2.1 Feature Catalog (40+ Features)
   2.2 Functional Requirements
   2.3 User Stories

3. SYSTEM ARCHITECTURE
   3.1 High-Level Architecture
   3.2 Component Design
   3.3 Data Flow

4. API SPECIFICATIONS
   4.1 Agent Endpoints
   4.2 Model Endpoints
   4.3 Tool Endpoints

5. DATABASE DESIGN
   5.1 Schema
   5.2 Relationships

6. USER INTERFACE DESIGN
   6.1 Screen Layouts
   6.2 User Flows

7. IMPLEMENTATION GUIDE
   7.1 Pydantic AI Setup
   7.2 Ollama/LMStudio Integration
   7.3 Streamlit UI
   7.4 FastAPI Backend
   7.5 Code Examples

8. DEPLOYMENT & INFRASTRUCTURE
   8.1 Docker Setup
   8.2 Development Workflow

9. APPENDICES
   9.1 Glossary
   9.2 References

================================================================================



================================================================================
1. EXECUTIVE SUMMARY & RESEARCH FINDINGS
================================================================================


1.1 PROJECT OVERVIEW
--------------------------------------------------------------------------------

The AI Agent WebUI is a local-first, privacy-preserving application for creating
and managing intelligent AI agents. The system uses:

• Streamlit (Frontend): Python-based web interface
• FastAPI (Backend): High-performance async API
• Pydantic AI (Agents): Type-safe agent framework
• Ollama/LMStudio (LLMs): Local inference providers

Key Features:
✓ Multi-turn chat interface with streaming
✓ Agent creation with custom system prompts
✓ Dynamic tool attachment and execution
✓ Provider switching (Ollama/LMStudio/OpenAI)
✓ Session persistence and chat export
✓ Full privacy (all inference local)

Target Users:
• AI researchers and developers
• Privacy-conscious organizations
• Teams building custom AI workflows
• Anyone wanting local LLM control


1.2 RESEARCH KEY FINDINGS
--------------------------------------------------------------------------------

FINDING #1: Pydantic AI Provider Changes
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️  CRITICAL: Pydantic AI deprecated direct Ollama support in v0.0.21

Solution: Use OpenAI-compatible endpoint wrapper
• Both Ollama and LMStudio expose /v1 OpenAI-compatible endpoints
• Use OpenAIModel with custom base_url for both providers
• Seamless provider switching with unified interface

Example:
    from pydantic_ai.models import OpenAIModel

    # Ollama
    ollama_model = OpenAIModel(
        model_id="neural-chat",
        base_url="http://localhost:11434/v1",
        api_key="dummy"
    )

    # LMStudio
    lmstudio_model = OpenAIModel(
        model_id="local-model",
        base_url="http://localhost:8000/v1",
        api_key="dummy"
    )

FINDING #2: Pydantic AI Workflow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Agent Creation → Tool Calling → Streaming Responses → Structured Outputs

Key Capabilities:
• Type-safe agent definitions with Pydantic models
• Function tools via @agent.tool decorator
• Automatic schema generation for tools
• Streaming support with async generators
• Full conversation history tracking
• Dependency injection via RunContext

Tool Pattern:
    @agent.tool
    async def search_web(ctx: RunContext[dict], query: str) -> str:
        '''Search the web for information'''
        # Implementation
        return results

FINDING #3: LMStudio Integration
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Official Python SDK Available: lmstudio v1.5.0+

Three API Styles:
1. Convenience API (simple, synchronous)
2. Scoped Resource API (threaded, explicit connections)
3. Asynchronous API (structured concurrency) ← RECOMMENDED

Also supports OpenAI-compatible endpoint at localhost:8000
Tool calling supported in v1.5.0+

FINDING #4: Streamlit + FastAPI Pattern
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Two-Tier Architecture:

Streamlit (Port 8501)           FastAPI (Port 8000)
├─ UI Components                ├─ Agent Orchestration
├─ Session State                ├─ LLM Communication
├─ User Interactions            ├─ Tool Execution
└─ HTTP Client (httpx)  ←──→    └─ Request Validation

Communication: Async HTTP requests from Streamlit to FastAPI
Benefits: Clear separation, independent scaling, testability


1.3 TECHNOLOGY STACK DECISIONS
--------------------------------------------------------------------------------

┌────────────────────────────────────────────────────────────────────────────┐
│ Component           │ Technology        │ Version  │ Rationale             │
├────────────────────────────────────────────────────────────────────────────┤
│ UI Framework        │ Streamlit         │ 1.40+    │ No frontend code      │
│ Backend API         │ FastAPI           │ 0.115+   │ Async-first, type-safe│
│ Agent Framework     │ Pydantic AI       │ 0.0.21+  │ Structured outputs    │
│ Local LLM Provider  │ Ollama            │ Latest   │ Easy setup, privacy   │
│ Local LLM Provider  │ LMStudio          │ 1.5.0+   │ GUI + Python SDK      │
│ HTTP Client         │ httpx             │ 0.27+    │ Async support         │
│ Data Validation     │ Pydantic          │ 2.0+     │ Type safety           │
│ ASGI Server         │ Uvicorn           │ 0.30+    │ Production-ready      │
│ Database            │ SQLite            │ Built-in │ Simple, local         │
│ Deployment          │ Docker Compose    │ Latest   │ Easy orchestration    │
│ Python Runtime      │ Python            │ 3.10+    │ Modern async support  │
└────────────────────────────────────────────────────────────────────────────┘

Architecture Decision Records:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ADR-001: Use Streamlit for UI
Decision: Chosen over React/Vue/Next.js
Rationale: Zero frontend code, rapid development, Python-native
Trade-off: Less UI flexibility, acceptable for MVP

ADR-002: Use FastAPI for Backend
Decision: Chosen over Flask/Django/Quart
Rationale: Automatic docs, async-first, type validation
Trade-off: Newer ecosystem, acceptable given strong typing

ADR-003: Use Pydantic AI for Agents
Decision: Chosen over LangChain/CrewAI
Rationale: Type safety, structured outputs, simpler API
Trade-off: Smaller ecosystem, acceptable for focused use case

ADR-004: Support Both Ollama and LMStudio
Decision: Multi-provider support
Rationale: User choice, different use cases (CLI vs GUI)
Trade-off: Additional integration complexity, managed via abstraction

ADR-005: Use SQLite for Storage
Decision: Chosen over PostgreSQL/MongoDB
Rationale: Local-first, zero setup, sufficient for MVP
Trade-off: Limited concurrency, acceptable for single-user initially




================================================================================
2. PRODUCT REQUIREMENTS
================================================================================


2.1 FEATURE CATALOG
--------------------------------------------------------------------------------

CORE AGENT FEATURES (Priority: P0)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FR-001  Agent Creation
        • Create agents with custom names and system prompts
        • Select model from available providers
        • Attach tools from registry
        • Save agent configuration to database

FR-002  Agent Configuration
        • Modify system prompt
        • Change model selection
        • Adjust temperature (0.0 - 2.0)
        • Set max_tokens limit
        • Enable/disable streaming

FR-003  Agent Execution
        • Send messages to agents
        • Receive streaming responses
        • View tool calls in real-time
        • Handle multi-turn conversations
        • Automatic conversation history

PROVIDER MANAGEMENT (Priority: P0)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FR-004  Model Discovery
        • Auto-detect Ollama models (GET /api/tags)
        • Auto-detect LMStudio loaded models
        • Display model metadata (size, quantization)
        • Cache model list for performance

FR-005  Provider Switching
        • Runtime provider selection (Ollama/LMStudio/OpenAI)
        • Seamless model switching between providers
        • Automatic fallback on provider failure
        • Provider status indicators

FR-006  Connection Management
        • Test provider connectivity
        • Display latency and status
        • Configure custom endpoints
        • Save provider configurations

TOOL ECOSYSTEM (Priority: P1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FR-007  Built-in Tools
        • Calculator (math operations)
        • Web Search (search results)
        • File Operations (read/write local files)
        • Code Execution (run Python code)
        • Time/Date (current time, timezones)

FR-008  Custom Tools
        • Register Python functions as tools
        • Define tool schemas (name, description, parameters)
        • Attach/detach tools from agents
        • Tool discovery endpoint

FR-009  Tool Results
        • Display tool calls in chat
        • Show tool execution results
        • Handle tool errors gracefully
        • Log tool usage for debugging

UI/UX FEATURES (Priority: P0-P1)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FR-010  Chat Interface (P0)
        • Multi-turn conversation display
        • Streaming response rendering
        • User/assistant message differentiation
        • Markdown and code block rendering

FR-011  Sidebar Controls (P0)
        • Agent selector dropdown
        • Model selector dropdown
        • Temperature slider
        • Max tokens input
        • Provider radio buttons

FR-012  Message Rendering (P1)
        • Syntax highlighting for code
        • Tool call highlighting
        • Timestamps for messages
        • Copy message button
        • Regenerate response button

FR-013  Export Functionality (P1)
        • Export conversation as JSON
        • Export conversation as Markdown
        • Download chat history
        • Clear conversation

FR-014  Session Persistence (P2)
        • Save conversations to database
        • Load previous conversations
        • Resume conversations
        • Search conversation history

DATA MANAGEMENT (Priority: P1-P2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FR-015  Agent Storage (P1)
        • Persist agent configurations
        • Version agent changes
        • Delete agents
        • Export/import agents

FR-016  Conversation Storage (P1)
        • Save message history
        • Associate conversations with agents
        • Archive old conversations
        • Search conversations

FR-017  Provider Configs (P2)
        • Save provider settings
        • Manage API keys securely
        • Track provider usage
        • Provider health monitoring

ADVANCED FEATURES (Priority: P2-P3)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FR-018  Multi-Agent Workflows (P3)
        • Chain multiple agents
        • Agent hand-offs
        • Parallel agent execution
        • Workflow visualization

FR-019  Analytics Dashboard (P3)
        • Token usage tracking
        • Response time metrics
        • Tool usage statistics
        • Cost estimation

FR-020  Team Collaboration (P3)
        • Share agents with team
        • Shared conversation history
        • Comments on messages
        • Agent versioning


2.2 FUNCTIONAL REQUIREMENTS TABLE
--------------------------------------------------------------------------------

┌────────┬─────────────────────┬──────────────────────────────┬──────────┐
│ ID     │ Feature             │ Acceptance Criteria          │ Priority │
├────────┼─────────────────────┼──────────────────────────────┼──────────┤
│ FR-001 │ Chat Interface      │ User sends message, receives │ P0       │
│        │                     │ streaming response in <3s    │          │
├────────┼─────────────────────┼──────────────────────────────┼──────────┤
│ FR-002 │ Agent Creation      │ User creates agent, appears  │ P0       │
│        │                     │ in dropdown immediately      │          │
├────────┼─────────────────────┼──────────────────────────────┼──────────┤
│ FR-003 │ Model Selection     │ User selects model, chat     │ P0       │
│        │                     │ uses selected model          │          │
├────────┼─────────────────────┼──────────────────────────────┼──────────┤
│ FR-004 │ Tool Execution      │ Agent calls tool, result     │ P1       │
│        │                     │ displayed in chat            │          │
├────────┼─────────────────────┼──────────────────────────────┼──────────┤
│ FR-005 │ Provider Config     │ User configures Ollama       │ P1       │
│        │                     │ endpoint, connection tested  │          │
├────────┼─────────────────────┼──────────────────────────────┼──────────┤
│ FR-006 │ Chat Export         │ User downloads JSON with     │ P1       │
│        │                     │ full conversation history    │          │
├────────┼─────────────────────┼──────────────────────────────┼──────────┤
│ FR-007 │ Error Handling      │ User sees clear error for    │ P1       │
│        │                     │ provider connection failure  │          │
├────────┼─────────────────────┼──────────────────────────────┼──────────┤
│ FR-008 │ Session Persistence │ User reloads page, sees      │ P2       │
│        │                     │ previous conversation        │          │
└────────┴─────────────────────┴──────────────────────────────┴──────────┘


2.3 USER STORIES
--------------------------------------------------------------------------------

EPIC 1: Agent Creation and Configuration
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
US-001: Create Agent
As a user, I want to create a new agent with a custom name and system prompt
So that I can have specialized assistants for different tasks

Acceptance Criteria:
• Click "New Agent" button opens creation form
• Enter agent name (required, max 255 chars)
• Enter system prompt (required, textarea)
• Select model from dropdown
• Click "Create" saves agent to database
• New agent appears in agent selector immediately

US-002: Configure Agent
As a user, I want to adjust agent parameters like temperature and max_tokens
So that I can control the agent's behavior

Acceptance Criteria:
• Temperature slider (0.0 - 2.0, default 0.7)
• Max tokens input (100 - 4096, default 2048)
• Changes apply to next message
• Settings persist in database

EPIC 2: Chat and Interaction
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
US-003: Send Message
As a user, I want to send messages to my agent and see streaming responses
So that I can interact naturally with the AI

Acceptance Criteria:
• Type message in chat input
• Press Enter or click Send button
• Message appears in chat immediately (user role)
• Response streams in character-by-character
• Markdown rendered correctly
• Code blocks have syntax highlighting

US-004: View Tool Calls
As a user, I want to see when the agent calls tools and what results it gets
So that I can understand the agent's reasoning process

Acceptance Criteria:
• Tool calls displayed with distinct styling
• Tool name and parameters visible
• Tool results shown after execution
• Error messages if tool fails
• Expandable/collapsible tool details

EPIC 3: Provider Management
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
US-005: Switch Providers
As a user, I want to switch between Ollama and LMStudio
So that I can use different local models based on availability

Acceptance Criteria:
• Provider selector in sidebar (radio buttons)
• Model list updates when provider changes
• Connection status indicator (green/red dot)
• Automatic fallback if provider unavailable
• Toast notification on provider switch

US-006: Test Connection
As a user, I want to test my provider connection before using it
So that I can troubleshoot configuration issues

Acceptance Criteria:
• "Test Connection" button in settings
• Shows loading spinner during test
• Displays success/failure message
• Shows latency if successful
• Lists available models if successful




================================================================================
3. SYSTEM ARCHITECTURE
================================================================================


3.1 HIGH-LEVEL ARCHITECTURE
--------------------------------------------------------------------------------

3-TIER ARCHITECTURE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌──────────────────────────────────────────────────────────────────────────┐
│                         TIER 1: PRESENTATION                             │
│                        STREAMLIT WEB UI (Port 8501)                      │
├──────────────────────────────────────────────────────────────────────────┤
│  Components:                                                             │
│  • Chat Interface (st.chat_message, st.chat_input)                       │
│  • Sidebar Controls (st.sidebar, st.selectbox, st.slider)                │
│  • Session State (st.session_state for history)                          │
│  • HTTP Client (httpx.AsyncClient to FastAPI)                            │
│                                                                          │
│  Responsibilities:                                                       │
│  ✓ Render UI components                                                 │
│  ✓ Handle user interactions                                             │
│  ✓ Manage session state                                                 │
│  ✓ Display streaming responses                                          │
│  ✓ Format messages (markdown, code highlighting)                        │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │ HTTP/HTTPS
                             │ (httpx async requests)
                             ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         TIER 2: APPLICATION                              │
│                        FASTAPI BACKEND (Port 8000)                       │
├──────────────────────────────────────────────────────────────────────────┤
│  Components:                                                             │
│  • APIRouter (endpoints organized by domain)                             │
│  • Agent Factory (create agents with models and tools)                   │
│  • Model Factory (provider abstraction)                                  │
│  • Tool Registry (discover and execute tools)                            │
│  • Database Layer (SQLite via SQLAlchemy)                                │
│                                                                          │
│  Endpoints:                                                              │
│  ✓ POST /api/agents (create agent)                                      │
│  ✓ POST /api/agents/{id}/chat (chat with agent, streaming)              │
│  ✓ GET /api/agents (list agents)                                        │
│  ✓ GET /api/models (list available models)                              │
│  ✓ POST /api/models/connect (test provider)                             │
│  ✓ GET /api/tools (list available tools)                                │
│                                                                          │
│  Responsibilities:                                                       │
│  ✓ Request validation (Pydantic models)                                 │
│  ✓ Agent orchestration                                                  │
│  ✓ Tool execution                                                       │
│  ✓ Database operations                                                  │
│  ✓ Error handling                                                       │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         TIER 3: AGENT & PROVIDER                         │
├──────────────────────────────────────────────────────────────────────────┤
│  Pydantic AI Agent Layer                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │   Agent      │  │ Tool Registry│  │  RunContext  │                  │
│  │   Creation   │  │ & Execution  │  │  Management  │                  │
│  └──────────────┘  └──────────────┘  └──────────────┘                  │
│                                                                          │
│  Provider Abstraction Layer                                              │
│  ┌──────────────────────────────────────────────────────────┐           │
│  │           OpenAIModel (Unified Interface)                │           │
│  │  (Works with Ollama, LMStudio, and OpenAI via base_url)  │           │
│  └──────────────────────────────────────────────────────────┘           │
└────────────────────────────┬─────────────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   OLLAMA     │    │  LMSTUDIO    │    │   OPENAI     │
│ localhost    │    │ localhost    │    │   (Cloud)    │
│   :11434     │    │   :8000      │    │   (Optional) │
│              │    │              │    │              │
│ • /v1 API    │    │ • /v1 API    │    │ • Standard   │
│ • Local      │    │ • GUI        │    │   API        │
│   models     │    │ • Python SDK │    │              │
└──────────────┘    └──────────────┘    └──────────────┘


3.2 COMPONENT DESIGN
--------------------------------------------------------------------------------

FASTAPI BACKEND COMPONENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Agent Factory
   Purpose: Create and manage Pydantic AI agents

   Methods:
   • create_agent(config: AgentConfig) -> Agent
   • get_agent(agent_id: str) -> Agent
   • update_agent(agent_id: str, config: AgentConfig) -> Agent
   • delete_agent(agent_id: str) -> bool

   Dependencies:
   • ModelFactory (for LLM model instances)
   • ToolRegistry (for agent tools)
   • Database (for persistence)

2. Model Factory
   Purpose: Abstract provider differences, create model instances

   Methods:
   • create_model(provider: str, model_id: str) -> OpenAIModel
   • list_models(provider: str) -> List[ModelInfo]
   • test_connection(provider: str) -> ConnectionStatus

   Supported Providers:
   • ollama: OpenAIModel(base_url="http://localhost:11434/v1")
   • lmstudio: OpenAIModel(base_url="http://localhost:8000/v1")
   • openai: OpenAIModel(api_key=OPENAI_API_KEY)

3. Tool Registry
   Purpose: Manage available tools and their execution

   Methods:
   • register_tool(tool_func: Callable) -> str
   • get_tool(tool_name: str) -> Callable
   • list_tools() -> List[ToolSchema]
   • execute_tool(tool_name: str, **kwargs) -> Any

   Built-in Tools:
   • calculator: Perform math operations
   • web_search: Search the web (mock/API)
   • file_read: Read local files
   • file_write: Write local files
   • get_time: Get current time/date

4. Database Layer
   Purpose: Persist agents, conversations, messages

   Tables:
   • agents (id, name, system_prompt, model_id, tools, created_at)
   • conversations (id, agent_id, title, created_at)
   • messages (id, conversation_id, role, content, tool_calls, created_at)
   • provider_configs (id, provider, endpoint, api_key, active)

   Operations:
   • CRUD for all entities
   • Query conversations by agent
   • Search messages by content
   • Archive old conversations

PYDANTIC AI AGENT COMPONENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Agent Instance
   Properties:
   • model: OpenAIModel instance
   • system_prompt: str
   • tools: List[Callable] (decorated functions)

   Methods:
   • run(message: str, deps: RunContext) -> str
   • run_stream(message: str, deps: RunContext) -> AsyncIterator[str]

2. Tool Functions
   Decorator: @agent.tool
   Pattern:
       @agent.tool
       async def my_tool(ctx: RunContext[dict], param: str) -> str:
           '''Tool description for LLM'''
           # Implementation
           return result

3. RunContext
   Purpose: Pass context/dependencies to tools

   Usage:
       ctx = {"user_id": "123", "session": session_data}
       result = await agent.run(message, deps=ctx)

STREAMLIT UI COMPONENTS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Chat Interface
   Components:
   • st.chat_message("user") - User messages
   • st.chat_message("assistant") - Assistant messages
   • st.chat_input() - Message input

   State:
   • st.session_state.messages: List[dict]
   • st.session_state.current_agent: str

2. Sidebar Controls
   Components:
   • st.sidebar.selectbox("Agent", agents) - Agent selector
   • st.sidebar.selectbox("Model", models) - Model selector
   • st.sidebar.slider("Temperature", 0.0, 2.0, 0.7)
   • st.sidebar.number_input("Max Tokens", 100, 4096, 2048)
   • st.sidebar.radio("Provider", ["Ollama", "LMStudio", "OpenAI"])

   Actions:
   • st.sidebar.button("New Chat") - Clear conversation
   • st.sidebar.button("Export Chat") - Download JSON

3. API Client
   Library: httpx.AsyncClient

   Methods:
   • post_message(agent_id: str, message: str) -> AsyncIterator[str]
   • create_agent(config: dict) -> dict
   • list_agents() -> List[dict]
   • list_models() -> List[dict]


3.3 DATA FLOW
--------------------------------------------------------------------------------

CHAT MESSAGE FLOW
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. User Types Message
   ↓
2. Streamlit st.chat_input() captures input
   ↓
3. Message added to st.session_state.messages
   ↓
4. Streamlit renders user message with st.chat_message("user")
   ↓
5. Streamlit makes HTTP POST to FastAPI /api/agents/{id}/chat
   Request Body: {"message": "user message"}
   ↓
6. FastAPI receives request, validates with Pydantic model
   ↓
7. FastAPI retrieves agent from AgentFactory
   ↓
8. Agent loaded with model, tools, system prompt
   ↓
9. Pydantic AI agent.run_stream(message) called
   ↓
10. Model (Ollama/LMStudio) processes message
    ↓
11. Model decides: Need tool call? Or direct response?
    ↓
    ├─ [YES - Tool Needed]
    │  ├─ Model returns tool call with parameters
    │  ├─ Tool Registry executes tool
    │  ├─ Tool result returned to model
    │  └─ Model continues reasoning with tool result
    │
    └─ [NO - Direct Response]
       └─ Model generates text response
    ↓
12. Response streamed back to FastAPI
    ↓
13. FastAPI streams response to Streamlit
    (Server-Sent Events or chunked JSON)
    ↓
14. Streamlit displays streaming text in real-time
    placeholder.write(accumulated_text)
    ↓
15. Response complete, saved to st.session_state.messages
    ↓
16. Message persisted to database (optional, async)
    ↓
17. User sees complete response

Time: ~500ms - 5s depending on model and response length


AGENT CREATION FLOW
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. User clicks "New Agent" button
   ↓
2. Streamlit displays form with fields:
   • Name (text input)
   • System Prompt (text area)
   • Model (dropdown)
   • Tools (multiselect)
   ↓
3. User fills form and clicks "Create"
   ↓
4. Streamlit makes HTTP POST to FastAPI /api/agents
   Request Body:
   {
     "name": "My Assistant",
     "system_prompt": "You are a helpful assistant",
     "model_id": "neural-chat",
     "provider": "ollama",
     "tools": ["calculator", "web_search"],
     "temperature": 0.7,
     "max_tokens": 2048
   }
   ↓
5. FastAPI validates request with AgentConfig model
   ↓
6. AgentFactory.create_agent() called
   ↓
7. Model created via ModelFactory
   model = ModelFactory.create_model("ollama", "neural-chat")
   ↓
8. Tools loaded from ToolRegistry
   tools = [ToolRegistry.get_tool(t) for t in config.tools]
   ↓
9. Pydantic AI Agent created
   agent = Agent(model=model, system_prompt=prompt, tools=tools)
   ↓
10. Agent config saved to database
    INSERT INTO agents (...) VALUES (...)
    ↓
11. FastAPI returns agent_id and config
    Response: {"agent_id": "uuid-here", "name": "My Assistant", ...}
    ↓
12. Streamlit receives response
    ↓
13. Agent added to sidebar dropdown
    ↓
14. Agent automatically selected
    ↓
15. Success message displayed to user

Time: ~100-300ms


PROVIDER CONNECTION TEST FLOW
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. User enters provider endpoint in settings
   ↓
2. User clicks "Test Connection"
   ↓
3. Streamlit shows loading spinner
   ↓
4. Streamlit makes HTTP POST to /api/models/connect
   Request Body:
   {
     "provider": "ollama",
     "endpoint": "http://localhost:11434",
     "api_key": null
   }
   ↓
5. FastAPI attempts connection
   ↓
6. For Ollama: GET {endpoint}/api/tags
   For LMStudio: GET {endpoint}/v1/models
   ↓
7. Measure latency and parse response
   ↓
8. If successful:
   • Extract model list
   • Return ConnectionStatus(success=True, models=[...], latency=ms)
   ↓
9. If failed:
   • Capture error message
   • Return ConnectionStatus(success=False, error="...")
   ↓
10. FastAPI returns result to Streamlit
    ↓
11. Streamlit displays result:
    • Success: Green checkmark + model count + latency
    • Failure: Red X + error message
    ↓
12. If successful, provider config saved to database
    ↓
13. Model list updated in sidebar

Time: ~50-500ms depending on network




================================================================================
4. API SPECIFICATIONS
================================================================================


4.1 AGENT ENDPOINTS
--------------------------------------------------------------------------------

POST /api/agents
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Create a new agent

Request Body:
{
  "name": "string",              // Required, max 255 chars
  "system_prompt": "string",     // Required
  "model_id": "string",          // Required, e.g., "neural-chat"
  "provider": "ollama",          // Required, enum: ollama|lmstudio|openai
  "tools": ["calculator"],       // Optional, array of tool names
  "temperature": 0.7,            // Optional, 0.0-2.0, default 0.7
  "max_tokens": 2048            // Optional, 100-4096, default 2048
}

Response (201 Created):
{
  "agent_id": "uuid",
  "name": "string",
  "system_prompt": "string",
  "model_id": "string",
  "provider": "ollama",
  "tools": ["calculator"],
  "temperature": 0.7,
  "max_tokens": 2048,
  "created_at": "2026-01-22T16:32:00Z"
}

Errors:
• 400: Invalid request body
• 500: Internal server error

────────────────────────────────────────────────────────────────────────────────

POST /api/agents/{agent_id}/chat
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Send a message to an agent, receive streaming response

Path Parameters:
• agent_id: UUID of the agent

Request Body:
{
  "message": "string",           // Required, user message
  "stream": true                 // Optional, default true
}

Response (200 OK, streaming):
Content-Type: application/x-ndjson

Stream format (newline-delimited JSON):
{"type": "token", "content": "Hello", "timestamp": "2026-01-22T16:32:00Z"}
{"type": "token", "content": " there", "timestamp": "2026-01-22T16:32:00Z"}
{"type": "tool_call", "tool": "calculator", "args": {"expr": "2+2"}}
{"type": "tool_result", "tool": "calculator", "result": "4"}
{"type": "token", "content": "The answer is 4", "timestamp": "..."}
{"type": "done", "timestamp": "2026-01-22T16:32:05Z"}

Non-streaming response (stream=false):
{
  "role": "assistant",
  "content": "Hello there! The answer is 4.",
  "tool_calls": [
    {"tool": "calculator", "args": {"expr": "2+2"}, "result": "4"}
  ],
  "timestamp": "2026-01-22T16:32:05Z"
}

Errors:
• 404: Agent not found
• 500: LLM provider error

────────────────────────────────────────────────────────────────────────────────

GET /api/agents
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
List all agents

Query Parameters:
• limit: int (optional, default 100)
• offset: int (optional, default 0)

Response (200 OK):
{
  "agents": [
    {
      "agent_id": "uuid",
      "name": "My Assistant",
      "model_id": "neural-chat",
      "provider": "ollama",
      "tools": ["calculator"],
      "created_at": "2026-01-22T16:32:00Z"
    }
  ],
  "total": 1,
  "limit": 100,
  "offset": 0
}

────────────────────────────────────────────────────────────────────────────────

GET /api/agents/{agent_id}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Get agent details including configuration

Response (200 OK):
{
  "agent_id": "uuid",
  "name": "My Assistant",
  "system_prompt": "You are a helpful assistant",
  "model_id": "neural-chat",
  "provider": "ollama",
  "tools": ["calculator", "web_search"],
  "temperature": 0.7,
  "max_tokens": 2048,
  "created_at": "2026-01-22T16:32:00Z",
  "updated_at": "2026-01-22T16:32:00Z"
}

Errors:
• 404: Agent not found

────────────────────────────────────────────────────────────────────────────────

DELETE /api/agents/{agent_id}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Delete an agent

Response (200 OK):
{
  "success": true,
  "message": "Agent deleted successfully"
}

Errors:
• 404: Agent not found


4.2 MODEL ENDPOINTS
--------------------------------------------------------------------------------

GET /api/models
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
List all available models from all configured providers

Query Parameters:
• provider: string (optional, filter by provider)

Response (200 OK):
{
  "models": [
    {
      "model_id": "neural-chat",
      "provider": "ollama",
      "name": "Neural Chat 7B",
      "size": "4.1GB",
      "quantization": "Q4_0",
      "available": true
    },
    {
      "model_id": "mistral",
      "provider": "ollama",
      "name": "Mistral 7B",
      "size": "4.1GB",
      "quantization": "Q4_0",
      "available": true
    },
    {
      "model_id": "local-llama",
      "provider": "lmstudio",
      "name": "Llama 2 7B",
      "size": "3.8GB",
      "quantization": "Q4_K_M",
      "available": true
    }
  ],
  "total": 3
}

────────────────────────────────────────────────────────────────────────────────

POST /api/models/connect
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Test provider connection and retrieve available models

Request Body:
{
  "provider": "ollama",          // Required, enum
  "endpoint": "http://localhost:11434",  // Required
  "api_key": null                // Optional, for OpenAI
}

Response (200 OK):
{
  "connected": true,
  "provider": "ollama",
  "endpoint": "http://localhost:11434",
  "latency_ms": 45,
  "models": [
    {"model_id": "neural-chat", "name": "Neural Chat 7B"},
    {"model_id": "mistral", "name": "Mistral 7B"}
  ]
}

Response (200 OK, failed):
{
  "connected": false,
  "provider": "ollama",
  "endpoint": "http://localhost:11434",
  "error": "Connection refused"
}

────────────────────────────────────────────────────────────────────────────────

GET /api/models/{provider}/status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Check provider health status

Path Parameters:
• provider: string (ollama|lmstudio|openai)

Response (200 OK):
{
  "provider": "ollama",
  "available": true,
  "latency_ms": 32,
  "endpoint": "http://localhost:11434",
  "model_count": 5,
  "last_check": "2026-01-22T16:32:00Z"
}

Response (200 OK, unavailable):
{
  "provider": "ollama",
  "available": false,
  "error": "Connection timeout",
  "endpoint": "http://localhost:11434",
  "last_check": "2026-01-22T16:32:00Z"
}


4.3 TOOL ENDPOINTS
--------------------------------------------------------------------------------

GET /api/tools
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
List all available tools

Response (200 OK):
{
  "tools": [
    {
      "name": "calculator",
      "description": "Perform mathematical calculations",
      "parameters": {
        "type": "object",
        "properties": {
          "expression": {
            "type": "string",
            "description": "Math expression to evaluate"
          }
        },
        "required": ["expression"]
      }
    },
    {
      "name": "web_search",
      "description": "Search the web for information",
      "parameters": {
        "type": "object",
        "properties": {
          "query": {
            "type": "string",
            "description": "Search query"
          }
        },
        "required": ["query"]
      }
    }
  ],
  "total": 2
}

────────────────────────────────────────────────────────────────────────────────

GET /api/tools/{agent_id}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Get tools attached to a specific agent

Response (200 OK):
{
  "agent_id": "uuid",
  "tools": ["calculator", "web_search"],
  "tool_details": [
    {
      "name": "calculator",
      "description": "Perform mathematical calculations"
    }
  ]
}

────────────────────────────────────────────────────────────────────────────────

POST /api/tools/register
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Register a custom tool (advanced feature)

Request Body:
{
  "name": "my_custom_tool",
  "description": "Custom tool description",
  "parameters": {
    "type": "object",
    "properties": {
      "param1": {"type": "string"}
    }
  },
  "code": "def my_custom_tool(param1): return f'Result: {param1}'"
}

Response (201 Created):
{
  "tool_id": "uuid",
  "name": "my_custom_tool",
  "registered_at": "2026-01-22T16:32:00Z"
}

Note: This endpoint requires careful security validation in production




================================================================================
5. DATABASE DESIGN
================================================================================


5.1 SCHEMA
--------------------------------------------------------------------------------

-- Agents Table
CREATE TABLE agents (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    system_prompt TEXT NOT NULL,
    model_id TEXT NOT NULL,
    provider TEXT NOT NULL,
    tools TEXT,  -- JSON array of tool names
    temperature REAL DEFAULT 0.7,
    max_tokens INTEGER DEFAULT 2048,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_agents_name ON agents(name);
CREATE INDEX idx_agents_created ON agents(created_at DESC);

-- Conversations Table
CREATE TABLE conversations (
    id TEXT PRIMARY KEY,
    agent_id TEXT NOT NULL,
    title TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (agent_id) REFERENCES agents(id) ON DELETE CASCADE
);

CREATE INDEX idx_conversations_agent ON conversations(agent_id);
CREATE INDEX idx_conversations_created ON conversations(created_at DESC);

-- Messages Table
CREATE TABLE messages (
    id TEXT PRIMARY KEY,
    conversation_id TEXT NOT NULL,
    role TEXT NOT NULL,  -- 'user', 'assistant', 'system'
    content TEXT NOT NULL,
    tool_calls TEXT,  -- JSON array of tool calls
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (conversation_id) REFERENCES conversations(id) ON DELETE CASCADE
);

CREATE INDEX idx_messages_conversation ON messages(conversation_id);
CREATE INDEX idx_messages_created ON messages(created_at);

-- Provider Configs Table
CREATE TABLE provider_configs (
    id TEXT PRIMARY KEY,
    provider TEXT NOT NULL UNIQUE,
    endpoint TEXT NOT NULL,
    api_key TEXT,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_provider_active ON provider_configs(provider, active);

-- Tools Table
CREATE TABLE tools (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    parameters TEXT,  -- JSON schema
    code TEXT,  -- Function code (for custom tools)
    builtin BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tools_name ON tools(name);
CREATE INDEX idx_tools_builtin ON tools(builtin);


5.2 RELATIONSHIPS
--------------------------------------------------------------------------------

Entity Relationship Diagram:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌────────────────┐
│    AGENTS      │
│ ─────────────  │
│ • id (PK)      │
│ • name         │
│ • system_prompt│
│ • model_id     │
│ • provider     │
│ • tools (JSON) │
│ • temperature  │
│ • max_tokens   │
└────────┬───────┘
         │ 1
         │
         │ Many
         ▼
┌────────────────┐
│ CONVERSATIONS  │
│ ──────────────│
│ • id (PK)      │
│ • agent_id (FK)│
│ • title        │
└────────┬───────┘
         │ 1
         │
         │ Many
         ▼
┌────────────────┐
│   MESSAGES     │
│ ──────────────│
│ • id (PK)      │
│ • conv_id (FK) │
│ • role         │
│ • content      │
│ • tool_calls   │
└────────────────┘

┌────────────────┐
│PROVIDER_CONFIGS│
│ ──────────────│
│ • id (PK)      │
│ • provider     │
│ • endpoint     │
│ • api_key      │
│ • active       │
└────────────────┘

┌────────────────┐
│    TOOLS       │
│ ──────────────│
│ • id (PK)      │
│ • name (UNIQUE)│
│ • description  │
│ • parameters   │
│ • code         │
│ • builtin      │
└────────────────┘

Relationships:
• One Agent has Many Conversations (1:N)
• One Conversation has Many Messages (1:N)
• Agent deletion cascades to Conversations and Messages
• Provider Configs are standalone
• Tools are referenced by name in Agent.tools JSON array




================================================================================
6. USER INTERFACE DESIGN
================================================================================

See IMPLEMENTATION GUIDE section for detailed Streamlit code examples.

Main Layout:
• Header: App title, current agent, provider status
• Sidebar: Agent/model selectors, settings, actions
• Main Area: Chat interface with streaming messages
• Input: Chat input box with send button

Key Screens:
1. Chat Interface (main)
2. Agent Creation Dialog
3. Settings Panel




================================================================================
7. IMPLEMENTATION GUIDE
================================================================================


7.1 PYDANTIC AI SETUP
--------------------------------------------------------------------------------

from pydantic_ai import Agent, RunContext
from pydantic_ai.models import OpenAIModel

# Create model (works for Ollama, LMStudio, OpenAI)
model = OpenAIModel(
    model_id="neural-chat",
    base_url="http://localhost:11434/v1",  # Ollama endpoint
    api_key="dummy"  # Not needed for local
)

# Create agent with tools
agent = Agent(
    model=model,
    system_prompt="You are a helpful assistant with access to tools.",
    tools=[calculator_tool, web_search_tool]
)

# Define tool
@agent.tool
async def calculator_tool(ctx: RunContext[dict], expression: str) -> str:
    '''Evaluate a mathematical expression'''
    try:
        result = eval(expression)  # Use safe_eval in production!
        return f"Result: {result}"
    except Exception as e:
        return f"Error: {str(e)}"

# Run agent (non-streaming)
result = await agent.run("What is 2 + 2?")
print(result.data)

# Run agent (streaming)
async with agent.run_stream("Tell me about AI") as response:
    async for chunk in response:
        print(chunk.delta, end="", flush=True)


7.2 OLLAMA/LMSTUDIO INTEGRATION
--------------------------------------------------------------------------------

# Provider Factory Pattern
class ProviderFactory:
    @staticmethod
    def create_model(provider: str, model_id: str) -> OpenAIModel:
        configs = {
            "ollama": {
                "base_url": "http://localhost:11434/v1",
                "api_key": "dummy"
            },
            "lmstudio": {
                "base_url": "http://localhost:8000/v1",
                "api_key": "dummy"
            },
            "openai": {
                "api_key": os.getenv("OPENAI_API_KEY")
            }
        }

        config = configs.get(provider)
        if not config:
            raise ValueError(f"Unknown provider: {provider}")

        return OpenAIModel(model_id=model_id, **config)

# Usage
model = ProviderFactory.create_model("ollama", "neural-chat")
agent = Agent(model=model, system_prompt="...")

# Test connection
import httpx

async def test_ollama_connection():
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(
                "http://localhost:11434/api/tags",
                timeout=5.0
            )
            if response.status_code == 200:
                models = response.json().get("models", [])
                return {"connected": True, "models": models}
        except Exception as e:
            return {"connected": False, "error": str(e)}


7.3 STREAMLIT UI
--------------------------------------------------------------------------------

import streamlit as st
import httpx

st.title("AI Agent Chat")

# Initialize session state
if "messages" not in st.session_state:
    st.session_state.messages = []

# Sidebar
with st.sidebar:
    st.header("Settings")

    agent_id = st.selectbox("Agent", ["agent-1", "agent-2"])
    model = st.selectbox("Model", ["neural-chat", "mistral"])
    temperature = st.slider("Temperature", 0.0, 2.0, 0.7)
    provider = st.radio("Provider", ["Ollama", "LMStudio"])

    if st.button("New Chat"):
        st.session_state.messages = []

    if st.button("Export Chat"):
        # Download logic here
        pass

# Display chat history
for message in st.session_state.messages:
    with st.chat_message(message["role"]):
        st.write(message["content"])

# Chat input
if prompt := st.chat_input("Type your message..."):
    # Add user message
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.write(prompt)

    # Call API (streaming)
    with st.chat_message("assistant"):
        placeholder = st.empty()
        full_response = ""

        async with httpx.AsyncClient() as client:
            async with client.stream(
                "POST",
                f"http://localhost:8000/api/agents/{agent_id}/chat",
                json={"message": prompt},
                timeout=30.0
            ) as response:
                async for line in response.aiter_lines():
                    if line:
                        data = json.loads(line)
                        if data["type"] == "token":
                            full_response += data["content"]
                            placeholder.write(full_response)

        st.session_state.messages.append({
            "role": "assistant",
            "content": full_response
        })


7.4 FASTAPI BACKEND
--------------------------------------------------------------------------------

from fastapi import FastAPI, APIRouter
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
import json

app = FastAPI()
router = APIRouter(prefix="/api")

class ChatRequest(BaseModel):
    message: str
    stream: bool = True

class AgentConfig(BaseModel):
    name: str
    system_prompt: str
    model_id: str
    provider: str = "ollama"
    tools: list[str] = []
    temperature: float = 0.7
    max_tokens: int = 2048

# In-memory storage (use database in production)
agents = {}

@router.post("/agents")
async def create_agent(config: AgentConfig):
    agent_id = str(uuid.uuid4())

    # Create model
    model = ProviderFactory.create_model(config.provider, config.model_id)

    # Create agent
    agent = Agent(
        model=model,
        system_prompt=config.system_prompt,
        tools=load_tools(config.tools)
    )

    agents[agent_id] = {
        "agent": agent,
        "config": config
    }

    return {"agent_id": agent_id, **config.dict()}

@router.post("/agents/{agent_id}/chat")
async def chat(agent_id: str, request: ChatRequest):
    if agent_id not in agents:
        raise HTTPException(status_code=404, detail="Agent not found")

    agent = agents[agent_id]["agent"]

    async def generate():
        async with agent.run_stream(request.message) as response:
            async for chunk in response:
                yield json.dumps({
                    "type": "token",
                    "content": chunk.delta,
                    "timestamp": datetime.now().isoformat()
                }) + "\n"

        yield json.dumps({"type": "done"}) + "\n"

    return StreamingResponse(generate(), media_type="application/x-ndjson")

@router.get("/agents")
async def list_agents():
    return {
        "agents": [
            {"agent_id": aid, **agents[aid]["config"].dict()}
            for aid in agents
        ]
    }

@router.get("/models")
async def list_models():
    # Query Ollama
    async with httpx.AsyncClient() as client:
        ollama_response = await client.get("http://localhost:11434/api/tags")
        ollama_models = ollama_response.json().get("models", [])

    return {"models": ollama_models}

app.include_router(router)


7.5 CODE EXAMPLES - COMPLETE PATTERNS
--------------------------------------------------------------------------------

# Example 1: Agent with Multiple Tools
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
from pydantic_ai import Agent
from datetime import datetime

agent = Agent(model, system_prompt="You are a helpful assistant.")

@agent.tool
async def get_current_time(ctx) -> str:
    '''Get the current time'''
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

@agent.tool
async def calculate(ctx, expression: str) -> str:
    '''Calculate a math expression'''
    return str(eval(expression))

# Example 2: FastAPI with CORS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:8501"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Example 3: Database Setup with SQLAlchemy
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
from sqlalchemy import create_engine, Column, String, Text, Float
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

class AgentModel(Base):
    __tablename__ = "agents"

    id = Column(String, primary_key=True)
    name = Column(String, nullable=False)
    system_prompt = Column(Text, nullable=False)
    model_id = Column(String, nullable=False)
    provider = Column(String, nullable=False)
    temperature = Column(Float, default=0.7)

engine = create_engine("sqlite:///agents.db")
Base.metadata.create_all(engine)
SessionLocal = sessionmaker(bind=engine)




================================================================================
8. DEPLOYMENT & INFRASTRUCTURE
================================================================================


8.1 DOCKER SETUP
--------------------------------------------------------------------------------

# docker-compose.yml
version: '3.8'

services:
  fastapi:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - OLLAMA_ENDPOINT=http://ollama:11434
      - DATABASE_URL=sqlite:///data/agents.db
    volumes:
      - ./data:/app/data
    depends_on:
      - ollama
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  streamlit:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "8501:8501"
    environment:
      - FASTAPI_URL=http://fastapi:8000
    depends_on:
      - fastapi
    command: streamlit run app.py --server.port 8501

  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama

volumes:
  ollama_data:

# backend/Dockerfile
FROM python:3.10-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

# frontend/Dockerfile
FROM python:3.10-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .


8.2 DEVELOPMENT WORKFLOW
--------------------------------------------------------------------------------

# Setup
1. Install dependencies:
   pip install fastapi uvicorn pydantic-ai streamlit httpx

2. Start Ollama:
   ollama serve

3. Pull model:
   ollama pull neural-chat

4. Start FastAPI:
   cd backend && uvicorn app.main:app --reload

5. Start Streamlit:
   cd frontend && streamlit run app.py

# Testing
pytest tests/

# Deployment
docker-compose up --build




================================================================================
9. APPENDICES
================================================================================


9.1 GLOSSARY
--------------------------------------------------------------------------------

Agent: An AI entity that can reason, call tools, and interact with users
Tool: A function that an agent can call to perform specific tasks
Provider: An LLM inference service (Ollama, LMStudio, OpenAI)
Streaming: Real-time output delivery as tokens are generated
Pydantic AI: Python framework for building type-safe AI agents
OpenAI-compatible: API endpoint that follows OpenAI's format
RunContext: Context object passed to agent tools
Session State: Streamlit's mechanism for preserving state across reruns


9.2 REFERENCES
--------------------------------------------------------------------------------

• Pydantic AI: https://ai.pydantic.dev/
• FastAPI: https://fastapi.tiangolo.com/
• Streamlit: https://docs.streamlit.io/
• Ollama: https://ollama.ai/docs
• LMStudio: https://lmstudio.ai/docs
• LMStudio Python SDK: https://github.com/lmstudio-ai/lmstudio-python




================================================================================
END OF DOCUMENT
================================================================================

Generated: 2026-01-22 20:36:39
Total Sections: 9
Status: COMPLETE & READY FOR IMPLEMENTATION

================================================================================