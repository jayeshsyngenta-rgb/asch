# PNS Demand Forecast: Multi-Agent System (MAS) Report

---

## 1. Executive Summary

The **PNS Demand Forecast Multi-Agent System** is an AI-powered conversational chatbot built for Syngenta's demand planning teams. It enables non-technical business users to interact with complex demand forecasting data through natural language queries.

The system is designed as a **Multi-Agent System (MAS)** where a central orchestrator agent (the Demand Analyst) delegates specialized tasks to five domain-specific sub-agents. Each agent is an independent n8n workflow with its own LLM, logic, and data access layer. Additionally, a standalone FVA (Forecast Value Added) agent operates independently for forecast accuracy analysis.

**Key Capabilities:**
- Natural language querying of demand forecast data
- Scenario-based product/region/brand-level analysis
- Forecast volume retrieval with trend visualization
- Model performance metrics reporting (Business FA, MAPE, Bias, R-squared)
- Feature/driver importance analysis (what drives consumption predictions)
- Data drift and reconciliation monitoring
- Forecast Value Added (FVA) analysis across opinion lines
- Automatic chart generation (line, bar, pie, heatmap)
- Multi-country support: Brazil, USA, Canada, Argentina, Italy

**Platform:** Built on n8n (workflow automation platform), powered by multiple LLMs (Claude Sonnet 4.5, Gemini 2.5 Flash, GPT-4o) accessed via Portkey proxy, with Databricks SQL as the data warehouse.

---

## 2. System Architecture Overview

### 2.1 Architecture Pattern: Orchestrator-Worker MAS

The system follows the **Orchestrator-Worker** multi-agent pattern:

- **One Orchestrator Agent** (Demand Analyst) receives all user queries and decides which specialized worker agent(s) to invoke.
- **Five Specialized Worker Agents** each handle a specific domain task (forecasting, metrics, drift detection, driver analysis, scenario detection).
- **One Standalone Agent** (FVA Agent) operates independently with its own webhook entry point for forecast accuracy analysis.

```
                          +------------------+
                          |      User        |
                          | (Chat Interface) |
                          +--------+---------+
                                   |
                            HTTP POST (Webhook)
                                   |
                 +-----------------v------------------+
                 |                                    |
     +-----------v-----------+          +-------------v-----------+
     |   Demand Analyst      |          |     FVA Agent           |
     |   (Orchestrator)      |          |   (Standalone Agent)    |
     |   LLM: Claude 4.5    |          |   LLM: Claude 4.5      |
     +-----------+-----------+          +---+---+---+---+---------+
                 |                          |   |   |   |
     +-----------+-----------+              |   |   |   |
     |     |     |     |     |              |   |   |   |
     v     v     v     v     v              v   v   v   v
   +---+ +---+ +---+ +---+ +---+     +-----+ +---+ +---+ +------+
   | S | | F | | M | | Dr| | DD|     |Skill| |Sce| |Qry| |Month |
   | c | | o | | e | | iv| | ri|     |Inst | |nar| |   | |ly    |
   | e | | r | | t | | er| | ft|     |     | |ios| |   | |Detl  |
   | n | | e | | r | |   | |   |     +-----+ +---+ +---+ +------+
   | a | | c | | i | |   | |   |
   | r | | a | | c | |   | |   |
   | i | | s | | s | |   | |   |
   | o | | t | |   | |   | |   |
   +---+ +---+ +---+ +---+ +---+

   Legend:
   Scen = Scenario Detection Agent
   Forec = Forecast Agent
   Metric = Model Metrics Agent
   Driver = Driver/Feature Agent
   Drift = Data Drift Detection Agent
   SkillInst = Fetch Skill Instructions (tool)
   Scnarios = FVA Scenarios (tool)
   Qry = FVA Query (sub-workflow)
   MonthlyDetl = FVA Detailed Monthly (tool)
```

### 2.2 Communication Flow

**Standard Query Flow (Demand Analyst path):**

