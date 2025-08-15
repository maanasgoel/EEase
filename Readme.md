# EEase: The AI‑Powered Engineering Workbench

EEase (Electrical Engineering Assistant Suite) is a local, agentic workbench for electrical engineers and students. It gives you a conversational UI that orchestrates specialized tools to:

- Run LTSpice simulations end‑to‑end from plain English (generate netlists, run headless, plot signals)
- Solve textbook‑style problems using Retrieval‑Augmented Generation (RAG) grounded in your PDFs/TXT, with citations
- Generate Markdown lab reports from your data/plots using an optional template

Unlike a simple LLM chat, EEase executes real tools locally, saves artifacts with absolute paths, and cites sources. You keep full control of data and workflows.

---

## Table of Contents
- [What You Get](#what-you-get)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Running EEase](#running-eease)
- [Modes and Capabilities](#modes-and-capabilities)
  - [PyLTSpice Agent (simulation)](#pyltspice-agent-simulation)
  - [Problem Solver (RAG)](#problem-solver-rag)
  - [Lab Report Generator](#lab-report-generator)
- [End‑to‑End Examples](#end-to-end-examples)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Roadmap (short)](#roadmap-short)
- [License](#license)
- [Credits](#credits)

---

## What You Get

- **Agentic automation**: A LangGraph agent that calls MCP tools to do real work (not just chat)
- **Glass‑box answers**: RAG citations to file paths and page numbers in your PDFs
- **Artifacts**: Netlists, logs, RAW data, and plots saved with absolute paths
- **Local first**: Runs on your machine; you control inputs/outputs and persistence

---

## Architecture

- **Frontend**: Next.js app with CopilotKit chat UI (`app/`)
- **Agent backend**: LangGraph dev server (Python) orchestrating workflows (`agent/sample_agent/`)
- **MCP servers** (stdio):
  - `agent/ltspice_server.py` — LTSpice automation
  - `agent/rag/rag_server.py` — RAG tools over FAISS index
  - `agent/lab/lab_server.py` — Lab report generation
- **Knowledge base**: PDFs/TXT under `agent/knowledge_base/`; FAISS index persisted in `agent/knowledge_base/db_faiss/`

Everything runs locally; the OpenAI API is used for embeddings and LLM calls.

---

## Prerequisites

- Python 3.10–3.12 and **Poetry**
- Node.js 18+ and **pnpm**
- An OpenAI API key in a repo‑root `.env` file:
  ```
  OPENAI_API_KEY=sk-...
  ```
- **LTSpice** installed.
  - On macOS/Linux, install **Wine** to run LTSpice headless: `brew install wine-stable`
- Place any PDFs/TXT you want indexed by RAG into `agent/knowledge_base/`.

---

## Setup

1) Clone
```bash
git clone https://github.com/your-username/EEase.git
cd EEase
```

2) Backend deps (Poetry)
```bash
cd agent
poetry install
```

3) Frontend deps (pnpm)
```bash
cd ..
pnpm install
```

4) Environment
- Create `.env` at repo root with `OPENAI_API_KEY` (see Prerequisites).
- Ensure LTSpice and Wine are installed (macOS/Linux), and LTSpice is runnable headless.

---

## Running EEase

Open two terminals.

- Terminal A (agent):
```bash
cd /Users/maanasgoel/Documents/Cursor/EEase/agent
poetry run langgraph dev --host localhost --port 8123 --no-browser | cat
```

- Terminal B (frontend):
```bash
cd /Users/maanasgoel/Documents/Cursor/EEase
pnpm run dev-frontend | cat
```

Open the app in your browser (printed dev URL). Select the desired Mode (top‑right badge).

---

## Modes and Capabilities

### PyLTSpice Agent (simulation)
MCP server: `agent/ltspice_server.py`

- Natural language → netlist → headless LTSpice run → list traces → plot PNGs
- Robust plotting: auto time/frequency axis, AC magnitude handling, path self‑healing
- Returns absolute paths for netlists, logs, RAW files, and plot images

Typical flow:
1. Generate netlist and create a simulation session
2. Run headless simulation (Wine on macOS/Linux)
3. List available traces
4. Plot requested traces to PNG

### Problem Solver (RAG)
MCP server: `agent/rag/rag_server.py`

- Build a FAISS index from your PDFs/TXT (persisted on disk), then answer questions grounded in your docs
- Tools:
  - `rag_status` — readiness (in‑memory/on‑disk), kb/db paths, has_api_key
  - `rag_ingest` — build the FAISS index and save to `agent/knowledge_base/db_faiss/`
  - `rag_load` — load the on‑disk index into memory
  - `rag_answer(question)` — answer with citations (file path + page)
  - `rag_stats` — per‑file chunk counts (diagnostics)

Notes:
- The workflow preflights (status → load/ingest) and can auto‑load the index.
- You can also run offline ingestion:
  ```bash
  cd agent
  poetry run python rag/ingest.py
  ```

### Lab Report Generator
MCP server: `agent/lab/lab_server.py`

- Generate a Markdown report from:
  - `topic` (title), `objective` (goal)
  - `file_paths` (text/csv/logs embedded as code blocks; images embedded by absolute paths)
  - Optional `template` with placeholders: `{{TOPIC}}`, `{{OBJECTIVE}}`, `{{DATA}}`, `{{ANALYSIS}}`, `{{CONCLUSION}}`
- Tool:
  - `lab_status` — quick health check (API key, CWD)
  - `generate_lab_report(...)` — returns `report_markdown`

---

## End‑to‑End Examples

### 1) PyLTSpice: RC Transient
Prompt (PyLTSpice mode):
> Simulate an RC circuit with R=1k, C=1uF. Drive with a 5V pulse width 2ms; run a 5ms transient. Show the netlist first.

Expected:
- Tool returns netlist and a `session_dir` (absolute path)
- “Run simulation” → success + absolute paths: `circuit.net`, `circuit.log`, `circuit.raw`
- “What can I plot?” → list traces
- “Plot V(out)” → absolute `plot_*.png` path

### 2) Problem Solver: RLC Impedance with Citations
In Problem Solver mode:
- “call rag_status” → If the index isn’t in memory but exists on disk, run “call rag_load”. If no index exists, “call rag_ingest”.
- Ask: “Derive impedance of series RLC at frequency ω; cite page numbers.”
- Expect: `Z = R + j(ωL − 1/(ωC))` plus short definitions and citations to your PDFs with page anchors.

Diagnostics:
- “List the files you indexed and the number of chunks per file.” → returns per‑file chunk counts via `rag_stats`.

### 3) Lab Report: Markdown from Data + Template
In Lab Report mode, provide a structured JSON prompt:
```json
{
  "topic": "RC Transient Experiment",
  "objective": "Measure and verify the RC time constant using step response.",
  "file_paths": [
    "/absolute/path/scope.csv",
    "/absolute/path/plot.png"
  ],
  "template": "# {{TOPIC}}\n\n## Objective\n{{OBJECTIVE}}\n\n## Data & Plots\n{{DATA}}\n\n## Analysis\n{{ANALYSIS}}\n\n## Conclusion\n{{CONCLUSION}}\n"
}
```
Expect: JSON with `report_markdown` (ready to save as `.md`).

---

## Troubleshooting

- “RAG token limit” or generic answers
  - You probably didn’t hit tools. Ensure backend is running and mode is **Problem Solver (RAG)**.
  - “call rag_status” should return JSON. If not, restart backend and verify `.env` contains `OPENAI_API_KEY`.

- RAG `rag_load`/retriever errors
  - The code uses `FAISS.as_retriever(k=8)` (stable). If you experimented with custom retrievers, revert.

- LTSpice run fails
  - The tool returns readable errors and `circuit.log` path. Ask: “Read the simulation log.”

- No tools available / server fails to start
  - Run `poetry install` in `agent/` and watch backend logs for import errors.

---

## FAQ

- Does this upload my PDFs to a server?
  - No. Documents are processed locally; embeddings and LLM calls are made to OpenAI. Index files are stored under `agent/knowledge_base/db_faiss/`.

- Can I add more PDFs later?
  - Yes. Place them under `agent/knowledge_base/` and run “call rag_ingest” (or the offline ingest script).

- Where are LTSpice artifacts stored?
  - Session folders are created under a timestamped directory and returned as absolute paths (e.g., netlists, logs, RAW, plots).

---

## Roadmap (short)
- Small CSV tools: `csv_describe`, `csv_plot` to augment Lab Reports
- “Save report” tool to write Markdown to `reports/` directory
- Additional EE utilities (impedance/time‑constant calculators, citation peeking)

---

## License
MIT (see LICENSE if included in this repository).

---

## Credits
- **Author**: Maanas Goel
- **Special Thanks**: Qian Zhang, Dr. Le Xie
- EEase uses LangGraph, CopilotKit, FAISS, and OpenAI APIs; LTSpice is by Analog Devices.
