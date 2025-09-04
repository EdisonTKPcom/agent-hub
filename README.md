
# agent-hub
**Central registry & runner** for CrewAI workflows.

[![Python](https://img.shields.io/badge/python-3.11%2B-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](#license)
[![CI](https://img.shields.io/github/actions/workflow/status/your-org/agent-hub/ci.yml?branch=main)](../../actions)
[![CrewAI](https://img.shields.io/badge/CrewAI-ready-000)](https://github.com/joaomdmoura/crewai)

> YAML-first agent & task configs, **sequential** or **hierarchical** orchestration with a **manager agent/LLM**, plus CLI & REST API to trigger runs and collect artifacts.

---

## Table of Contents
- [Why agent-hub](#why-agent-hub)
- [Features](#features)
- [Architecture](#architecture)
- [Quickstart](#quickstart)
- [Configuration](#configuration)
  - [Agents YAML](#agents-yaml)
  - [Tasks YAML](#tasks-yaml)
- [Usage](#usage)
  - [CLI](#cli)
  - [REST API](#rest-api)
- [Project Layout](#project-layout)
- [Environment & Secrets](#environment--secrets)
- [Observability](#observability)
- [Testing & Quality](#testing--quality)
- [Docker (optional)](#docker-optional)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Why agent-hub
- 🗂️ **Central registry** for agent & task definitions (YAML), easy to reuse and version.
- 🧠 **Orchestrate your way**: sequential pipelines or hierarchical with a manager persona/LLM.
- 🔌 **Interfaces**: trigger via CLI or REST API; artifacts stored under `output/`.
- 🔭 **Transparent runs**: logs, transcripts, and optional tracing (Langfuse).

---

## Features
- **YAML configs** for agents (role/goal/backstory/tools/models) and tasks (description, expected_output, output_file).
- **Variable interpolation** in YAML (e.g., `{topic}`) passed at kickoff.
- **Orchestration modes**:
  - `Process.sequential` — linear, predictable pipelines.
  - `Process.hierarchical` — manager plans, delegates, and verifies with `manager_agent` or `manager_llm`.
- **Interfaces**:
  - CLI: `python -m agent_hub.main --topic "..."`.
  - REST: `POST /run` to kick off crews from other services.
- **Artifacts & Logs**: saved under `./output/`.
- **Optional**: search tools, code execution, document knowledge sources, Langfuse tracing.

---

## Architecture
```
+-------------------+         +-----------------------+
|   agents.yaml     |         |      tasks.yaml       |
|  (roles/goals)    |         | (desc, outputs, etc.) |
+---------+---------+         +-----------+-----------+
          \                             /
           \                           /
            v                         v
                +-----------------------------+
                |        agent-hub Core       |
                |  Crew builder (sequential/  |
                |  hierarchical + manager)    |
                +--------+--------------------+
                         |
                         v
                 +---------------+
                 | CrewAI Engine |
                 +-------+-------+
                         |
                         v
             +-----------------------+
             | output/ (artifacts)   |
             | logs, md, json, etc.  |
             +-----------------------+
```

---

## Quickstart

### 1) Setup
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

### 2) Minimal run (sequential or hierarchical)
Edit `config/agents.yaml` and `config/tasks.yaml` (see below), then:
```bash
python -m agent_hub.main --topic "AI Agents"
```

### 3) Start API
```bash
uvicorn agent_hub.api:app --reload --port 8000
# POST http://localhost:8000/run
# { "topic": "AI Agents" }
```

---

## Configuration

### Agents YAML
`config/agents.yaml`
```yaml
researcher:
  role: "{topic} Researcher"
  goal: "Find fresh, credible info on {topic}"
  backstory: "You dig fast and deep on {topic}."

writer:
  role: "{topic} Writer"
  goal: "Compose a clear, structured brief on {topic}"
  backstory: "You turn raw notes into polished prose."
```

### Tasks YAML
`config/tasks.yaml`
```yaml
research_task:
  description: "Research {topic}. Prioritize 2025 sources and include URLs."
  expected_output: "10 concise bullets with links."
  agent: researcher

writing_task:
  description: "Expand bullets into a 1-page Markdown brief."
  expected_output: "A brief in Markdown (no code fences)."
  agent: writer
  output_file: "output/brief.md"
```

> You can add variables like `{topic}` to either file and pass values via `kickoff(inputs={...})`.

---

## Usage

### CLI
```bash
python -m agent_hub.main --topic "AI Agents"
```
- Prints a run summary to stdout.
- Saves artifacts (e.g., `brief.md`) under `./output/`.

### REST API
`src/agent_hub/api.py` exposes:
- `POST /run` — Kick off a run with JSON body:
  ```json
  { "topic": "AI Agents" }
  ```
- Response:
  ```json
  { "ok": true, "summary": "..." }
  ```

---

## Project Layout
```
agent-hub/
├─ config/
│  ├─ agents.yaml
│  └─ tasks.yaml
├─ src/agent_hub/
│  ├─ crew.py         # Crew builder & process mode
│  ├─ main.py         # CLI entrypoint
│  └─ api.py          # FastAPI endpoints
├─ output/            # run artifacts & logs
├─ tests/
│  └─ test_flow.py
├─ .env.example
├─ requirements.txt
├─ pyproject.toml     # optional
├─ .gitignore
└─ LICENSE
```

**Minimal code samples**

`src/agent_hub/crew.py`
```python
from crewai import Agent, Task, Crew, Process
from crewai.project import CrewBase, agent, task, crew
from pathlib import Path
import yaml

def _load_yaml(path: str) -> dict:
    with open(path, "r") as f:
        return yaml.safe_load(f)

CONFIG_DIR = Path(__file__).resolve().parents[2] / "config"
AGENTS = _load_yaml(CONFIG_DIR / "agents.yaml")
TASKS = _load_yaml(CONFIG_DIR / "tasks.yaml")

@CrewBase
class AgentHubCrew:
    agents_config = AGENTS
    tasks_config = TASKS

    @agent
    def researcher(self) -> Agent:
        return Agent(config=self.agents_config["researcher"], verbose=True)

    @agent
    def writer(self) -> Agent:
        return Agent(config=self.agents_config["writer"], verbose=True)

    @agent
    def manager(self) -> Agent:
        return Agent(
            role="Project Manager",
            goal="Plan, delegate, and verify outputs for quality",
            backstory="Seasoned PM coordinating multi-agent work",
            allow_delegation=True,
            verbose=True,
        )

    @task
    def research_task(self) -> Task:
        return Task(config=self.tasks_config["research_task"])

    @task
    def writing_task(self) -> Task:
        return Task(config=self.tasks_config["writing_task"])

    @crew
    def app(self) -> Crew:
        # switch to Process.sequential for linear pipelines
        return Crew(
            agents=[self.researcher(), self.writer()],
            tasks=[self.research_task(), self.writing_task()],
            process=Process.hierarchical,
            manager_agent=self.manager(),
            verbose=True,
        )
```

`src/agent_hub/main.py`
```python
import argparse
from agent_hub.crew import AgentHubCrew

def main():
  parser = argparse.ArgumentParser()
  parser.add_argument("--topic", required=True, help="Topic to run through the crew")
  args = parser.parse_args()
  result = AgentHubCrew().app().kickoff(inputs={"topic": args.topic})
  print(result)

if __name__ == "__main__":
  main()
```

`src/agent_hub/api.py`
```python
from fastapi import FastAPI
from pydantic import BaseModel
from agent_hub.crew import AgentHubCrew

app = FastAPI(title="agent-hub")

class RunReq(BaseModel):
  topic: str

@app.post("/run")
def run(req: RunReq):
  res = AgentHubCrew().app().kickoff(inputs={"topic": req.topic})
  return {"ok": True, "summary": str(res)}
```

---

## Environment & Secrets
Copy `.env.example` → `.env` and set keys as needed:
```env
# Models (configure your provider for CrewAI)
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GROQ_API_KEY=
AZURE_OPENAI_API_KEY=
AZURE_OPENAI_ENDPOINT=

# Observability (optional)
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=https://cloud.langfuse.com

APP_ENV=dev
```

> Keep secrets out of version control. Use local `.env`, CI secrets, or a vault.

---

## Observability
- Logs & artifacts saved under `./output/`.
- Optional **Langfuse**: set `LANGFUSE_*` env vars to send traces (runs, generations, tool calls).

---

## Testing & Quality
```bash
pytest -q               # unit tests
ruff check .            # lint
ruff format .           # format
mypy src                # type-check
```

---

## Docker (optional)
**Dockerfile**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV PYTHONUNBUFFERED=1
CMD ["python", "-m", "agent_hub.main", "--topic", "AI Agents"]
```

**Build & run**
```bash
docker build -t agent-hub .
docker run --rm -it --env-file .env -v $(pwd)/output:/app/output agent-hub
```

---

## Roadmap
- [ ] Built-in tool presets (web search, code exec, file tools).
- [ ] Knowledge sources (local docs, URLs) per-agent.
- [ ] Manager **LLM** mode example and toggle.
- [ ] Run metadata DB (SQLite) and dashboard.
- [ ] Templates for common flows (research→brief, triage→fix plan).

---

## Contributing
1. Fork & create a feature branch.
2. Add tests for your change.
3. Run `ruff`, `mypy`, and `pytest`.
4. Open a PR with a clear description.

---

## License
MIT © Your Name / Organization