```
User Query (natural language)
    |
    v
[1] Webhook receives HTTP POST (with chatInput, chatHistory, country)
    |
    v
[2] Demand Analyst Agent (Orchestrator) interprets the query
    |
    v
[3] Calls Scenario Detection Agent first (maps query -> valid scenario names)
    |
    v
[4] Based on user intent, calls one or more specialist agents:
    - Forecast Agent (for volume/trend data)
    - Model Metrics Agent (for accuracy/performance)
    - Driver Agent (for feature importance)
    - Data Drift Agent (for data quality checks)
    |
    v
[5] Each sub-agent:
    a) Parses input JSON
    b) Queries Databricks SQL warehouse
    c) Processes results through AI summarizer
    d) Returns structured JSON response
    |
    v
[6] Demand Analyst synthesizes all tool outputs
    |
    v
[7] Generates final markdown response with tables, charts, and insights
    |
    v
[8] Streams response back to user via webhook
```

**FVA Agent Flow (independent path):**

```
User Query
    |
    v
[1] Webhook receives HTTP POST
    |
    v
[2] FVA Agent parses query, extracts country + filters
    |
    v
[3] Calls tools: fetch_skill_instructions -> FVA_scenarios -> FVA_monthly / FVA_detailed_monthly / FVA_query
    |
    v
[4] Returns analysis with tables and charts
```

### 2.3 Inter-Agent Communication Protocol

All sub-agents are invoked as **n8n "Execute Workflow" tool nodes** by the orchestrator. The communication contract is:

**Input (from Orchestrator to Sub-Agent):**
```json
{
  "scenario_names": ["SCENARIO_1", "SCENARIO_2"],
  "country": "Brazil",
  "instructions": "Clear 1-2 line instruction for the sub-agent"
}
```

**Output (from Sub-Agent back to Orchestrator):**
Each agent returns a structured JSON response specific to its domain (forecast rows, metrics, driver rankings, drift checks, etc.)

---

## 3. Detailed Agent Profiles

---

### 3.1 Demand Analyst Agent (Orchestrator)

| Property | Details |
|---|---|
| **Role** | Central orchestrator and user-facing conversational agent |
| **Workflow Name** | `workflow_Demand_Analyst_Agent` |
| **LLM** | Claude Sonnet 4.5 (via AWS Bedrock through Portkey proxy) |
| **Entry Point** | HTTP POST Webhook (`/n8n-demand-analyst-agent`) |
| **Response Mode** | Streaming |
| **Workflow Size** | ~124 KB (largest workflow) |

**Responsibilities:**
1. Receives and interprets natural language user queries
2. Classifies user intent (forecast, metrics, drivers, drift, accuracy)
3. Always calls Scenario Detection Agent first to resolve scenario names
4. Routes to appropriate specialist agent(s) based on intent
5. Can call multiple agents for combined queries (e.g., forecast + accuracy)
6. Synthesizes all tool outputs into a comprehensive markdown response
7. Generates charts (line, bar, pie, heatmap) as JSON code blocks
8. Handles clarification flows when scenarios are ambiguous
9. Manages country override logic (retry with different country if needed)

**Tools Available to the Orchestrator:**

| Tool Name | Target Agent | When Used |
|---|---|---|
| Scenario Detector | Scenario Detection Agent | Always called first for every query |
| Forecast Agent | Forecast Agent | Forecast volumes, trends, breakdowns, comparisons |
| Model Metrics Agent | Model Metrics Agent | Model performance, Business FA, MAPE, Bias, R-squared |
| Data Drift Agent | Data Drift Detection Agent | Data drift / recon check details |
| Feature/Driver Agent | Driver Agent | Top drivers, feature importance analysis |

**Key Orchestration Rules:**
- **Mandatory first step:** Always call Scenario Detector before any other tool
- **Clarification handling:** If Scenario Detector returns a "question" field (ambiguity), stop and ask the user before proceeding
- **Country override:** If `needs_country_retry=true`, retry Scenario Detector with the `country_hint`
- **Combined queries:** For accuracy + forecast queries, calls both Model Metrics and Forecast agents, then merges results
- **Retry control:** Maximum 1 retry per tool; stops if output is semantically identical to prevent loops
- **ALL-COUNTRY:** If user asks for whole-country details, uses the `ALL-COUNTRY` scenario

**Response Format:**
- Output in **Markdown only** (no raw HTML)
- Tables for data comparisons
- Charts as fenced JSON code blocks with `chart` language identifier
- Supports: line, bar, scatter, area, pie, heatmap chart types
- Automatic chart generation for forecast trends, multi-scenario comparisons, and driver rankings

