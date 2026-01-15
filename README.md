# 🌊 Float Chat - Argo Float AI Query System

An intelligent conversational AI system for querying and analyzing Argo oceanographic float data using multi-agent architecture powered by Google's Generative AI (Gemini).

## 🏗️ Architecture Overview

Float Chat uses a **multi-agent system** with three specialized agents working together:

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Query                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MAIN AGENT                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  1. Relevance Check (irrelevant/simple/data)            │    │
│  │  2. Query Decomposition → Filter Agent + SQL Agent      │    │
│  │  3. Orchestration & Response Generation                 │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌──────────────────────┐         ┌──────────────────────┐
│    FILTER AGENT      │         │     SQL AGENT        │
│  ┌────────────────┐  │         │  ┌────────────────┐  │
│  │ Coordinate     │  │         │  │ Natural Lang   │  │
│  │ Filtering      │  │   ───►  │  │ to SQL         │  │
│  │ Date Filtering │  │  IDs    │  │ Conversion     │  │
│  │ Parameter      │  │         │  │ Data Fetch     │  │
│  │ Filtering      │  │         │  │ JSON Format    │  │
│  └────────────────┘  │         │  └────────────────┘  │
└──────────────────────┘         └──────────────────────┘
```

---

## 🤖 Agent Descriptions

### 1. Main Agent (`Main_Agent.py`)

The **orchestrator** that coordinates the entire query workflow.

**Responsibilities:**
- **Relevance Classification**: Classifies queries as:
  - `irrelevant` - Not related to Argo floats
  - `simple` - Answerable without data access (e.g., "What are Argo floats?")
  - `data` - Requires SQL/filter agents for actual data retrieval
- **Query Decomposition**: Splits complex queries into instructions for Filter Agent and SQL Agent
- **Decision Making**: Determines if user only needs float IDs or actual measurement data
- **Workflow Management**: Uses LangGraph to orchestrate the agent pipeline

**Workflow:**
```
check_relevance → decompose_query → execute_queries → END
```

---

### 2. Filter Agent (`filter_agent.py`)

An intelligent **metadata filtering agent** that identifies relevant Argo floats based on various criteria.

**Key Features:**
- Uses LangGraph for stateful workflow management
- Supports query decomposition for complex multi-part queries (OR/AND conditions)
- Built-in geographic knowledge (Bay of Bengal, Arabian Sea, Indian Ocean coordinates)

**Available Filter Tools:**

| Tool | Description | Parameters |
|------|-------------|------------|
| `filter_by_coordinates` | Filter by lat/lon boundaries | `lat_min`, `lat_max`, `lon_min`, `lon_max`, `k` |
| `filter_by_date_range` | Filter by launch date | `start_date`, `end_date`, `k` |
| `filter_by_parameter_value` | Filter by measurement values | `parameter`, `operator`, `value`, `k` |
| `find_extreme_values` | Find min/max parameter values | `parameter`, `extreme_type`, `k` |
| `filter_by_platform_type` | Filter by float type | `platform_type`, `k` |
| `filter_by_negation` | Exclude floats matching criteria | `filter_type`, `k`, `**kwargs` |
| `combine_filters` | Intersect multiple filter results | `float_lists` |

**Supported Parameters:**
- Temperature (`temp_min`, `temp_max`, `temp_avg`)
- Salinity/PSAL (`psal_min`, `psal_max`, `psal_avg`)
- Pressure/Depth (`pres_min`, `pres_max`, `pres_avg`)
- Dissolved Oxygen (`doxy_min`, `doxy_max`, `doxy_avg`)
- Chlorophyll-a, BBP700, Nitrate, pH, Turbidity, CDOM

**Example Queries:**
```python
# Simple coordinate filter
"Find floats in the Bay of Bengal"

# Parameter filtering
"Find floats with temperature greater than 25°C"

# Complex multi-condition
"Get floats in Arabian Sea with minimum temperature above 20"

# Negation queries
"Find floats NOT in the Indian Ocean"

