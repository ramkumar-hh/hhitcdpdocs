# Vesper Analytics — Multi-Agent NLP Chatbot
## Technical Design Document

> **Companion Document**: This TDD covers the chatbot agent layer in full detail.
> The platform-level API, WebSocket transport, authentication, and Power BI embed flow are
> defined in `docs/README.md` §12–§17.

---

## Table of Contents

- [Part I — Overview](#part-i--overview)
  - [1. Executive Summary](#1-executive-summary)
  - [2. Supported Domains](#2-supported-domains)
  - [3. Technology Stack](#3-technology-stack)
- [Part II — Architecture](#part-ii--architecture)
  - [4. High-Level Architecture](#4-high-level-architecture)
  - [5. Component Responsibilities](#5-component-responsibilities)
- [Part III — LangGraph Design](#part-iii--langgraph-design)
  - [6. State Definition](#6-state-definition)
  - [7. Graph Topology](#7-graph-topology)
  - [8. Supervisor / Router Logic](#8-supervisor--router-logic)
- [Part IV — Domain Agents](#part-iv--domain-agents)
  - [9. DayAhead Curve Agent](#9-dayahead-curve-agent)
  - [10. Price Curve Intelligence Agent](#10-price-curve-intelligence-agent)
  - [11. PBI Usage Metrics Agent](#11-pbi-usage-metrics-agent)
  - [12. Project Execution (PER) Agent](#12-project-execution-per-agent)
- [Part V — Schema & Entity Resolution](#part-v--schema--entity-resolution)
  - [13. Schema Catalogue](#13-schema-catalogue)
  - [14. Entity Resolution Layer](#14-entity-resolution-layer)
  - [15. Text-to-SQL Strategy](#15-text-to-sql-strategy)
- [Part VI — Tools & Implementations](#part-vi--tools--implementations)
  - [16. Tool Registry](#16-tool-registry)
  - [17. SQL Tool Implementations](#17-sql-tool-implementations)
  - [18. Power BI Deep Link Tool](#18-power-bi-deep-link-tool)
- [Part VII — Response Generation](#part-vii--response-generation)
  - [19. Response Synthesizer](#19-response-synthesizer)
  - [20. Streaming to WebSocket](#20-streaming-to-websocket)
- [Part VIII — Extensibility](#part-viii--extensibility)
  - [21. Adding a New Domain](#21-adding-a-new-domain)
- [Part IX — Security & Multi-Tenancy](#part-ix--security--multi-tenancy)
  - [22. Permission Enforcement](#22-permission-enforcement)
  - [23. SQL Injection Prevention](#23-sql-injection-prevention)
- [Part X — Database Schema](#part-x--database-schema)
  - [24. Chat Session Tables](#24-chat-session-tables)
- [Appendix A — Prompt Templates](#appendix-a--prompt-templates)
- [Appendix B — Example Conversations](#appendix-b--example-conversations)

---

# Part I — Overview

## 1. Executive Summary

The Vesper Chatbot is a **multi-agent NLP system** that allows business users to query energy market data and renewable project metrics using natural language. Instead of navigating Power BI dashboards manually, users type questions like:

> *"What was the average day-ahead price for NP15 in CAISO for 2025 from Ascend?"*
> *"Show me the health status of Nazareth Solar."*
> *"Which Power BI reports were most viewed last month?"*

The system translates these queries into precise SQL operations against the underlying data warehouse views, synthesizes a business-friendly response, and optionally links to the corresponding Power BI report.

### Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Modular by domain** | Each business domain is a self-contained LangGraph node with its own tools and schema knowledge |
| **Text-to-SQL over semantic layer** | LLM generates parameterized SQL against real views — no custom semantic layer needed |
| **Entity resolution first** | Fuzzy-matched filters (vendor names, hub-zones, project names) are resolved before SQL generation |
| **Zero-trust security** | Every tool call validates tenant context and RBAC permissions before executing SQL |
| **Streamable** | Responses stream token-by-token through WebSocket using LangChain callbacks |
| **Expandable** | New domains added as LangGraph nodes + tool registration without changing core router |

---

## 2. Supported Domains

| Domain | Agent Name | Primary Views | Key Dimensions |
|--------|-----------|---------------|----------------|
| Day-Ahead Energy Curves | `dayahead_agent` | `Stage.mc_dayahead`, `Stage.mc_vendor_files` | Vendor, Market, Hub-Zone, Year, As-Of date |
| Price Curve Intelligence | `price_curve_agent` | `Stage.mc_dayahead` (aggregated) | Market, Hub-Zone, Hour, Year |
| PBI Usage Metrics | `pbi_metrics_agent` | Power BI Activity API / usage tables | Report name, user, date range |
| Project Execution (PER) | `per_agent` | `dbo.per_project`, `dbo.per_projecthealth`, `dbo.per_tasks`, `dbo.per_project45`, `dbo.activeportfolio_mm`, `dbo.changes` | Project name, stage, sponsor, technology, market |

---

## 3. Technology Stack

| Layer | Technology | Notes |
|-------|------------|-------|
| LLM | Claude `claude-sonnet-4-6` (Anthropic) | Primary reasoning and generation model |
| Orchestration | LangGraph `StateGraph` | Directed graph for multi-agent routing |
| Agent Framework | LangChain `create_react_agent` | ReAct loop for tool use within each domain agent |
| Tool Execution | LangChain `@tool` / `StructuredTool` | Type-safe tool definitions with Pydantic schemas |
| Database | Azure SQL (via pyodbc / SQLAlchemy async) | Read-only connection to data warehouse views |
| Caching | Redis | Entity resolution cache, session state |
| Transport | FastAPI WebSocket `/ws/v1/chat` | Streaming token delivery |
| Session Store | Azure SQL `chat_sessions` / `chat_messages` | Persistent conversation history |

---

# Part II — Architecture

## 4. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER (Browser)                              │
│  "What is the avg day-ahead price for NP15 in 2025 from Ascend?"   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  WebSocket /ws/v1/chat
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    FastAPI WebSocket Handler                         │
│  • JWT authentication (from ?token= query param)                    │
│  • Session management (load/create chat_session)                    │
│  • Tenant context injection                                         │
│  • Passes to ChatOrchestrator                                       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  ChatOrchestrator (LangGraph Runner)                │
│  • Loads conversation history from Redis / DB                       │
│  • Injects state: tenant_id, user_id, permissions                   │
│  • Invokes StateGraph with user message                             │
│  • Streams output tokens back via WebSocket callback                │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    LangGraph StateGraph                             │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                  SUPERVISOR NODE                              │   │
│  │  • Classifies intent using Claude + system prompt            │   │
│  │  • Resolves entities (vendor, market, hub-zone, project)     │   │
│  │  • Routes to ONE or MANY domain agents                       │   │
│  │  • Decides when to call Response Synthesizer                 │   │
│  └───────┬──────────┬──────────┬──────────┬──────────────────────┘  │
│          │          │          │          │                          │
│          ▼          ▼          ▼          ▼                          │
│  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌──────────────────────────┐  │
│  │DayAhead  │ │  Price   │ │  PBI   │ │  Project Execution (PER) │  │
│  │  Agent   │ │  Curve   │ │Metrics │ │         Agent            │  │
│  │          │ │  Agent   │ │ Agent  │ │                          │  │
│  │ Tools:   │ │ Tools:   │ │ Tools: │ │ Tools:                   │  │
│  │ avg_year │ │ hourly_  │ │ top_   │ │ project_overview         │  │
│  │ _price   │ │ compare  │ │reports │ │ project_health           │  │
│  │ list_    │ │ mkt_diff │ │ views_ │ │ milestone_status         │  │
│  │ curves   │ │          │ │ trend  │ │ budget_overview          │  │
│  │          │ │          │ │        │ │ task_risks               │  │
│  │          │ │          │ │        │ │ field_changes            │  │
│  └────┬─────┘ └────┬─────┘ └───┬────┘ └────────────┬─────────────┘  │
│       │            │           │                    │               │
│       └────────────┴───────────┴────────────────────┘               │
│                               │                                     │
│                               ▼                                     │
│                    ┌─────────────────────┐                          │
│                    │ Response Synthesizer │                          │
│                    │ • Merges agent data  │                          │
│                    │ • Formats markdown   │                          │
│                    │ • Adds PBI deep link │                          │
│                    │ • Streams tokens     │                          │
│                    └──────────┬──────────┘                          │
└───────────────────────────────┼─────────────────────────────────────┘
                                │
            ┌───────────────────┼───────────────────┐
            │                   │                   │
            ▼                   ▼                   ▼
     ┌────────────┐    ┌────────────────┐   ┌──────────────┐
     │  Azure SQL │    │     Redis      │   │  Power BI    │
     │  (Views)   │    │    (Cache)     │   │  REST API    │
     │            │    │                │   │  (deep links)│
     └────────────┘    └────────────────┘   └──────────────┘
```

---

## 5. Component Responsibilities

### 5.1 FastAPI WebSocket Handler

Located in `app/api/v1/endpoints/chat.py`. Responsibilities:

- Authenticate JWT from `?token=` query param
- Load or create `chat_session` record
- Inject `tenant_id`, `user_id`, `permissions` into `ChatOrchestrator`
- Stream LangGraph output tokens back to the browser
- Persist `chat_messages` to DB after each turn

### 5.2 ChatOrchestrator

Located in `app/agents/orchestrator.py`. Wraps the LangGraph `StateGraph` and manages:

- Session state hydration (last N messages from Redis/DB)
- Graph compilation and invocation with `astream_events`
- WebSocket callback for token streaming
- Error handling and graceful degradation

### 5.3 LangGraph StateGraph

The compiled directed graph managing execution flow:

- **Nodes**: supervisor, dayahead_agent, price_curve_agent, pbi_metrics_agent, per_agent, response_synthesizer
- **Edges**: conditional routing from supervisor to domain agents; all agents → response_synthesizer
- **State**: typed `VesperChatState` dict persisted across the graph execution

### 5.4 Domain Agents

Each domain agent is a LangChain `ReAct` agent with:
- A domain-specific system prompt describing its data and capabilities
- 2–6 structured tools backed by parameterized SQL
- Access to the shared `SqlExecutor` for tenant-safe query execution

### 5.5 SqlExecutor

Located in `app/agents/sql_executor.py`. All SQL execution routes through this class which enforces:
- Read-only connections
- Tenant-scoped filtering (RLS or explicit WHERE clauses)
- Parameterized queries (no string interpolation)
- Result-set size limits (max 500 rows)
- Query timeout (10 seconds)

---

# Part III — LangGraph Design

## 6. State Definition

```python
# app/agents/state.py
from typing import Annotated, Any, Literal, Optional
from pydantic import BaseModel, Field
from langgraph.graph.message import add_messages


class FilterContext(BaseModel):
    """Resolved business filters extracted from the user's message."""

    # Energy market filters
    vendor: Optional[str] = None           # "Ascend", "Hitachi", "ICF"
    market: Optional[str] = None           # "CAISO", "ERCOT", "MISO", "PJM"
    hub_zone: Optional[str] = None         # "NP15", "SP15", "HB_NORTH"
    asof_date: Optional[str] = None        # "2025Q1", "Jan_2025", raw ASOF string
    year: Optional[int] = None             # 2024, 2025, 2026
    hour: Optional[int] = None             # 0-23 (for hourly queries)

    # Project filters
    project_name: Optional[str] = None     # "Nazareth Solar", fuzzy-matched
    project_id: Optional[str] = None       # Internal ID after resolution
    project_stage: Optional[str] = None    # "Stage 3", "Stage 4 Pre-Con"
    technology: Optional[str] = None       # "Solar", "Wind", "BESS"
    sponsor: Optional[str] = None          # Sponsor name

    # PBI filters
    report_name: Optional[str] = None
    date_range_start: Optional[str] = None
    date_range_end: Optional[str] = None
    top_n: Optional[int] = 10


class AgentResult(BaseModel):
    """Structured result from a domain agent."""
    domain: str                                     # "dayahead", "per", "pbi", "price_curve"
    tool_calls: list[dict[str, Any]] = Field(default_factory=list)
    data: Optional[list[dict[str, Any]]] = None    # Raw SQL result rows
    summary: Optional[str] = None                  # Agent's initial summary
    error: Optional[str] = None
    pbi_report_key: Optional[str] = None           # Key for PBI deep link lookup


class VesperChatState(BaseModel):
    """
    Full state object passed between LangGraph nodes.
    All fields are optional — nodes only populate what they produce.
    """
    # Message history (LangGraph managed, append-only)
    messages: Annotated[list, add_messages]

    # Session context (injected by orchestrator, read-only in graph)
    tenant_id: str
    user_id: str
    session_id: str
    permissions: list[str]

    # Routing
    intent: Optional[Literal[
        "dayahead_curve",
        "price_curve_intelligence",
        "pbi_metrics",
        "project_execution",
        "multi_domain",
        "general"
    ]] = None
    active_agents: list[str] = Field(default_factory=list)

    # Extracted filters (populated by supervisor)
    filters: FilterContext = Field(default_factory=FilterContext)

    # Conversation memory (sliding window)
    conversation_summary: Optional[str] = None  # Compressed prior context

    # Results from domain agents
    agent_results: list[AgentResult] = Field(default_factory=list)

    # Final response
    final_response: Optional[str] = None
    pbi_deep_links: list[dict[str, str]] = Field(default_factory=list)
```

---

## 7. Graph Topology

```python
# app/agents/graph.py
from langgraph.graph import StateGraph, END
from app.agents.state import VesperChatState

def build_vesper_graph() -> StateGraph:
    graph = StateGraph(VesperChatState)

    # Register nodes
    graph.add_node("supervisor",           supervisor_node)
    graph.add_node("dayahead_agent",       dayahead_agent_node)
    graph.add_node("price_curve_agent",    price_curve_agent_node)
    graph.add_node("pbi_metrics_agent",    pbi_metrics_agent_node)
    graph.add_node("per_agent",            per_agent_node)
    graph.add_node("response_synthesizer", response_synthesizer_node)

    # Entry point
    graph.set_entry_point("supervisor")

    # Conditional routing from supervisor
    graph.add_conditional_edges(
        "supervisor",
        route_from_supervisor,   # Returns list of next nodes
        {
            "dayahead_agent":    "dayahead_agent",
            "price_curve_agent": "price_curve_agent",
            "pbi_metrics_agent": "pbi_metrics_agent",
            "per_agent":         "per_agent",
            "response_synthesizer": "response_synthesizer",
            END: END,
        }
    )

    # All domain agents flow to synthesizer
    for agent in ["dayahead_agent", "price_curve_agent", "pbi_metrics_agent", "per_agent"]:
        graph.add_edge(agent, "response_synthesizer")

    # Synthesizer terminates graph
    graph.add_edge("response_synthesizer", END)

    return graph.compile()
```

---

## 8. Supervisor / Router Logic

The supervisor is the most critical node. It performs three tasks:

1. **Intent Classification** — determines which domain(s) the query targets
2. **Entity Resolution** — extracts and normalizes named entities (vendor, market, project, etc.)
3. **Routing Decision** — dispatches to the appropriate agent(s)

### 8.1 Intent Classification Prompt

```python
SUPERVISOR_SYSTEM_PROMPT = """
You are the routing supervisor for the Vesper Energy Analytics chatbot.
Your job is to classify the user's intent and extract structured filters.

## Available Domains

### dayahead_curve
Questions about Day-Ahead energy price forecasts, average prices by year,
vendor curve comparisons (Ascend, Hitachi, ICF), hub-zone prices (NP15, SP15),
and "As-Of" file comparisons.
Example queries:
- "What is the average day-ahead price for NP15 in CAISO for 2025 from Ascend?"
- "Compare Ascend vs Hitachi for SP15 in ERCOT for 2026"
- "List available curve files for Q1 2025"

### price_curve_intelligence
Hourly price analysis, peak vs off-peak comparison, cross-market analysis,
price shape by hour-of-day, seasonal patterns.
Example queries:
- "Show me the hourly average price for CAISO NP15 in January 2025"
- "What is the peak vs off-peak spread for ERCOT HB_NORTH?"
- "Compare hourly prices between CAISO and ERCOT for 2025"

### pbi_metrics
Power BI usage statistics — report views, unique viewers, top users,
most-viewed reports, usage trends.
Example queries:
- "Which Power BI reports had the most views last month?"
- "Who are the top 10 Power BI users this quarter?"
- "Show me the usage trend for the PER dashboard"

### project_execution
Renewable energy project tracking — project health (green/yellow/red),
milestone dates, budget vs actuals, task status, risks, portfolio overview,
commercialization status. Data sourced from Asana via ETL.
Example queries:
- "What is the health status of Nazareth Solar?"
- "Show me all projects at risk"
- "What is the budget vs actuals for Nazareth Solar?"
- "List all Stage 3 projects and their COD dates"
- "What milestones changed this week?"

### general
Greetings, help requests, clarifications, or questions not mapping to a domain.

## Output Format (JSON)

Return ONLY valid JSON:
{
  "intent": "<domain_name or multi_domain or general>",
  "active_agents": ["<agent1>", "<agent2>"],
  "filters": {
    "vendor": "<normalized vendor or null>",
    "market": "<normalized market or null>",
    "hub_zone": "<normalized hub-zone or null>",
    "asof_date": "<asof string or null>",
    "year": <integer or null>,
    "hour": <integer or null>,
    "project_name": "<project name or null>",
    "project_stage": "<stage string or null>",
    "technology": "<technology or null>",
    "sponsor": "<sponsor or null>",
    "report_name": "<report name or null>",
    "date_range_start": "<ISO date or null>",
    "date_range_end": "<ISO date or null>",
    "top_n": <integer or null>
  },
  "clarification_needed": "<question to ask user if ambiguous, or null>"
}

## Entity Normalization Rules

Vendors: "ascend" → "Ascend" | "hitachi" / "icf" / "hitachi-icf" → "Hitachi"
Markets: "caiso" → "CAISO" | "ercot" → "ERCOT" | "miso" → "MISO" | "pjm" → "PJM"
Hub-Zones: "np15" → "NP15" | "sp15" → "SP15" | "hb_north" / "north hub" → "HB_NORTH"
Project names: preserve exact user text — entity resolution done by per_agent tool
"""
```

### 8.2 Supervisor Node Implementation

```python
# app/agents/supervisor.py
import json
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import SystemMessage, HumanMessage
from app.agents.state import VesperChatState, FilterContext

llm = ChatAnthropic(model="claude-sonnet-4-6", temperature=0)

async def supervisor_node(state: VesperChatState) -> dict:
    """Route the query and extract structured filters."""

    last_user_message = next(
        (m.content for m in reversed(state.messages) if m.type == "human"),
        ""
    )

    # Build context from conversation summary
    context_block = ""
    if state.conversation_summary:
        context_block = f"\n## Prior Conversation Context\n{state.conversation_summary}\n"

    response = await llm.ainvoke([
        SystemMessage(content=SUPERVISOR_SYSTEM_PROMPT + context_block),
        HumanMessage(content=last_user_message),
    ])

    try:
        routing = json.loads(response.content)
    except json.JSONDecodeError:
        # Fallback: treat as general
        routing = {"intent": "general", "active_agents": [], "filters": {}}

    # If clarification needed, short-circuit to synthesizer with a clarification
    if routing.get("clarification_needed"):
        return {
            "intent": "general",
            "active_agents": [],
            "final_response": routing["clarification_needed"],
        }

    return {
        "intent": routing.get("intent", "general"),
        "active_agents": routing.get("active_agents", []),
        "filters": FilterContext(**routing.get("filters", {})),
    }


def route_from_supervisor(state: VesperChatState) -> str:
    """
    Return the next node name.
    For multi-domain queries, LangGraph's parallel branches handle fan-out.
    Currently returning the first agent; extend with parallel branching as needed.
    """
    if state.final_response:
        return "response_synthesizer"

    agents = state.active_agents
    if not agents:
        return "response_synthesizer"

    # Route to first agent; graph handles sequential chaining
    # For true parallel: use Send() API in LangGraph
    return agents[0] if len(agents) == 1 else agents[0]
```

---

# Part IV — Domain Agents

Each domain agent follows this pattern:

```python
async def <domain>_agent_node(state: VesperChatState) -> dict:
    agent = create_react_agent(llm, tools=[...], prompt=DOMAIN_SYSTEM_PROMPT)

    # Build input including filters from supervisor
    agent_input = {
        "messages": state.messages,
        "filters": state.filters.model_dump(exclude_none=True),
        "tenant_id": state.tenant_id,
        "user_id": state.user_id,
    }

    result = await agent.ainvoke(agent_input)

    return {
        "agent_results": state.agent_results + [
            AgentResult(domain="<domain>", ...)
        ]
    }
```

---

## 9. DayAhead Curve Agent

### 9.1 Domain Overview

Queries the `Stage.mc_dayahead` and `Stage.mc_vendor_files` views to answer questions about energy price forecasts.

**Key relationships from SQL views:**
- `mc_dayahead`: fact table — `FILE_INFO_KEY`, `HUBZONE_KEY`, `DATE`, `YEAR`, `MONTH`, `HOUR`, `DAYAHEAD_PRICE`
- `mc_vendor_files`: dimension — maps `composite_key` to `vendor`, `market`, `hubzone`, `asof`, `YR`, `reference`, `SELECTION`
- Join key: `COMPOSITE_KEY` = `FILE_INFO_KEY || '_' || HUBZONE_KEY`

**Filter dimensions available:**
| Dimension | Column | Example Values |
|-----------|--------|----------------|
| Vendor | `vendor` | Ascend, Hitachi |
| Market | `market` | CAISO, ERCOT, MISO |
| Hub-Zone | `hubzone` | NP15, SP15, HB_NORTH |
| Year | `YR` | 2024, 2025, 2026 |
| As-Of | `asof` | "Jan_2025", "Q1_2025" |
| Reference | `reference` | Base, High, Low |

### 9.2 System Prompt

```python
DAYAHEAD_AGENT_PROMPT = """
You are the Day-Ahead Energy Curve specialist for Vesper Analytics.
You answer questions about energy price forecasts from vendor curve files.

## Data Available
- Average annual day-ahead prices by Vendor, Market, Hub-Zone, Year, As-Of date
- Hourly price distributions for any filter combination
- List of available curve files (vendor/market/hub-zone/asof combinations)
- Vendor comparisons (Ascend vs Hitachi for same market/hub-zone)

## Tools Available
- `get_average_annual_price`: Avg price by year for given vendor/market/hub-zone/asof
- `get_hourly_price_distribution`: Hourly avg prices (hour 0-23) for given filters
- `list_available_curves`: List available vendor+market+hub-zone+asof combinations
- `compare_vendors`: Side-by-side vendor price comparison for same market/hub-zone/year

## Instructions
1. Use the filters provided in the state to call the appropriate tool
2. If a required filter is missing, use `list_available_curves` to show options
3. Always state the data source (vendor, market, hub-zone, asof) in your response
4. Format prices as $/MWh with 2 decimal places
"""
```

### 9.3 Tools

```python
# app/agents/tools/dayahead_tools.py
from langchain_core.tools import tool
from pydantic import BaseModel, Field
from app.agents.sql_executor import SqlExecutor


class AveragePriceInput(BaseModel):
    vendor: str = Field(description="Vendor name: 'Ascend' or 'Hitachi'")
    market: str = Field(description="Market: 'CAISO', 'ERCOT', 'MISO', 'PJM'")
    hub_zone: str = Field(description="Hub-Zone: e.g. 'NP15', 'SP15', 'HB_NORTH'")
    year: int = Field(description="Forecast year, e.g. 2025")
    asof: Optional[str] = Field(None, description="As-Of identifier, e.g. 'Jan_2025'")
    tenant_id: str = Field(description="Tenant context (injected by agent)")


@tool(args_schema=AveragePriceInput)
async def get_average_annual_price(
    vendor: str, market: str, hub_zone: str, year: int,
    asof: Optional[str], tenant_id: str
) -> dict:
    """
    Returns the average day-ahead price ($/MWh) for a given vendor, market,
    hub-zone, and forecast year. Optionally filtered by As-Of date.
    """
    sql = """
    SELECT
        vf.vendor                           AS Vendor,
        vf.market                           AS Market,
        vf.hubzone                          AS HubZone,
        vf.asof                             AS AsOf,
        da.YEAR                             AS ForecastYear,
        ROUND(AVG(da.DAYAHEAD_PRICE), 2)    AS AvgPrice_PerMWh,
        COUNT(*)                            AS DataPoints
    FROM Stage.mc_dayahead da
    JOIN Stage.mc_vendor_files vf
        ON da.FILE_INFO_KEY = vf.file_info_key
        AND da.HUBZONE_KEY  = vf.hubzone_key
    WHERE
        vf.vendor  = :vendor
        AND vf.market  = :market
        AND vf.hubzone = :hub_zone
        AND da.YEAR    = :year
        {asof_clause}
    GROUP BY vf.vendor, vf.market, vf.hubzone, vf.asof, da.YEAR
    ORDER BY vf.asof
    """
    asof_clause = "AND vf.asof = :asof" if asof else ""

    params = {
        "vendor": vendor, "market": market,
        "hub_zone": hub_zone, "year": year
    }
    if asof:
        params["asof"] = asof

    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql.format(asof_clause=asof_clause), params)

    return {
        "data": rows,
        "pbi_report_key": "dayahead_curve_report",
        "filters_used": {"vendor": vendor, "market": market, "hub_zone": hub_zone, "year": year}
    }


class ListCurvesInput(BaseModel):
    vendor: Optional[str] = Field(None, description="Filter by vendor")
    market: Optional[str] = Field(None, description="Filter by market")
    hub_zone: Optional[str] = Field(None, description="Filter by hub-zone")
    tenant_id: str


@tool(args_schema=ListCurvesInput)
async def list_available_curves(
    vendor: Optional[str], market: Optional[str],
    hub_zone: Optional[str], tenant_id: str
) -> dict:
    """
    Lists available vendor curve files. Use when user wants to know what
    combinations are available or when required filters are missing.
    """
    conditions = []
    params = {}

    if vendor:
        conditions.append("vf.vendor = :vendor")
        params["vendor"] = vendor
    if market:
        conditions.append("vf.market = :market")
        params["market"] = market
    if hub_zone:
        conditions.append("vf.hubzone = :hub_zone")
        params["hub_zone"] = hub_zone

    where = "WHERE " + " AND ".join(conditions) if conditions else ""

    sql = f"""
    SELECT DISTINCT
        vf.vendor, vf.market, vf.hubzone, vf.asof, vf.YR, vf.reference,
        vf.SELECTION
    FROM Stage.mc_vendor_files vf
    {where}
    ORDER BY vf.vendor, vf.market, vf.hubzone, vf.asof
    """

    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, params)
    return {"data": rows}


class CompareVendorsInput(BaseModel):
    market: str
    hub_zone: str
    year: int
    vendors: list[str] = Field(default=["Ascend", "Hitachi"],
                                description="Vendors to compare")
    tenant_id: str


@tool(args_schema=CompareVendorsInput)
async def compare_vendors(
    market: str, hub_zone: str, year: int,
    vendors: list[str], tenant_id: str
) -> dict:
    """
    Side-by-side average price comparison between vendors for the same
    market, hub-zone, and year. Returns pivot-style result.
    """
    placeholders = ",".join([f":v{i}" for i in range(len(vendors))])
    params = {"market": market, "hub_zone": hub_zone, "year": year}
    for i, v in enumerate(vendors):
        params[f"v{i}"] = v

    sql = f"""
    SELECT
        da.YEAR             AS ForecastYear,
        vf.vendor           AS Vendor,
        vf.asof             AS AsOf,
        ROUND(AVG(da.DAYAHEAD_PRICE), 2) AS AvgPrice_PerMWh
    FROM Stage.mc_dayahead da
    JOIN Stage.mc_vendor_files vf
        ON da.FILE_INFO_KEY = vf.file_info_key
        AND da.HUBZONE_KEY  = vf.hubzone_key
    WHERE vf.market   = :market
      AND vf.hubzone  = :hub_zone
      AND da.YEAR     = :year
      AND vf.vendor   IN ({placeholders})
    GROUP BY da.YEAR, vf.vendor, vf.asof
    ORDER BY vf.vendor, vf.asof
    """

    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, params)
    return {"data": rows, "pbi_report_key": "dayahead_curve_report"}
```

---

## 10. Price Curve Intelligence Agent

### 10.1 Domain Overview

Provides deeper analytical views of hourly price data: peak vs off-peak analysis, cross-market comparison, seasonal price shapes.

### 10.2 Tools

```python
# app/agents/tools/price_curve_tools.py

class HourlyPriceInput(BaseModel):
    market: str
    hub_zone: str
    year: int
    month: Optional[int] = Field(None, description="1-12, or None for full year")
    vendor: Optional[str] = None
    tenant_id: str


@tool(args_schema=HourlyPriceInput)
async def get_hourly_price_distribution(
    market: str, hub_zone: str, year: int,
    month: Optional[int], vendor: Optional[str], tenant_id: str
) -> dict:
    """
    Returns average price by hour-of-day (0-23) for a given market/hub-zone/year.
    Useful for peak vs off-peak analysis.
    """
    month_clause = "AND da.MONTH = :month" if month else ""
    vendor_clause = "AND vf.vendor = :vendor" if vendor else ""

    params = {"market": market, "hub_zone": hub_zone, "year": year}
    if month: params["month"] = month
    if vendor: params["vendor"] = vendor

    sql = f"""
    SELECT
        da.HOUR,
        ROUND(AVG(da.DAYAHEAD_PRICE), 2) AS AvgPrice_PerMWh,
        ROUND(MIN(da.DAYAHEAD_PRICE), 2) AS MinPrice,
        ROUND(MAX(da.DAYAHEAD_PRICE), 2) AS MaxPrice
    FROM Stage.mc_dayahead da
    JOIN Stage.mc_vendor_files vf
        ON da.FILE_INFO_KEY = vf.file_info_key
        AND da.HUBZONE_KEY  = vf.hubzone_key
    WHERE vf.market  = :market
      AND vf.hubzone = :hub_zone
      AND da.YEAR    = :year
      {month_clause}
      {vendor_clause}
    GROUP BY da.HOUR
    ORDER BY da.HOUR
    """

    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, params)
    return {
        "data": rows,
        "pbi_report_key": "price_curve_hourly_report",
        "analysis_hint": "hours 7-22 are typically peak, 0-6 and 23 off-peak"
    }


class MarketDiffInput(BaseModel):
    hub_zones: list[str] = Field(description="Two hub-zones to compare, e.g. ['NP15','SP15']")
    year: int
    vendor: Optional[str] = None
    tenant_id: str


@tool(args_schema=MarketDiffInput)
async def compare_hub_zones(
    hub_zones: list[str], year: int,
    vendor: Optional[str], tenant_id: str
) -> dict:
    """
    Compares average day-ahead prices between two hub-zones for the same year.
    Shows the spread (differential) by hour.
    """
    if len(hub_zones) != 2:
        return {"error": "Exactly 2 hub-zones required for comparison"}

    params = {
        "hub_zone_a": hub_zones[0],
        "hub_zone_b": hub_zones[1],
        "year": year
    }
    vendor_clause = "AND vf.vendor = :vendor" if vendor else ""
    if vendor: params["vendor"] = vendor

    sql = f"""
    WITH hourly AS (
        SELECT
            vf.hubzone,
            da.HOUR,
            ROUND(AVG(da.DAYAHEAD_PRICE), 2) AS AvgPrice
        FROM Stage.mc_dayahead da
        JOIN Stage.mc_vendor_files vf
            ON da.FILE_INFO_KEY = vf.file_info_key
            AND da.HUBZONE_KEY  = vf.hubzone_key
        WHERE vf.hubzone IN (:hub_zone_a, :hub_zone_b)
          AND da.YEAR = :year
          {vendor_clause}
        GROUP BY vf.hubzone, da.HOUR
    )
    SELECT
        a.HOUR,
        a.AvgPrice AS [{hub_zones[0]}],
        b.AvgPrice AS [{hub_zones[1]}],
        ROUND(a.AvgPrice - b.AvgPrice, 2) AS Spread
    FROM hourly a
    JOIN hourly b ON a.HOUR = b.HOUR
    WHERE a.hubzone = :hub_zone_a
      AND b.hubzone = :hub_zone_b
    ORDER BY a.HOUR
    """

    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, params)
    return {"data": rows, "pbi_report_key": "price_curve_comparison_report"}
```

---

## 11. PBI Usage Metrics Agent

### 11.1 Domain Overview

Power BI usage data is collected either from the Power BI Activity Events API (Microsoft) or from a custom `pbi_usage` staging table populated via the Azure Monitor / PBI Admin API pipeline.

### 11.2 Schema Assumption

```sql
-- Stage table populated by PBI Activity API ingestion job
CREATE TABLE stage.pbi_activity_events (
    event_id        BIGINT IDENTITY PRIMARY KEY,
    tenant_id       NVARCHAR(100),
    report_id       NVARCHAR(200),
    report_name     NVARCHAR(500),
    workspace_name  NVARCHAR(500),
    user_email      NVARCHAR(500),
    activity_type   NVARCHAR(100),  -- 'ViewReport', 'ExportReport', etc.
    activity_date   DATE,
    created_at      DATETIME2 DEFAULT GETUTCDATE()
);
```

### 11.3 Tools

```python
# app/agents/tools/pbi_tools.py

class TopReportsInput(BaseModel):
    date_range_start: str = Field(description="ISO date, e.g. '2025-01-01'")
    date_range_end: str = Field(description="ISO date, e.g. '2025-01-31'")
    top_n: int = Field(default=10, description="How many top reports to return")
    tenant_id: str


@tool(args_schema=TopReportsInput)
async def get_top_reports_by_views(
    date_range_start: str, date_range_end: str,
    top_n: int, tenant_id: str
) -> dict:
    """Returns the most-viewed Power BI reports in a given date range."""
    sql = """
    SELECT TOP (:top_n)
        report_name,
        workspace_name,
        COUNT(*)                    AS TotalViews,
        COUNT(DISTINCT user_email)  AS UniqueViewers
    FROM stage.pbi_activity_events
    WHERE tenant_id      = :tenant_id
      AND activity_type  = 'ViewReport'
      AND activity_date BETWEEN :start_date AND :end_date
    GROUP BY report_name, workspace_name
    ORDER BY TotalViews DESC
    """
    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, {
        "top_n": top_n, "tenant_id": tenant_id,
        "start_date": date_range_start, "end_date": date_range_end
    })
    return {"data": rows, "pbi_report_key": "pbi_usage_metrics_report"}


class TopUsersInput(BaseModel):
    date_range_start: str
    date_range_end: str
    top_n: int = 10
    report_name: Optional[str] = Field(None, description="Filter to a specific report")
    tenant_id: str


@tool(args_schema=TopUsersInput)
async def get_top_users(
    date_range_start: str, date_range_end: str,
    top_n: int, report_name: Optional[str], tenant_id: str
) -> dict:
    """Returns the most active Power BI users in a date range."""
    report_clause = "AND report_name = :report_name" if report_name else ""
    params = {
        "top_n": top_n, "tenant_id": tenant_id,
        "start_date": date_range_start, "end_date": date_range_end
    }
    if report_name: params["report_name"] = report_name

    sql = f"""
    SELECT TOP (:top_n)
        user_email,
        COUNT(*)                    AS TotalViews,
        COUNT(DISTINCT report_name) AS UniqueReportsViewed,
        MAX(activity_date)          AS LastActivity
    FROM stage.pbi_activity_events
    WHERE tenant_id     = :tenant_id
      AND activity_type = 'ViewReport'
      AND activity_date BETWEEN :start_date AND :end_date
      {report_clause}
    GROUP BY user_email
    ORDER BY TotalViews DESC
    """
    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, params)
    return {"data": rows}
```

---

## 12. Project Execution (PER) Agent

### 12.1 Domain Overview

This is the richest domain, backed by six SQL views. Data originates from **Asana** (project management) via a daily ETL pipeline that populates both current-state and snapshot tables.

**View Summary:**

| View | Purpose | Key Columns |
|------|---------|-------------|
| `dbo.per_project` | Current project snapshot with budget & commercial data | name, currentprojectstage, spa, sponsor, technology, gencap, storecap, market, Actuals_LTD, Total_Operating_Budget, buyer, price |
| `dbo.per_projecthealth` | Health scores per domain (site, permits, interconnection, etc.) | section, healthcolor (green/yellow/red), notes, due_on, pctcomplete |
| `dbo.per_tasks` | Task list with status and risk | projectname, taskname, task_type, finishdate, status, risk, notesforper |
| `dbo.per_project45` | Budget detail by group/subgroup/metric | group, subgrouping, metric, value_numbers, value_dates |
| `dbo.activeportfolio_mm` | Monthly snapshot portfolio with all milestone dates | Stage 1–8 dates, COD, Decisioning Date, Project Status Color, Commercialization Status |
| `dbo.changes` | Audit trail of field changes | dateofchange, fieldchanged, previousvalue, newvalue |

### 12.2 System Prompt

```python
PER_AGENT_PROMPT = """
You are the Project Execution (PER) specialist for Vesper Analytics.
You answer questions about renewable energy projects tracked in Asana.

## Data Available
- Project overview: current stage, technology, market, sponsor, SPA, location,
  budget vs actuals, commercial execution (buyer, tenor, PPA price, revision date)
- Project health: health color (green/yellow/red) per domain (site, permits,
  interconnection, engineering, procurement, offtake, financing, construction, schedule)
- Task status and risks for any project
- Portfolio-level milestone timeline (Stage 1-8, COD dates)
- Historical changes to project fields and task dates

## Tools Available
- `get_project_overview`: High-level project summary
- `get_project_health`: Health status per domain
- `get_milestone_dates`: Stage milestone dates and COD
- `get_budget_overview`: Budget, actuals, forecast, IC_LC, PPA_LC
- `get_task_risks`: Tasks with risk or status flags
- `get_portfolio_summary`: Portfolio-level view (all active projects)
- `get_recent_changes`: Recent field changes for a project or across portfolio

## Instructions
1. When user asks about a specific project, use `get_project_overview` first
2. Map health colors: green = on track, yellow = at risk, red = off track, blue = on hold
3. Format financial figures as US dollars with appropriate units ($K or $M)
4. When discussing COD (Commercial Operation Date), always state the year
5. If the project name is ambiguous, list matching projects and ask for clarification
"""
```

### 12.3 Tools

```python
# app/agents/tools/per_tools.py
from langchain_core.tools import tool
from app.agents.sql_executor import SqlExecutor


class ProjectInput(BaseModel):
    project_name: str = Field(description="Project name or partial name, e.g. 'Nazareth Solar'")
    tenant_id: str


@tool(args_schema=ProjectInput)
async def get_project_overview(project_name: str, tenant_id: str) -> dict:
    """
    Returns a complete overview of a project including current stage, technology,
    market, sponsor, budget, and commercial execution data.
    Uses fuzzy LIKE matching on project name.
    """
    sql = """
    SELECT TOP 1
        p.name                      AS ProjectName,
        p.project_id                AS ProjectID,
        p.currentprojectstage       AS CurrentStage,
        p.technology                AS Technology,
        p.market                    AS Market,
        p.location                  AS Location,
        p.spa                       AS SPA,
        p.sponsor                   AS Sponsor,
        p.gencap                    AS GenerationCapacity_MW,
        p.storecap                  AS StorageCapacity_MW,
        p.utility                   AS Utility,
        p.Status_Type               AS ProjectStatus,
        p.summary                   AS StatusSummary,
        -- Budget
        p.Actuals_LTD               AS ActualsLTD,
        p.Total_Operating_Budget    AS TotalBudget,
        p.Total_Operating_Forecast  AS TotalForecast,
        p.IC_LC,
        p.PPA_LC,
        -- Commercial execution
        p.buyer                     AS Buyer,
        p.tenor                     AS Tenor,
        p.products                  AS Products,
        p.price                     AS PPAPrice,
        p.revision_date             AS RevisionDate
    FROM dbo.per_project p
    WHERE p.name LIKE :project_pattern
    ORDER BY LEN(p.name)  -- prefer shortest (most specific) match
    """
    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, {"project_pattern": f"%{project_name}%"})

    if not rows:
        # Try to list similar projects
        list_sql = """
        SELECT DISTINCT name AS ProjectName, currentprojectstage, technology, market
        FROM dbo.per_project
        WHERE name LIKE :broad_pattern
        ORDER BY name
        """
        words = project_name.split()
        broad = f"%{words[0]}%"
        suggestions = await executor.execute(list_sql, {"broad_pattern": broad})
        return {
            "found": False,
            "suggestions": suggestions,
            "message": f"No exact match for '{project_name}'. Did you mean one of these?"
        }

    return {"found": True, "data": rows, "pbi_report_key": "per_project_report"}


@tool(args_schema=ProjectInput)
async def get_project_health(project_name: str, tenant_id: str) -> dict:
    """
    Returns the health color (green/yellow/red) for each domain of a project
    (Site, Permits, Interconnection, Engineering, Procurement, Offtake,
    Financing, Construction, Schedule) with notes and completion %.
    """
    sql = """
    SELECT
        h.Section,
        h.name          AS HealthArea,
        h.healthcolor   AS HealthColor,
        h.notes         AS Notes,
        h.due_on        AS DueDate,
        h.pctcomplete   AS PercentComplete,
        h.sort          AS SortOrder
    FROM dbo.per_projecthealth h
    JOIN dbo.per_project p ON p.gid = h.project_gid
    WHERE p.name LIKE :project_pattern
    ORDER BY h.sort
    """
    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, {"project_pattern": f"%{project_name}%"})
    return {"data": rows, "pbi_report_key": "per_project_health_report"}


@tool(args_schema=ProjectInput)
async def get_milestone_dates(project_name: str, tenant_id: str) -> dict:
    """
    Returns all milestone dates for a project: Stage 1 through Stage 8,
    COD (Commercial Operation Date), Decisioning Date, and key agreement dates.
    """
    sql = """
    SELECT TOP 1
        [Project Name]              AS ProjectName,
        [Current Project Stage]     AS CurrentStage,
        [Project Status Color]      AS StatusColor,
        [CommercializationStatus]   AS CommercializationStatus,
        [Stage 1 Early Dev],
        [Stage 2 Mid Dev],
        [Stage 3 Late Dev],
        [Stage 4 Pre-Con & Finance],
        [Stage 5 Construction],
        [Stage 6 Operation],
        [Stage 7 Sale],
        [Stage 8 Closing],
        [Minimum Site Control],
        [Substantial Completion],
        [Mechanical Completion],
        [Project Finance],
        [EPC FNTP],
        [Execute Power Purchase Agreement],
        [Execute Interconnection Agreement],
        [Commercial Operation]      AS COD,
        [Decisioning Date],
        [COD Year]
    FROM dbo.activeportfolio_mm
    WHERE [Project Name] LIKE :project_pattern
      AND monthlysnapshot = 1       -- latest monthly snapshot
    ORDER BY asofday DESC
    """
    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, {"project_pattern": f"%{project_name}%"})
    return {"data": rows, "pbi_report_key": "per_portfolio_report"}


class BudgetInput(BaseModel):
    project_name: str
    tenant_id: str


@tool(args_schema=BudgetInput)
async def get_budget_overview(project_name: str, tenant_id: str) -> dict:
    """
    Returns budget detail broken down by group (Capital Budget, Capital At Risk,
    Project Return, Schedule) and subgrouping/metric.
    """
    sql = """
    SELECT
        p45.[group]         AS BudgetGroup,
        p45.subgrouping     AS SubGroup,
        p45.metric          AS Metric,
        p45.value_numbers   AS ValueAmount,
        p45.value_dates     AS ValueDate
    FROM dbo.per_project45 p45
    JOIN dbo.per_project pp ON pp.project_id = p45.project_id
    WHERE pp.name LIKE :project_pattern
      AND p45.MonthFromFileName IS NULL  -- current (not archived)
    ORDER BY p45.[group], p45.subgrouping, p45.metric
    """
    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, {"project_pattern": f"%{project_name}%"})
    return {"data": rows, "pbi_report_key": "per_budget_report"}


class RiskInput(BaseModel):
    project_name: Optional[str] = None
    risk_level: Optional[str] = Field(None, description="'High', 'Medium', 'Low'")
    tenant_id: str


@tool(args_schema=RiskInput)
async def get_task_risks(
    project_name: Optional[str], risk_level: Optional[str], tenant_id: str
) -> dict:
    """
    Returns tasks flagged with risk or non-green status.
    Can filter by project name and/or risk level.
    """
    conditions = ["(t.risk IS NOT NULL AND t.risk <> '')"]
    params = {}

    if project_name:
        conditions.append("t.projectname LIKE :project_pattern")
        params["project_pattern"] = f"%{project_name}%"
    if risk_level:
        conditions.append("t.risk = :risk_level")
        params["risk_level"] = risk_level

    sql = f"""
    SELECT
        t.projectname   AS ProjectName,
        t.taskname      AS TaskName,
        t.task_type     AS TaskType,
        t.finishdate    AS DueDate,
        t.status        AS Status,
        t.risk          AS RiskLevel,
        t.notesforper   AS Notes
    FROM dbo.per_tasks t
    WHERE {' AND '.join(conditions)}
    ORDER BY t.projectname, t.risk DESC, t.finishdate
    """
    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, params)
    return {"data": rows, "pbi_report_key": "per_project_report"}


class PortfolioInput(BaseModel):
    technology: Optional[str] = None
    stage: Optional[str] = None
    status_color: Optional[str] = Field(None, description="'green', 'yellow', 'red', 'blue'")
    tenant_id: str


@tool(args_schema=PortfolioInput)
async def get_portfolio_summary(
    technology: Optional[str], stage: Optional[str],
    status_color: Optional[str], tenant_id: str
) -> dict:
    """
    Returns a portfolio-level summary of all active projects.
    Filterable by technology, stage, and project status color.
    """
    conditions = ["p.is_archived = 0 OR p.is_archived IS NULL"]
    params = {}

    if technology:
        conditions.append("pp.technology = :technology")
        params["technology"] = technology
    if stage:
        conditions.append("pp.currentprojectstage LIKE :stage")
        params["stage"] = f"%{stage}%"
    if status_color:
        conditions.append("pp.Status_Type = :status_color")
        params["status_color"] = status_color

    sql = f"""
    SELECT
        pp.name                     AS ProjectName,
        pp.project_id               AS ProjectID,
        pp.currentprojectstage      AS CurrentStage,
        pp.technology               AS Technology,
        pp.market                   AS Market,
        pp.sponsor                  AS Sponsor,
        pp.Status_Type              AS StatusColor,
        pp.gencap                   AS GenCapacity_MW,
        pp.storecap                 AS StorCapacity_MW,
        pp.Actuals_LTD              AS ActualsLTD,
        pp.Total_Operating_Budget   AS TotalBudget,
        pp.summary                  AS StatusSummary
    FROM dbo.per_project pp
    WHERE {' AND '.join(conditions)}
    ORDER BY pp.Status_Type, pp.currentprojectstage, pp.name
    """
    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, params)
    return {"data": rows, "pbi_report_key": "per_portfolio_report"}


class ChangesInput(BaseModel):
    project_name: Optional[str] = None
    field_changed: Optional[str] = Field(None,
        description="'FinishDate', 'Status', 'Risk Assessment', 'Budget', 'Forecast', etc.")
    since_date: str = Field(description="ISO date, e.g. '2025-01-01'")
    tenant_id: str


@tool(args_schema=ChangesInput)
async def get_recent_changes(
    project_name: Optional[str], field_changed: Optional[str],
    since_date: str, tenant_id: str
) -> dict:
    """
    Returns the audit trail of recent changes to project fields, task dates,
    status, risk, and budget. Useful for 'what changed this week?' queries.
    """
    conditions = ["dateofchange >= :since_date"]
    params = {"since_date": since_date}

    if project_name:
        conditions.append("projectname LIKE :project_pattern")
        params["project_pattern"] = f"%{project_name}%"
    if field_changed:
        conditions.append("fieldchanged = :field_changed")
        params["field_changed"] = field_changed

    sql = f"""
    SELECT TOP 100
        dateofchange    AS ChangeDate,
        projectname     AS ProjectName,
        taskname        AS TaskName,
        fieldchanged    AS FieldChanged,
        previousvalue   AS PreviousValue,
        newvalue        AS NewValue
    FROM dbo.changes
    WHERE {' AND '.join(conditions)}
    ORDER BY dateofchange DESC
    """
    executor = SqlExecutor(tenant_id=tenant_id)
    rows = await executor.execute(sql, params)
    return {"data": rows}
```

---

# Part V — Schema & Entity Resolution

## 13. Schema Catalogue

The agent system uses a **Schema Catalogue** — a structured in-memory description of each domain's SQL views that is included in tool docstrings and agent system prompts. This eliminates the need for a separate semantic layer.

```python
# app/agents/schema_catalogue.py

SCHEMA_CATALOGUE = {
    "dayahead": {
        "primary_views": ["Stage.mc_dayahead", "Stage.mc_vendor_files"],
        "join_key": "COMPOSITE_KEY (FILE_INFO_KEY + '_' + HUBZONE_KEY)",
        "dimensions": {
            "vendor":   {"column": "vf.vendor",   "values": ["Ascend", "Hitachi"]},
            "market":   {"column": "vf.market",   "values": ["CAISO", "ERCOT", "MISO", "PJM", "NYISO", "ISO-NE"]},
            "hub_zone": {"column": "vf.hubzone",  "values": ["NP15", "SP15", "ZP26", "HB_NORTH", "HB_HOUSTON", "HB_WEST"]},
            "asof":     {"column": "vf.asof",     "format": "MMM_YYYY or YYYY_QN"},
            "year":     {"column": "da.YEAR",     "type": "integer"},
        },
        "measures": {
            "DAYAHEAD_PRICE": {"unit": "$/MWh", "agg": "AVG"}
        }
    },

    "per": {
        "primary_views": [
            "dbo.per_project", "dbo.per_projecthealth", "dbo.per_tasks",
            "dbo.per_project45", "dbo.activeportfolio_mm", "dbo.changes"
        ],
        "join_key": "project_gid (Asana GID) or project_id (internal ID)",
        "dimensions": {
            "project_name": {"column": "p.name", "match": "LIKE fuzzy"},
            "technology":   {"column": "pp.technology", "values": ["Solar", "Wind", "BESS", "Hybrid"]},
            "stage":        {"column": "pp.currentprojectstage", "values": ["Stage 1", "Stage 2", "Stage 3", "Stage 4", "Stage 5", "Stage 6"]},
            "status_type":  {"column": "pp.Status_Type", "values": ["on_track", "at_risk", "off_track", "on_hold"]},
            "healthcolor":  {"column": "h.healthcolor", "values": ["green", "yellow", "red"]},
        },
        "health_domains": ["site", "permits", "interconnection", "engineering",
                           "procurement", "offtake", "financing", "construction", "schedule"],
        "change_fields": ["FinishDate", "Status", "Risk Assessment", "NotesForPER",
                          "Duration(Hrs)", "GenerationCapacity", "StorageCapacity",
                          "TotalCapacity", "HuborUtility", "Location", "Technology",
                          "ProjectStatus", "Spend", "Budget", "Forecast", "IC_LC", "PPA_LC"]
    }
}
```

---

## 14. Entity Resolution Layer

Before SQL generation, the supervisor resolves ambiguous user-provided entity names against known values using fuzzy matching backed by Redis cache.

```python
# app/agents/entity_resolver.py
import redis.asyncio as aioredis
from difflib import get_close_matches
from typing import Optional

KNOWN_ENTITIES = {
    "vendor":   ["Ascend", "Hitachi"],
    "market":   ["CAISO", "ERCOT", "MISO", "PJM", "NYISO", "ISO-NE", "SPP"],
    "hub_zone": ["NP15", "SP15", "ZP26", "HB_NORTH", "HB_HOUSTON", "HB_WEST",
                 "HB_BUSAVG", "LZ_NORTH", "LZ_HOUSTON", "LZ_AEN", "LZ_CPS",
                 "LZ_RAYBN", "LZ_WEST"],
    "technology": ["Solar", "Wind", "BESS", "Hybrid", "Solar+Storage"],
    "project_status": ["on_track", "at_risk", "off_track", "on_hold"],
}

ALIAS_MAP = {
    # Vendors
    "hitachi icf": "Hitachi", "icf": "Hitachi", "hitachi": "Hitachi",
    "ascend": "Ascend",
    # Markets
    "california": "CAISO", "texas": "ERCOT", "midwest": "MISO",
    "mid-atlantic": "PJM", "new york": "NYISO", "new england": "ISO-NE",
    # Hub-zones
    "north": "NP15", "south": "SP15", "north hub": "HB_NORTH",
    "houston hub": "HB_HOUSTON", "west hub": "HB_WEST",
    # Status
    "green": "on_track", "yellow": "at_risk", "red": "off_track",
    "blue": "on_hold", "on hold": "on_hold", "at risk": "at_risk",
}


class EntityResolver:
    def __init__(self, redis_client: aioredis.Redis):
        self.redis = redis_client

    def resolve(self, entity_type: str, value: Optional[str]) -> Optional[str]:
        if not value:
            return None

        lowered = value.strip().lower()

        # Check alias map first
        if lowered in ALIAS_MAP:
            return ALIAS_MAP[lowered]

        # Fuzzy match against known values
        known = KNOWN_ENTITIES.get(entity_type, [])
        matches = get_close_matches(lowered, [k.lower() for k in known], n=1, cutoff=0.7)

        if matches:
            # Return the original-cased known value
            idx = [k.lower() for k in known].index(matches[0])
            return known[idx]

        return value  # Return as-is if no match (let SQL handle it)

    async def resolve_project_name(
        self, partial_name: str, tenant_id: str, executor
    ) -> dict:
        """Resolve a partial project name against live DB data."""
        cache_key = f"entity:project:{tenant_id}:{partial_name.lower()}"
        cached = await self.redis.get(cache_key)
        if cached:
            return {"resolved": cached.decode(), "from_cache": True}

        sql = """
        SELECT TOP 5 name, project_id
        FROM dbo.per_project
        WHERE name LIKE :pattern
        ORDER BY LEN(name)
        """
        rows = await executor.execute(sql, {"pattern": f"%{partial_name}%"})

        if len(rows) == 1:
            await self.redis.setex(cache_key, 300, rows[0]["name"])
            return {"resolved": rows[0]["name"], "exact": True}
        elif len(rows) > 1:
            return {"ambiguous": True, "matches": [r["name"] for r in rows]}
        else:
            return {"not_found": True}
```

---

## 15. Text-to-SQL Strategy

The system uses a **Constrained Text-to-SQL** approach:

```
User Query
    ↓
Supervisor (Claude) → Extracts structured filters (not raw SQL)
    ↓
Domain Agent Tools (Python) → Assembles parameterized SQL from filter inputs
    ↓
SqlExecutor → Executes with positional parameters
    ↓
Result rows → Domain Agent formats → Response Synthesizer
```

**Why not free-form Text-to-SQL?**

| Approach | Pros | Cons |
|----------|------|------|
| Free-form Text-to-SQL | Flexible | SQL injection risk, schema hallucination, hard to validate |
| Structured filter → parameterized SQL | Safe, predictable, auditable | Requires tool per query type |
| **Vesper approach: Structured filters** | Safe + LLM validates filters before parameterized SQL | Slight rigidity — mitigated by `general_sql_query` escape hatch |

**Escape hatch — supervised ad-hoc SQL:**

```python
@tool
async def general_sql_query(
    description: str,
    sql: str,
    tenant_id: str
) -> dict:
    """
    [RESTRICTED — requires 'chat:advanced_sql' permission]
    Execute a custom SELECT query against approved views.
    The SQL must only reference views in the approved list.
    """
    APPROVED_VIEWS = {
        "Stage.mc_dayahead", "Stage.mc_vendor_files",
        "dbo.per_project", "dbo.per_projecthealth", "dbo.per_tasks",
        "dbo.per_project45", "dbo.activeportfolio_mm", "dbo.changes",
        "stage.pbi_activity_events",
    }

    # Validate SQL only touches approved views
    sql_upper = sql.upper()
    for forbidden in ["INSERT", "UPDATE", "DELETE", "DROP", "EXEC", "EXECUTE",
                       "sp_", "xp_", "--", "/*", "OPENROWSET", "BULK"]:
        if forbidden in sql_upper:
            return {"error": f"Forbidden keyword: {forbidden}"}

    executor = SqlExecutor(tenant_id=tenant_id, read_only=True)
    rows = await executor.execute(sql, {})
    return {"data": rows}
```

---

# Part VI — Tools & Implementations

## 16. Tool Registry

Each domain agent receives only the tools relevant to its domain. The tool registry makes this explicit and supports per-tenant enabling/disabling.

```python
# app/agents/tool_registry.py
from typing import dict

TOOL_REGISTRY: dict[str, list] = {
    "dayahead_agent": [
        get_average_annual_price,
        get_hourly_price_distribution,  # shared with price_curve_agent
        list_available_curves,
        compare_vendors,
    ],
    "price_curve_agent": [
        get_hourly_price_distribution,
        compare_hub_zones,
        get_average_annual_price,       # shared
    ],
    "pbi_metrics_agent": [
        get_top_reports_by_views,
        get_top_users,
    ],
    "per_agent": [
        get_project_overview,
        get_project_health,
        get_milestone_dates,
        get_budget_overview,
        get_task_risks,
        get_portfolio_summary,
        get_recent_changes,
    ],
}


async def get_tools_for_agent(agent_name: str, tenant_id: str) -> list:
    """
    Returns the active tool list for an agent, filtered by per-tenant
    tool enable/disable config in chat_agent_tools table.
    """
    all_tools = TOOL_REGISTRY.get(agent_name, [])

    # Check DB for disabled tools for this tenant
    sql = """
    SELECT tool_name FROM chat_agent_tools
    WHERE tenant_id = :tenant_id AND agent_name = :agent_name AND is_enabled = 0
    """
    disabled = await SqlExecutor(tenant_id=tenant_id).execute(
        sql, {"tenant_id": tenant_id, "agent_name": agent_name}
    )
    disabled_names = {r["tool_name"] for r in disabled}

    return [t for t in all_tools if t.name not in disabled_names]
```

---

## 17. SQL Tool Implementations

### 17.1 SqlExecutor

```python
# app/agents/sql_executor.py
import aioodbc
from typing import Optional
from app.core.config import settings

class SqlExecutor:
    """
    Tenant-safe, read-only SQL executor for chatbot agent tools.
    All queries run on a read-only connection with:
    - 10-second timeout
    - 500-row result cap
    - Parameterized queries only
    - SET CONTEXT_INFO for RLS (tenant isolation)
    """

    def __init__(self, tenant_id: str, read_only: bool = True):
        self.tenant_id = tenant_id
        self.conn_str = settings.AZURE_SQL_READONLY_CONNECTION_STRING

    async def execute(self, sql: str, params: dict) -> list[dict]:
        async with aioodbc.connect(dsn=self.conn_str) as conn:
            async with conn.cursor() as cur:
                # Set RLS context
                await cur.execute(
                    "EXEC sp_set_session_context @key = N'tenant_id', @value = ?",
                    (self.tenant_id,)
                )

                # Convert :name params to ? positional for ODBC
                ordered_params, odbc_sql = self._convert_params(sql, params)

                await cur.execute(odbc_sql, ordered_params)

                columns = [desc[0] for desc in cur.description]
                rows = await cur.fetchmany(500)  # max 500 rows

                return [dict(zip(columns, row)) for row in rows]

    def _convert_params(self, sql: str, params: dict) -> tuple[list, str]:
        """Convert :name-style params to ? positional for pyodbc."""
        import re
        ordered = []
        def replacer(match):
            name = match.group(1)
            ordered.append(params[name])
            return "?"
        odbc_sql = re.sub(r':(\w+)', replacer, sql)
        return ordered, odbc_sql
```

---

## 18. Power BI Deep Link Tool

```python
# app/agents/tools/pbi_deeplink_tool.py
from app.services.powerbi_service import PowerBIService

PBI_REPORT_MAP = {
    "dayahead_curve_report":          {"menu_key": "mc_dayahead_curves"},
    "price_curve_hourly_report":      {"menu_key": "mc_price_curves"},
    "price_curve_comparison_report":  {"menu_key": "mc_price_curves"},
    "pbi_usage_metrics_report":       {"menu_key": "pbi_usage_dashboard"},
    "per_project_report":             {"menu_key": "per_project_overview"},
    "per_project_health_report":      {"menu_key": "per_project_health"},
    "per_portfolio_report":           {"menu_key": "per_portfolio_view"},
    "per_budget_report":              {"menu_key": "per_budget_tracker"},
}


async def generate_pbi_deep_links(
    pbi_report_keys: list[str],
    tenant_id: str,
    user_id: str,
) -> list[dict[str, str]]:
    """
    For each report key collected from agent results, generates a Power BI
    embed URL with a fresh embed token for direct navigation.
    """
    deep_links = []
    pbi_service = PowerBIService()

    for key in set(pbi_report_keys):
        report_meta = PBI_REPORT_MAP.get(key)
        if not report_meta:
            continue

        try:
            # Look up menu_item_id from menu_key
            menu_item = await get_menu_item_by_key(
                report_meta["menu_key"], tenant_id
            )
            if not menu_item:
                continue

            # Generate embed token (cached 55 min per README §17)
            embed_data = await pbi_service.get_embed_token(
                menu_item_id=menu_item["id"],
                tenant_id=tenant_id,
                user_id=user_id,
            )

            deep_links.append({
                "report_key":  key,
                "report_name": menu_item["label"],
                "embed_url":   embed_data["embed_url"],
                "embed_token": embed_data["embed_token"],
                "menu_key":    report_meta["menu_key"],
            })
        except Exception:
            pass  # Deep links are best-effort, don't fail the response

    return deep_links
```

---

# Part VII — Response Generation

## 19. Response Synthesizer

The synthesizer receives all `agent_results` from the state and generates a single coherent, markdown-formatted response.

```python
# app/agents/response_synthesizer.py

SYNTHESIZER_SYSTEM_PROMPT = """
You are the response writer for Vesper Analytics chatbot.
You receive structured data from one or more domain agents and must produce
a clear, concise, business-friendly response.

## Formatting Rules
1. Lead with a direct answer to the user's question (1-2 sentences)
2. Present tabular data as markdown tables when > 3 rows
3. For project health, use color emoji: 🟢 Green, 🟡 Yellow, 🔴 Red, 🔵 Blue
4. For financial data, use $K or $M units with appropriate precision
5. For prices, use $/MWh with 2 decimal places
6. For dates, use human-friendly format (e.g., "March 2026", not "2026-03-01")
7. End with a "**Related Reports**" section if PBI deep links are available
8. If data is empty, explain what was searched and suggest alternatives
9. Keep responses under 600 words unless user explicitly requests detail

## Related Reports Format
When deep links are available, append:
---
**Related Reports**
- [Day-Ahead Curves Dashboard]({{embed_url}}) — View interactive price curves in Power BI
- [Project Health Report]({{embed_url}}) — Full health breakdown in Power BI
"""


async def response_synthesizer_node(state: VesperChatState) -> dict:
    """Merge agent results and generate the final response."""

    # If already has final_response (e.g. clarification), pass through
    if state.final_response:
        return {}

    # Compile agent data for LLM
    agent_data_summary = []
    pbi_report_keys = []

    for result in state.agent_results:
        if result.error:
            agent_data_summary.append(f"[{result.domain}] Error: {result.error}")
        elif result.data:
            rows_preview = result.data[:20]  # limit for context window
            agent_data_summary.append(
                f"[{result.domain}] Data ({len(result.data)} rows total):\n"
                + json.dumps(rows_preview, indent=2, default=str)
            )
        if result.pbi_report_key:
            pbi_report_keys.append(result.pbi_report_key)

    # Generate PBI deep links
    deep_links = await generate_pbi_deep_links(
        pbi_report_keys, state.tenant_id, state.user_id
    )

    # Inject deep links into prompt if available
    links_block = ""
    if deep_links:
        links_block = "\n## Available Power BI Deep Links\n"
        for link in deep_links:
            links_block += f"- {link['report_name']}: {link['embed_url']}\n"

    user_question = next(
        (m.content for m in reversed(state.messages) if m.type == "human"), ""
    )

    response = await llm.ainvoke([
        SystemMessage(content=SYNTHESIZER_SYSTEM_PROMPT + links_block),
        HumanMessage(content=(
            f"User question: {user_question}\n\n"
            f"Agent data:\n" + "\n\n".join(agent_data_summary)
        )),
    ])

    return {
        "final_response": response.content,
        "pbi_deep_links": deep_links,
    }
```

---

## 20. Streaming to WebSocket

```python
# app/api/v1/endpoints/chat.py
from fastapi import WebSocket, WebSocketDisconnect
from langchain_core.callbacks import AsyncCallbackHandler
from app.agents.orchestrator import ChatOrchestrator

class WebSocketStreamingCallback(AsyncCallbackHandler):
    def __init__(self, websocket: WebSocket):
        self.websocket = websocket

    async def on_llm_new_token(self, token: str, **kwargs) -> None:
        await self.websocket.send_json({
            "type": "chat.token",
            "token": token
        })


@router.websocket("/ws/v1/chat")
async def chat_websocket(
    websocket: WebSocket,
    token: str,
    session_id: Optional[str] = None,
):
    await websocket.accept()

    try:
        # Authenticate
        user = await authenticate_ws_token(token)
        tenant_id = user.tenant_id

        # Load or create session
        session = await get_or_create_session(session_id, user)

        # Streaming callback
        callback = WebSocketStreamingCallback(websocket)
        orchestrator = ChatOrchestrator(callback=callback)

        while True:
            data = await websocket.receive_json()

            if data["type"] == "chat.message":
                user_message = data["content"]

                # Send typing indicator
                await websocket.send_json({"type": "chat.thinking"})

                # Run graph with streaming
                async for event in orchestrator.astream_events(
                    user_message=user_message,
                    session=session,
                    tenant_id=tenant_id,
                    user_id=user.id,
                    permissions=user.permissions,
                ):
                    if event["event"] == "on_chat_model_stream":
                        token = event["data"]["chunk"].content
                        if token:
                            await websocket.send_json({
                                "type": "chat.token",
                                "token": token
                            })
                    elif event["event"] == "on_chain_end":
                        # Final response ready
                        pass

                # Send completion + deep links
                final_state = orchestrator.last_state
                await websocket.send_json({
                    "type": "chat.complete",
                    "session_id": session.id,
                    "pbi_deep_links": final_state.pbi_deep_links,
                })

                # Persist message to DB
                await persist_chat_message(session.id, user_message,
                                           final_state.final_response)

    except WebSocketDisconnect:
        pass
```

---

# Part VIII — Extensibility

## 21. Adding a New Domain

The architecture is designed so a new domain (e.g., "Transmission Congestion" or "Renewable Credits") can be added in **five steps**:

### Step 1 — Define Tool File

```
app/agents/tools/transmission_tools.py
```

Create 2–5 `@tool` functions with Pydantic input schemas. Each tool generates parameterized SQL against the new view(s).

### Step 2 — Register Tools

```python
# app/agents/tool_registry.py
TOOL_REGISTRY["transmission_agent"] = [
    get_congestion_rent,
    get_binding_constraints,
    list_transmission_paths,
]
```

### Step 3 — Create Agent Node

```python
# app/agents/nodes/transmission_node.py
async def transmission_agent_node(state: VesperChatState) -> dict:
    tools = await get_tools_for_agent("transmission_agent", state.tenant_id)
    agent = create_react_agent(llm, tools=tools, prompt=TRANSMISSION_SYSTEM_PROMPT)
    result = await agent.ainvoke({"messages": state.messages, ...})
    return {"agent_results": [...]}
```

### Step 4 — Register Graph Node + Edges

```python
# app/agents/graph.py
graph.add_node("transmission_agent", transmission_agent_node)
graph.add_conditional_edges("supervisor", route_from_supervisor, {
    ...
    "transmission_agent": "transmission_agent",
})
graph.add_edge("transmission_agent", "response_synthesizer")
```

### Step 5 — Update Supervisor Prompt

Add a new intent `transmission` with example queries to `SUPERVISOR_SYSTEM_PROMPT`. The router immediately starts recognizing the new domain.

**No other changes required.** The state definition, orchestrator, WebSocket handler, and response synthesizer are all domain-agnostic.

---

# Part IX — Security & Multi-Tenancy

## 22. Permission Enforcement

Every tool call validates that the user has the required RBAC permission before executing SQL.

```python
# Permission gating decorator for tools
def require_permission(permission: str):
    def decorator(tool_func):
        async def wrapper(*args, **kwargs):
            tenant_id = kwargs.get("tenant_id")
            user_id = kwargs.get("user_id")

            has_perm = await check_permission(
                tenant_id=tenant_id,
                user_id=user_id,
                resource="chat",
                action=permission,
            )
            if not has_perm:
                return {"error": f"Permission denied: chat:{permission}"}

            return await tool_func(*args, **kwargs)
        return wrapper
    return decorator
```

**Permission matrix for chat tools:**

| Tool Category | Required Permission | Default Role |
|---------------|--------------------|-|
| DayAhead price queries | `chat:read_market_data` | Analyst+ |
| Price curve comparison | `chat:read_market_data` | Analyst+ |
| PBI usage metrics | `chat:read_pbi_metrics` | Manager+ |
| PER project overview | `chat:read_projects` | Viewer+ |
| PER budget data | `chat:read_budget` | Manager+ |
| PER portfolio summary | `chat:read_projects` | Viewer+ |
| Advanced SQL (escape hatch) | `chat:advanced_sql` | Admin only |

---

## 23. SQL Injection Prevention

The system enforces multiple layers:

1. **Parameterized queries** — all user-controlled values go through `:param` binding; no string formatting of SQL
2. **SqlExecutor validation** — rejects queries containing DDL/DML keywords before execution
3. **Approved view allowlist** — `general_sql_query` escape hatch validates table references
4. **Read-only connection** — the chatbot DB user has only `SELECT` on approved schemas
5. **Row-Level Security** — `sp_set_session_context` sets `tenant_id` for every connection; views enforce RLS via `SESSION_CONTEXT(N'tenant_id')`
6. **Result cap** — `fetchmany(500)` prevents data exfiltration via large result sets
7. **Query timeout** — 10-second timeout prevents long-running denial-of-service

---

# Part X — Database Schema

## 24. Chat Session Tables

```sql
-- Chat sessions (one per conversation thread)
CREATE TABLE chat_sessions (
    session_id      UNIQUEIDENTIFIER    PRIMARY KEY DEFAULT NEWID(),
    tenant_id       NVARCHAR(100)       NOT NULL,
    user_id         UNIQUEIDENTIFIER    NOT NULL,
    title           NVARCHAR(500),       -- Auto-generated from first message
    is_active       BIT                 DEFAULT 1,
    created_at      DATETIME2           DEFAULT GETUTCDATE(),
    updated_at      DATETIME2           DEFAULT GETUTCDATE(),

    INDEX IX_chat_sessions_tenant_user (tenant_id, user_id)
);

-- Chat messages (append-only)
CREATE TABLE chat_messages (
    message_id      BIGINT              IDENTITY(1,1) PRIMARY KEY,
    session_id      UNIQUEIDENTIFIER    NOT NULL
                    REFERENCES chat_sessions(session_id),
    tenant_id       NVARCHAR(100)       NOT NULL,
    role            NVARCHAR(20)        NOT NULL,   -- 'user' | 'assistant'
    content         NVARCHAR(MAX)       NOT NULL,
    intent          NVARCHAR(100),                   -- classified intent
    agents_used     NVARCHAR(500),                   -- comma-separated
    tool_calls      NVARCHAR(MAX),                   -- JSON array
    pbi_deep_links  NVARCHAR(MAX),                   -- JSON array
    token_count     INT,
    latency_ms      INT,
    created_at      DATETIME2           DEFAULT GETUTCDATE(),

    INDEX IX_chat_messages_session (session_id, created_at)
);

-- Per-tenant tool enable/disable
CREATE TABLE chat_agent_tools (
    id              INT                 IDENTITY(1,1) PRIMARY KEY,
    tenant_id       NVARCHAR(100)       NOT NULL,
    agent_name      NVARCHAR(100)       NOT NULL,
    tool_name       NVARCHAR(200)       NOT NULL,
    is_enabled      BIT                 DEFAULT 1,
    updated_by      UNIQUEIDENTIFIER,
    updated_at      DATETIME2           DEFAULT GETUTCDATE(),

    UNIQUE (tenant_id, agent_name, tool_name)
);

-- Conversation summaries (compressed long-context)
CREATE TABLE chat_session_summaries (
    summary_id      BIGINT              IDENTITY(1,1) PRIMARY KEY,
    session_id      UNIQUEIDENTIFIER    NOT NULL
                    REFERENCES chat_sessions(session_id),
    summary_text    NVARCHAR(MAX)       NOT NULL,
    covers_through  BIGINT              NOT NULL,   -- last message_id covered
    created_at      DATETIME2           DEFAULT GETUTCDATE()
);
```

---

# Appendix A — Prompt Templates

## A.1 Conversation Compression Prompt

Used when conversation history exceeds the context window (triggered at ~30 messages):

```python
COMPRESSION_PROMPT = """
Summarize the following conversation in 3-5 bullet points, preserving:
- The user's primary data interests (projects, markets, vendors)
- Any filters the user has established (e.g., "prefers Ascend curves for CAISO")
- Open questions or follow-ups that were not yet answered
- Key numbers or facts that may be referenced again

Be concise. The summary will be prepended to future messages as context.
"""
```

## A.2 Agent Node Wrapper Pattern

Standardized pattern for all domain agent nodes:

```python
async def create_domain_agent_node(
    domain: str,
    system_prompt: str,
) -> Callable:
    """Factory that creates a standardized agent node function."""

    async def agent_node(state: VesperChatState) -> dict:
        tools = await get_tools_for_agent(f"{domain}_agent", state.tenant_id)

        # Inject tenant context into all tools via partial
        bound_tools = [
            bind_tool_context(t, tenant_id=state.tenant_id, user_id=state.user_id)
            for t in tools
        ]

        agent = create_react_agent(
            llm,
            tools=bound_tools,
            state_modifier=system_prompt,
        )

        # Build filter context message
        filter_msg = f"\n[Filter Context: {state.filters.model_dump(exclude_none=True)}]"

        result = await agent.ainvoke({
            "messages": state.messages + [SystemMessage(content=filter_msg)],
        })

        # Extract tool call results and final answer
        tool_results = [
            m for m in result["messages"] if hasattr(m, "tool_call_id")
        ]

        agent_result = AgentResult(domain=domain)
        for tool_msg in tool_results:
            if isinstance(tool_msg.content, dict):
                agent_result.data = tool_msg.content.get("data", [])
                agent_result.pbi_report_key = tool_msg.content.get("pbi_report_key")

        return {"agent_results": state.agent_results + [agent_result]}

    return agent_node
```

---

# Appendix B — Example Conversations

## B.1 Day-Ahead Price Query

```
User:  What was the average day-ahead price for NP15 in CAISO for 2025 from Ascend?

System: [Supervisor] → intent: dayahead_curve, filters: {vendor: Ascend, market: CAISO,
         hub_zone: NP15, year: 2025}
         → routes to dayahead_agent

        [DayAhead Agent] → calls get_average_annual_price(vendor=Ascend, market=CAISO,
         hub_zone=NP15, year=2025)
         → SQL returns: [{AvgPrice_PerMWh: 48.73, AsOf: "Jan_2025", DataPoints: 8760}]

        [Synthesizer] → formats response + adds PBI deep link

Bot:   The average day-ahead price for **NP15 in CAISO for 2025** (Ascend forecast,
       As-Of Jan 2025) was **$48.73/MWh** based on 8,760 hourly data points.

       ---
       **Related Reports**
       - [Day-Ahead Curves Dashboard](https://app.powerbi.com/...) — View interactive
         price curves and vendor comparisons in Power BI
```

## B.2 Project Health Query

```
User:  What is the health status of Nazareth Solar?

System: [Supervisor] → intent: project_execution, filters: {project_name: "Nazareth Solar"}
         → routes to per_agent

        [PER Agent] → calls get_project_health(project_name="Nazareth Solar")
         → returns 9 health domain rows

        [Synthesizer] → formats with color emoji

Bot:   **Nazareth Solar** is currently in **Stage 4 (Pre-Con & Finance)** with
       overall project status **at risk** 🟡.

       | Domain | Status | % Complete | Due Date | Notes |
       |--------|--------|-----------|----------|-------|
       | Site Control | 🟢 Green | 100% | — | Site secured |
       | Permits | 🟡 Yellow | 65% | Jun 2025 | CEQA review pending |
       | Interconnection | 🔴 Red | 40% | Sep 2025 | Phase II study delayed |
       | Engineering | 🟢 Green | 80% | — | — |
       | Procurement | 🟡 Yellow | 30% | Aug 2025 | Panel pricing TBD |
       | Offtake | 🟢 Green | 100% | — | PPA executed |
       | Financing | 🔴 Red | 20% | Oct 2025 | Term sheet not yet received |
       | Construction | ⬜ N/A | 0% | — | Not started |
       | Schedule | 🟡 Yellow | — | — | COD at risk by ~3 months |

       **Key Risk**: Interconnection Phase II study delay is the critical path item.

       ---
       **Related Reports**
       - [Project Health Report](https://app.powerbi.com/...) — Full health breakdown
       - [Project Overview](https://app.powerbi.com/...) — Budget and milestone view
```

## B.3 Portfolio-Level Query

```
User:  Show me all projects that are off track (red)

System: [Supervisor] → intent: project_execution, filters: {status_color: "off_track"}
         → routes to per_agent

        [PER Agent] → calls get_portfolio_summary(status_color="off_track")

Bot:   There are **3 projects** currently flagged as **off track** 🔴:

       | Project | Stage | Technology | Market | Budget | Actuals | Summary |
       |---------|-------|-----------|--------|--------|---------|---------|
       | Nazareth Solar | Stage 4 | Solar | CAISO | $2.1M | $1.4M | Interconnection delay |
       | Sierra Wind | Stage 3 | Wind | ERCOT | $800K | $620K | Permitting stalled |
       | Mojave BESS | Stage 5 | BESS | CAISO | $5.4M | $4.9M | EPC contractor dispute |
```

## B.4 Vendor Comparison Query

```
User:  Compare Ascend vs Hitachi day-ahead prices for SP15 in CAISO for 2025

System: [DayAhead Agent] → calls compare_vendors(market=CAISO, hub_zone=SP15, year=2025)

Bot:   **Day-Ahead Price Comparison — SP15, CAISO, 2025**

       | Vendor | As-Of | Avg Price ($/MWh) |
       |--------|-------|------------------|
       | Ascend | Jan_2025 | $52.14 |
       | Hitachi | Q1_2025 | $54.80 |

       Hitachi's forecast is **$2.66/MWh (5.1%) higher** than Ascend's for SP15
       in 2025. This difference is most pronounced during peak hours (HE 16-21).
```

## B.5 Recent Changes Query

```
User:  What milestone dates changed in the last week?

System: [Supervisor] → intent: project_execution, filters: {field_changed: "FinishDate"}
         → routes to per_agent

        [PER Agent] → calls get_recent_changes(since_date="2026-02-20",
                       field_changed="FinishDate")

Bot:   **5 milestone date changes** recorded in the last 7 days:

       | Date | Project | Task | Previous Date | New Date | Change |
       |------|---------|------|--------------|----------|--------|
       | Feb 26 | Sierra Wind | Execute Interconnection Agreement | Mar 2026 | Jun 2026 | +3 months |
       | Feb 25 | Nazareth Solar | Project Finance | Aug 2025 | Nov 2025 | +3 months |
       | Feb 24 | Desert Sun | EPC FNTP | Apr 2026 | Apr 2026 | No change (re-confirmed) |
       | Feb 23 | Mojave BESS | Mechanical Completion | Dec 2026 | Feb 2027 | +2 months |
       | Feb 21 | Kern Wind | Commercial Operation | Jun 2027 | Sep 2027 | +3 months |

       **Pattern**: Multiple projects are pushing COD and key financing milestones
       to the right by 2-3 months, suggesting systemic supply chain or permitting pressure.
```

---

*Document version: 1.0 | Created: 2026-02-27 | Owner: Vesper Analytics Engineering*
*Companion: `docs/README.md` (platform architecture) | Stack: LangGraph + Claude claude-sonnet-4-6 + Azure SQL*