---

### 3.2 Scenario Detection Agent

| Property | Details |
|---|---|
| **Role** | Maps user queries to valid scenario names from the database |
| **Workflow Name** | `workflow_scenario_detection` |
| **LLM** | Gemini 2.5 Flash (for fast, lightweight matching) + Claude Sonnet 4.5 (for scenario AI agent) |
| **Trigger** | Execute Workflow Trigger (called by Demand Analyst) |

**What It Does:**
- Receives a user query and country name
- Looks up valid scenarios from a pre-cached `all_scenarios` data table (populated by a scheduled refresh chain)
- Uses AI to match user's natural language to exact scenario names
- Handles fuzzy matching, abbreviations, and partial names
- Returns structured JSON with matched scenarios

**Input:** User query + country name
**Output:**
```json
{
  "scenario_names": ["ACTARA 250 WG", "AMPLIGO"],
  "country_hint": "Brazil",
  "needs_country_retry": false,
  "question": null
}
```

**Key Logic:**
- **Exact match priority:** If user mentions a specific scenario name that exists, return only the exact match
- **Multi-category ambiguity:** If a term appears in multiple scenario categories (e.g., "CALLISTO" exists as both tradename and product brand), triggers a clarification question back to the user
- **Country handling:** If scenario not found in current country, sets `needs_country_retry=true` with `country_hint` for the orchestrator to retry
- **Product line/brand mapping:** Scenarios in `product_line_product_brand` category are split (e.g., `Insecticides_ACTARA 250 WG` returns `ACTARA 250 WG`)
- **Scenario refresh:** Has a separate scheduled chain that queries Databricks for all distinct scenarios per country (Brazil, USA, Argentina, Italy, Canada) and caches them in n8n data tables

**Supported Scenario Categories (vary by country):**
- `tradename` (product trade names)
- `region` (geographic regions)
- `product_line` (e.g., Insecticides, Fungicides, Herbicides)
- `product_brand` (brand groupings)
- `br_family_group_sop` (Brazil-specific family groups)
- `nombre_comercial_agrupado` (Argentina-specific grouped commercial names)
- `commercial_unit` / `material_group` (USA-specific)
- `product_line_product_brand` (mapping category)
- `ALL-COUNTRY` (whole country aggregate)

---

### 3.3 Forecast Agent

| Property | Details |
|---|---|
| **Role** | Retrieves and presents demand forecast data |
| **Workflow Name** | `workflow_forecast_agent` |
| **LLMs** | Gemini 2.5 Flash (SQL generation + validation) + Claude Sonnet 4.5 (data summarization) |
| **Trigger** | Execute Workflow Trigger (called by Demand Analyst) |
| **Execution Timeout** | 120 seconds |

**What It Does:**
- Receives scenario names, country, and instructions
- Dynamically generates SQL queries against the Databricks forecast tables
- Validates generated SQL in an iterative loop
- Executes the query and summarizes results using AI

**Node Pipeline:**
```
Input -> Parse JSON -> Get Country Metadata (from cache)
  -> [SQL Generation Loop]:
       Generate SQL (LLM) -> Validate SQL (LLM) -> If valid, execute; if not, regenerate
  -> Get Access Token -> Execute SQL on Databricks
  -> Parse SQL Result -> AI Data Summarizer (Claude) -> Structured Output -> Return
```

**Key Features:**

1. **Metadata Caching System:**
   - A scheduled trigger (weekly) pre-fetches and caches: table schema, available run dates, min/max yearmonth
   - Stored in n8n `forecast_cache` data table
   - Agents read from cache instead of querying metadata each time

2. **Iterative SQL Generation & Validation:**
   - **Generator Agent (Gemini 2.5 Flash):** Writes PostgreSQL SELECT queries based on user request + schema + metadata
   - **Validator Agent (Gemini 2.5 Flash):** Checks if the SQL is correct (run_date filter present, time range correct, columns match)
   - Loop: Generate -> Validate -> If invalid, feed back error to Generator -> Regenerate
   - Safety: After 4 iterations, validator auto-passes to prevent infinite loops