# Extreme value queries
"Find the deepest float" (max pressure)
"Find the coldest float" (min temperature)
```

---

### 3. SQL Agent (`sql_agent.py`)

Converts natural language queries into **PostgreSQL queries** and retrieves actual measurement data.

**Responsibilities:**
- Takes filtered float IDs from Filter Agent
- Generates SQL queries using Gemini LLM
- Executes queries against PostgreSQL database
- Formats results as structured JSON grouped by filter categories

**Database Schema (`argo_profiles` table):**

| Column | Type | Description |
|--------|------|-------------|
| `float_id` | VARCHAR(20) | Float identifier |
| `cycle_number` | INT | Measurement cycle number |
| `juld` | TIMESTAMP | Julian date of observation |
| `latitude/longitude` | DOUBLE PRECISION | Position coordinates |
| `pressure/pressure_adjusted` | REAL[] | Depth profile |
| `temperature/temperature_adjusted` | REAL[] | Temperature profile |
| `salinity/salinity_adjusted` | REAL[] | Salinity profile |
| `doxy/doxy_adjusted` | REAL[] | Dissolved oxygen profile |
| `chla/chla_adjusted` | REAL[] | Chlorophyll-a profile |
| `bbp700/bbp700_adjusted` | REAL[] | Backscatter profile |

**Output Format:**
```json
{
  "bay_of_bengal": {
    "float_id_1": {
      "1": {
        "juld": "2020-01-15",
        "latitude": 15.5,
        "longitude": 88.2,
        "temperature": [28.5, 27.2, 25.1, ...],
        "pressure": [5, 10, 20, ...]
      }
    }
  }
}
```

---

## 📁 Project Structure

```
Float Chat/
├── AGENTS_AND_BACKEND/
│   ├── Main_Agent.py        # Main orchestrator agent
│   ├── filter_agent.py      # Metadata filtering agent (614 lines)
│   ├── sql_agent.py         # SQL generation & execution agent
│   ├── sql_setup.py         # Database schema & data import
│   ├── sort_json.py         # Utility for sorting cycle data
│   ├── meta_data.json       # Float metadata (~1.2MB)
│   ├── requirements.txt     # Python dependencies
│   └── .env                 # Environment variables (API keys)
├── FRONTEND/                # Next.js frontend application
└── float-chat/              # Alternative frontend
```

---

## 🚀 Getting Started

### Prerequisites
- Python 3.9+
- PostgreSQL database
- Google AI API key (Gemini)

### 1. Install Dependencies

```bash
cd AGENTS_AND_BACKEND
pip install -r requirements.txt
```

### 2. Environment Configuration

Create/edit `.env` file:
```env
GOOGLE_API_KEY=your_google_api_key_here
```

Get your API key from [Google AI Studio](https://makersuite.google.com/app/apikey)

### 3. Database Setup

Start PostgreSQL and create the database:
```bash
# Create database and user
createdb argo_db
createuser argo_user -P  # password: argo_pass
```

Run the setup script to create tables and import data:
```bash
python sql_setup.py
```

### 4. Run the Agent

```python
from Main_Agent import run_agent_query

# Example queries
result = run_agent_query("Compare temperature of floats in Bay of Bengal and Arabian Sea")
result = run_agent_query("Find the deepest float in the Indian Ocean")
result = run_agent_query("Show me salinity data for float 1902677 in 2020")
```

---

## 📊 Data Structure

### Float Metadata Schema

```json
{
  "float_id": {
    "platform_number": "string",
    "wmo_inst_type": "string",
    "project_name": "string",
    "pi_name": "string",
    "data_centre": "string",
    "launch_info": {
      "date": "YYYYMMDD",
      "latitude": -90 to 90,
      "longitude": -180 to 180,
      "platform_type": "APEX/NOVA/ARVOR/etc",
      "float_serial_no": "string"
    },
    "temp_max/min/avg": float,
    "psal_max/min/avg": float,
    "pres_max/min/avg": float,
    "doxy_max/min/avg": float,
    "status": "active/inactive",
    "cycle": int
  }
}
```

---

## 🔧 Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `langchain` | 0.1.20 | LLM orchestration |
| `langchain-google-genai` | 1.0.3 | Google Gemini integration |
| `langgraph` | 0.0.69 | Stateful agent workflows |
| `google-generativeai` | 0.5.4 | Direct Gemini API access |
| `psycopg2-binary` | 2.9.9 | PostgreSQL connectivity |
| `xarray` | 2024.1.1 | NetCDF data processing |
| `pydantic` | 2.6.4 | Data validation |
| `Flask` | 3.0.2 | Web API framework |

---

## 🌍 Pre-configured Geographic Regions

| Region | Latitude Range | Longitude Range |
|--------|----------------|-----------------|
| Bay of Bengal | 5° to 22° N | 80° to 95° E |
| Arabian Sea | 8° to 25° N | 50° to 75° E |
| Indian Ocean | 30° S to 30° N | 30° to 120° E |

---

## 📝 Example Queries

```python
# Geographic queries
"Find all floats in the Bay of Bengal"
"List floats between latitude 10 and 20"

# Temporal queries
"Find floats launched between 2015 and 2020"
"Show floats deployed after 2018"

# Parameter-based queries
"Find floats with temperature above 25°C"
"Get floats with pressure greater than 500 dbar"

# Comparative queries
"Compare temperature of floats in Bay of Bengal and Arabian Sea"
"List one float from each major ocean basin"

# Extreme value queries
"Find the hottest float"
"Which float reached the maximum depth?"

# Complex queries
"Get floats in Arabian Sea with salinity above 35 and temperature below 20"
```

---

## 📄 License

This project is for oceanographic research and data analysis purposes.

---

## 🙏 Acknowledgments

- [Argo Program](https://argo.ucsd.edu/) - Global array of ocean profiling floats
- [Google Generative AI](https://ai.google.dev/) - Gemini LLM
- [LangChain](https://www.langchain.com/) - LLM application framework