3. **AI Data Summarizer (Claude Sonnet 4.5):**
   - Takes raw SQL result rows and produces structured JSON output
   - Output format includes: status, data_found, scenarios array (with observations and extra points), overall insights

**Data Source:**
- Table: `pns_analytics_{country}_output_forecast` on Databricks
- Key columns: `scenario_name`, `yearmonth`, `pns_run_date`, `forecast_volume`, `actual_volume`, `product_line`, `region`, `tradename`, etc.

**Output Format:**
```json
{
  "status": "ok",
  "data_found": true,
  "scenarios": [
    {
      "scenario_name": "ACTARA 250 WG",
      "observation": "Detailed analysis...",
      "extra_points": ["point 1", "point 2"]
    }
  ],
  "overall_insights": ["Cross-scenario insight"]
}
```

---

### 3.4 Model Metrics Agent

| Property | Details |
|---|---|
| **Role** | Returns model performance metrics for forecast scenarios |
| **Workflow Name** | `workflow_model_metrics` |
| **LLMs** | Gemini 2.5 Flash (run date selection) |
| **Trigger** | Execute Workflow Trigger (called by Demand Analyst) |
| **Execution Timeout** | 120 seconds |

**What It Does:**
- Receives scenario names, country, and instructions
- Determines which forecast run date(s) to use based on user intent
- Queries the model metrics table for performance data
- Returns raw metrics for the orchestrator to analyze

**Node Pipeline:**
```
Input -> Parse JSON -> Get Country Metadata
  -> AI Agent (Run Date Selector - Gemini 2.5 Flash)
  -> Assemble SQL Query -> Get Access Token -> Execute SQL on Databricks
  -> Parse SQL Result -> Return
```

**Key Logic:**
- **Run Date Selection:** Uses an AI agent (Gemini 2.5 Flash) to intelligently select the correct `pns_run_date` from available dates
  - Default: Most recent run date
  - Supports: Specific date requests, multi-date comparison ("compare last 3 runs")
  - Returns dates in chronological order
- **No AI summarizer:** Unlike other agents, this one returns raw parsed data (the Demand Analyst orchestrator handles interpretation)

**Data Source:**
- Table: `pns_analytics_{country}_output_metrics` on Databricks
- Key metrics: Business FA (Forecast Accuracy), MAPE, Bias, R-squared, RMSE, MAE
- Filtered by: scenario_name, pns_run_date

**Metrics Explained (for presentation context):**
| Metric | Description |
|---|---|
| Business FA | Forecast Accuracy - how close forecasts are to actuals (higher = better) |
| MAPE | Mean Absolute Percentage Error (lower = better) |
| Bias | Systematic over/under-forecasting tendency |
| R-squared | How well the model explains variance in the data |
| RMSE | Root Mean Square Error |
| MAE | Mean Absolute Error |

---

### 3.5 Driver / Feature Importance Agent

| Property | Details |
|---|---|
| **Role** | Analyzes and explains what drives consumption predictions |
| **Workflow Name** | `workflow_driver_agent` |
| **LLMs** | Gemini 2.5 Flash (run date selection) + Claude Sonnet 4.5 (data summarization) |
| **Trigger** | Execute Workflow Trigger (called by Demand Analyst) |
| **Execution Timeout** | 120 seconds |

**What It Does:**
- Receives scenario names, country, and instructions
- Queries the feature ranking table (based on SHAP values from ML models)
- AI summarizer translates technical ML output into business-friendly language
- Returns ranked drivers with insights per scenario and forecast horizon

**Node Pipeline:**
```
Input -> Parse JSON -> Get Country Metadata
  -> AI Agent (Run Date Selector - Gemini 2.5 Flash)
  -> Assemble SQL Query -> Get Access Token -> Execute SQL on Databricks
  -> Parse SQL Result -> AI Data Summarizer (Claude Sonnet 4.5) -> Structured Output -> Return
```

**Key Features:**

1. **SHAP Value Translation:**
   - Internally uses `mean_abs_shap` for ranking but NEVER exposes SHAP values to users
   - Translates to business-friendly language: "strongly influences consumption predictions" instead of "high SHAP value"
   - Uses `commone_feature_name` (friendly name) when available, falls back to raw feature name

2. **Forecast Horizon Analysis:**
   - Short-term: 1-4 months
   - Medium-term: 5-12 months
   - Long-term: 13-24 months
   - If user doesn't specify horizon, returns drivers across ALL horizons

3. **Run Comparison:**
   - Can compare drivers between current and past runs (if data available)
   - Highlights if drivers have remained stable or shifted

**Data Source:**
- Table: `pns_analytics_{country}_output_feature_ranking` on Databricks
- Key columns: `scenario_name`, `feature`, `commone_feature_name`, `mean_abs_shap`, `ranking`, `forecast_horizon`, `run_date`

**Output Format:**
```json
{
  "status": "ok",
  "data_found": true,
  "scenarios": [
    {
      "scenario_name": "ACTARA 250 WG",
      "observation": "The top drivers for ACTARA 250 WG consumption predictions are...",
      "extra_points": ["Weather patterns strongly influence short-term predictions", "..."]
    }
  ],
  "overall_insights": ["Across all scenarios, seasonal weather is the dominant driver"]
}
```

---

### 3.6 Data Drift Detection Agent

| Property | Details |
|---|---|
| **Role** | Monitors data quality through drift and reconciliation checks |
| **Workflow Name** | `workflow_data_drift_detection` |
| **LLMs** | None (pure data pipeline - no AI summarization) |
| **Trigger** | Execute Workflow Trigger (called by Demand Analyst) |
| **Execution Timeout** | 120 seconds |

**What It Does:**
- Receives scenario names and country
- Queries the data reconciliation checks table
- Returns raw drift/recon data or a "no significant drifts found" message
- Simplest agent in the system - no AI summarization layer

**Node Pipeline:**
```
Input -> Parse JSON -> Assemble SQL Query -> Get Access Token
  -> Execute SQL on Databricks -> Parse SQL Result
  -> If rows exist: return data
  -> If no rows: return "No significant data drifts found"
```

**Key Logic:**
- Direct SQL query without AI-based query generation (query is deterministic)
- Simple branching: data found vs. no data
- No run date selection needed (queries all available data for the scenario)

**Data Source:**
- Table: `pns_analytics_{country}_data_recon_checks_forecast` on Databricks
- Checks for: data completeness, distribution shifts, missing values, outlier detection

**Output:**
- If drifts found: Returns raw reconciliation check rows
- If no drifts: `{"error": "No significant data drifts found for these scenarios."}`

---

### 3.7 FVA Agent (Forecast Value Added) - Standalone

| Property | Details |
|---|---|
| **Role** | Analyzes forecast accuracy across opinion lines and methods |
| **Workflow Name** | `FVA_Agent_Main` |
| **LLM** | Claude Sonnet 4.5 (via AWS Bedrock through Portkey) |
| **Entry Point** | HTTP POST Webhook (`/fva-agent`) - independent of Demand Analyst |
| **Response Mode** | Streaming |

**What It Does:**
- Operates as a completely **independent agent** (not called by the Demand Analyst)
- Analyzes Forecast Value Added (FVA) - measuring which forecasting method adds the most value
- Compares 7 opinion lines (forecast methods) across multiple metrics
- Supports material-level drill-downs

**Tools Available:**

| Tool | Type | Purpose |
|---|---|---|
| `fetch_skill_instructions` | n8n Data Table | Retrieves detailed instructions for each tool from a skills table |
| `FVA_scenarios` | HTTP Request | Returns available filter values (product lines, tradenames, material IDs) per country |
| `FVA_monthly` | HTTP Request (POST) | **Primary tool** - Returns monthly FA, bias, MAPE for all 7 opinion lines |
| `FVA_detailed_monthly` | HTTP Request (POST) | Returns material-level breakdown for a specific opinion line and month |
| `FVA_query` | Sub-workflow | Builds query payload, fetches aggregated FVA data, filters via AI |

**7 Opinion Lines (Forecast Methods) Analyzed:**

| Opinion Line Key | Description |
|---|---|
| `ext_stat_forecast` | External Statistical Forecast |
| `fy_stat_forecast` | Full Year Statistical Forecast |
| `dp_usf` | Demand Planning USF |
| `sop_csp_p_and_s` | SOP / CSP P&S |
| `pm_usf` | Product Manager USF |
| `consensus_usf` | Consensus USF |
| `hos_usf` | Head of Sales USF |

**Key Features:**
- Uses `lag3_agg3` data (3-month lag aggregation) for all accuracy calculations
- Supports multi-dimensional filtering: product_line, tradename, material_id, UOM, ABC classification, XYZ classification, lead AI category, design code
- Can compare opinion lines head-to-head
- Generates charts (line, bar, pie, heatmap) for visual analysis

### 3.8 FVA Query Sub-Workflow (Helper)

| Property | Details |
|---|---|
| **Role** | Helper workflow that fetches and filters FVA data |
| **Workflow Name** | `FVA_query` |
| **LLM** | GPT-4o (via Azure OpenAI through Portkey) |
| **Trigger** | Execute Workflow Trigger (called by FVA Agent Main) |

**What It Does:**
- Receives a structured query with country, filters, fields_required, and user_query_context
- Calls the FVA monthly API endpoint
- Uses GPT-4o to intelligently filter the API response based on what the user actually needs
- Returns only relevant sections (aggregated, best_forecast, monthly) with only requested fields

**Filtering Logic:**
- "overall/average/aggregated" queries -> returns `aggregated_data.weighted_average` only
- "best/ranking/which method" queries -> returns `best_forecast` + `aggregated_data`
- "trend/over time/monthly" queries -> returns all 12 months of `monthly_data`
- "recent/latest" queries -> returns last 3 months of `monthly_data`

---

## 4. Technology Stack

### 4.1 AI / LLM Layer

| LLM | Provider | Used In | Purpose |
|---|---|---|---|
| **Claude Sonnet 4.5** | Anthropic (via AWS Bedrock) | Demand Analyst, Driver Agent, Forecast Agent (summarizer), FVA Agent | Main reasoning, orchestration, data summarization |
| **Gemini 2.5 Flash** | Google (via Vertex AI) | Forecast Agent (SQL gen/validation), Model Metrics Agent (run date selector), Driver Agent (run date selector) | Fast, lightweight tasks: SQL generation, date selection |
| **GPT-4o** | OpenAI (via Azure OpenAI) | FVA Query sub-workflow | Data filtering and response shaping |

All LLM calls are routed through **Portkey** (`portkey.syngenta.com/v1`) - an AI gateway that provides:
- Unified API interface for multiple LLM providers
- Request routing and load balancing
- Monitoring and observability

### 4.2 Data Layer

| Component | Details |
|---|---|
| **Databricks SQL Warehouse** | Primary data source for forecast data, metrics, drivers, drift checks |
| **n8n Data Tables** | Internal storage for metadata caching (schemas, run dates), scenario lists, environment variables, FVA skill instructions |
| **Google Sheets** | Test case management for evaluation framework |
| **REST APIs** | FVA data served via internal API (`deawilapsd0001.smartmart.syngenta.org:5001`) |

### 4.3 Platform & Infrastructure

| Component | Details |
|---|---|
| **n8n** | Workflow automation platform hosting all agent workflows |
| **n8n DEV (UAT)** | `https://n8n-uat.syngenta.com/` |
| **n8n PROD** | `https://n8n.syngenta.com/` |
| **Companion UI** | `pns-demand-forecast-ui` (React app at `localhost:3000`) |
| **Deployment** | Automated Python CLI tool for workflow import/credential management |

---

## 5. Key Design Patterns

### 5.1 Orchestrator-Worker Pattern
The Demand Analyst acts as a central coordinator that understands user intent and delegates to specialized workers. This provides:
- **Separation of concerns:** Each agent is an expert in one domain
- **Scalability:** New agents can be added without modifying the orchestrator significantly
- **Maintainability:** Each agent can be updated independently

### 5.2 Tool-Use Pattern (ReAct-style)
The orchestrator uses LLM tool-calling capabilities to invoke sub-agents. The LLM decides:
- Which tool(s) to call
- What parameters to pass
- Whether to call multiple tools sequentially or in combination
- When to stop and return results

### 5.3 Iterative SQL Validation Loop
The Forecast Agent uses a Generator-Validator loop:
```
Generate SQL (LLM) -> Validate (LLM) -> Execute or Regenerate
```
This ensures SQL correctness before hitting the database, with a safety valve (max 4 iterations) to prevent infinite loops.

### 5.4 Scenario Disambiguation with Clarification
When user queries are ambiguous (e.g., "CALLISTO" could be a tradename or a product brand), the system:
1. Detects ambiguity
2. Generates a clarification question
3. Returns the question to the user
4. Waits for user response
5. Proceeds with the clarified intent

This creates a **conversational disambiguation flow** rather than guessing.

### 5.5 Structured Output Parsing
All agents use n8n's Structured Output Parser to enforce JSON schema on LLM responses. This ensures:
- Consistent response format
- Parseable output for downstream processing
- No free-form text that could break the pipeline

### 5.6 Metadata Caching
The Forecast Agent pre-caches database metadata (table schemas, available run dates, yearmonth ranges) into n8n Data Tables via a weekly scheduled job. This:
- Reduces Databricks query load
- Speeds up agent response times
- Provides schema context to the SQL generator without runtime queries

### 5.7 Human-Friendly AI Translation
The Driver Agent exemplifies a key pattern: translating technical ML output (SHAP values, feature importance scores) into business-friendly language. Rules include:
- Never mention "SHAP", "shapley", or raw numeric values
- Use terms like "consumption predictions" instead of "model output"
- Present rankings without exposing the underlying statistical methodology

---

## 6. Evaluation Framework

The Demand Analyst workflow includes a built-in **automated evaluation pipeline** for quality assurance:

### 6.1 Test Case Management
- Test cases stored in Google Sheets (`demand forecast agent test cases all country`)
- Each test case has: input query, expected tool count, expected scenario list, expected country

### 6.2 Evaluation Dimensions (6 Scoring Criteria)

Each response is scored from 1-5 on:

| Dimension | Weight | What It Measures |
|---|---|---|
| **Query Fulfillment** | 40% | Does the response fully answer the user's question? |
| **Simple Language** | 15% | Is it understandable by non-technical users? |
| **Straight to Point** | 15% | Is it focused without unnecessary tangents? |
| **Business Language** | 15% | Uses business terminology and actionable insights? |
| **Data Accuracy** | 10% | Are numbers, scenario names, and calculations correct? |
| **Formatting** | 5% | Is the output well-structured with proper tables/charts? |

### 6.3 Automated Metrics
- **Tool Count Match:** Did the agent use the expected number of tools?
- **Scenario Correctness:** Did the agent identify the correct scenarios?
- **Domain-Specific Correctness:** Separate scoring for Forecast, Model Metrics, Data Drift, and Driver correctness

### 6.4 Query Classification
The evaluator first classifies each query type to adjust scoring expectations:
- **Simple Lookup:** Direct data retrieval (penalize verbosity)
- **Analytical:** Data + interpretation (allow detailed explanations)
- **Complex Multi-Step:** Comprehensive analysis (expect length)
- **Ambiguous:** Reward clarification attempts
- **Data Unavailable:** Reward transparency about missing data

---

## 7. Agent Interaction Matrix

This matrix shows which agent calls which:

| Caller | Scenario Detection | Forecast | Model Metrics | Driver | Data Drift | FVA Tools |
|---|---|---|---|---|---|---|
| **Demand Analyst** | Yes (always first) | Yes | Yes | Yes | Yes | No |
| **FVA Agent** | No | No | No | No | No | Yes (4 tools) |
| **Scenario Detection** | - | No | No | No | No | No |
| **Forecast** | No | - | No | No | No | No |
| **Model Metrics** | No | No | - | No | No | No |
| **Driver** | No | No | No | - | No | No |
| **Data Drift** | No | No | No | No | - | No |

**Key observations:**
- Only the Demand Analyst orchestrator calls other agents
- Sub-agents never call each other (star topology)
- FVA Agent is completely independent

---

## 8. Countries Supported

| Country | Scenario Categories Available |
|---|---|
| **Brazil** | tradename, region, product_line, product_brand, br_family_group_sop, product_line_product_brand, ALL-COUNTRY |
| **USA** | commercial_unit, material_group, tradename_en, scenario_name, ALL-COUNTRY |
| **Argentina** | tradename, region, product_line, nombre_comercial_agrupado, ALL-COUNTRY |
| **Italy** | tradename, region, product_line, product_brand, ALL-COUNTRY |
| **Canada** | tradename, region, product_line, ALL-COUNTRY |

---

## 9. Deployment Architecture

### 9.1 Environments

| Environment | URL | Purpose |
|---|---|---|
| **DEV (UAT)** | `https://n8n-uat.syngenta.com/` | Development and testing |
| **PROD** | `https://n8n.syngenta.com/` | Production |

### 9.2 Deployment Process

Automated via Python CLI (`update_workflow_code/main.py`):

1. **Clean up** existing workflows (delete by `_dev`/`_prod` suffix)
2. **Create credentials** (OpenAI/Portkey, Postgres) with new IDs
3. **Update workflow JSONs** with environment-specific credential IDs, table IDs, project IDs
4. **Import sub-workflows first** (order matters):
   - Forecast Agent
   - Model Metrics Agent
   - Data Drift Detection Agent
   - Driver Agent
   - Scenario Detection Agent
5. **Import Demand Analyst last** (needs all sub-workflow IDs for tool references)
6. **Optionally transfer** to n8n project space, tag, and activate

### 9.3 Environment Naming Convention
All deployed workflows get an environment suffix:
- DEV: `workflow_forecast_agent_dev`
- PROD: `workflow_forecast_agent_prod`

---

## 10. Summary: Why This Is a Multi-Agent System

This system qualifies as a **Multi-Agent System (MAS)** because it exhibits the following MAS characteristics:

| MAS Property | How This System Implements It |
|---|---|
| **Autonomy** | Each agent operates independently with its own LLM, prompts, and logic |
| **Specialization** | Each agent is an expert in one domain (forecasting, metrics, drivers, etc.) |
| **Coordination** | The Demand Analyst orchestrator coordinates agent invocations based on user intent |
| **Communication** | Agents communicate via structured JSON messages through n8n tool calls |
| **Modularity** | Agents can be added, removed, or updated independently |
| **Scalability** | New countries and agents can be added without redesigning the system |
| **Emergent Behavior** | Complex queries (forecast + accuracy + drivers) are answered by combining outputs from multiple specialized agents, producing richer insights than any single agent could provide |
| **Heterogeneity** | Different agents use different LLMs optimized for their task (Claude for reasoning, Gemini for speed, GPT-4o for filtering) |

---

## Appendix A: Node Count per Workflow

| Workflow | Approximate Node Count | Complexity |
|---|---|---|
| Demand Analyst (Orchestrator) | ~30+ nodes | High (includes evaluation pipeline) |
| Scenario Detection | ~15+ nodes | Medium (multi-country query chains) |
| Forecast Agent | ~15 nodes | Medium (SQL gen/validation loop + caching) |
| Model Metrics Agent | ~10 nodes | Low-Medium |
| Driver Agent | ~12 nodes | Medium |
| Data Drift Detection | ~8 nodes | Low (simplest agent) |
| FVA Agent Main | ~8 nodes | Medium (multiple HTTP tools) |
| FVA Query | ~5 nodes | Low (helper sub-workflow) |

---

## Appendix B: Glossary

| Term | Definition |
|---|---|
| **n8n** | Open-source workflow automation platform used to build and host the agent workflows |
| **Scenario** | A forecast unit - could be a product (tradename), region, brand, product line, or whole country |
| **Run Date (pns_run_date)** | The date when a forecast model was executed - not the target prediction date |
| **Yearmonth** | The target month for a forecast prediction |
| **Business FA** | Business Forecast Accuracy - primary metric for forecast quality |
| **SHAP** | SHapley Additive exPlanations - ML technique to explain feature importance (hidden from users) |
| **FVA** | Forecast Value Added - measures which forecasting step/method improves accuracy |
| **Opinion Line** | A specific forecasting method/step in the S&OP process (e.g., Statistical Forecast, Demand Planning, Consensus) |
| **Portkey** | AI gateway that routes LLM requests to different providers (Bedrock, Vertex AI, Azure OpenAI) |
| **Databricks** | Cloud data platform hosting the forecast data warehouse |
| **Structured Output Parser** | n8n component that enforces JSON schema on LLM outputs |

---

*Report generated for MAS (Multi-Agent System) academic presentation.*
*Source: PNS Demand Forecast AI Agent repository - n8n/brazil/workflows/*
