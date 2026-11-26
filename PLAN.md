# Implementation Plan: Paper-to-Notebook Agent

## What we are building

You give the agent an arXiv link. It:

1. Downloads and reads the paper.
2. Finds the one core idea (the method).
3. Writes a small Jupyter notebook that shows that idea working on toy data.
4. Runs the notebook to check it works, and fixes it if it breaks.

The goal is also to learn ADK. Each phase below teaches **one new thing**, and the agent works at the end of every phase.

## How it will look at the end

```
You: "Make a notebook for arxiv.org/abs/1706.03762"
              │
              ▼
      ┌───────────────┐
      │  Coordinator  │  (the main agent, runs the steps in order)
      └───────────────┘
              │
   ┌──────────┼───────────────┬──────────────┐
   ▼          ▼               ▼              ▼
 Reader    Planner         Coder          Tester
 reads     picks the core  writes the     runs it and
 paper     method, plans   notebook       sends errors
           the demo                       back to Coder
              │
              ▼
     outputs/attention_is_all_you_need.ipynb
```

## Folder layout (final)

```
paper-to-notebook/
├── .env                      # your Gemini API key (never committed)
├── PLAN.md                   # this file
├── pyproject.toml
├── outputs/                  # notebooks the agent makes
└── src/paper_to_notebook/
    ├── __init__.py           # tells ADK where the agent is
    ├── agent.py              # root_agent lives here
    ├── tools.py              # our own Python tools
    ├── sub_agents/           # reader, planner, coder, tester
    └── skills/
        └── notebook_style/SKILL.md   # rules for how notebooks look
```

---

## Phase 0: Setup (about 30 min)

**Goal:** ADK is installed and a "hello" agent answers in the browser.

Steps:
1. Get a free Gemini API key from Google AI Studio.
2. Add ADK: `uv add google-adk`
3. Create `.env` with `GOOGLE_API_KEY=...` and add `.env` to `.gitignore`.
4. Write a tiny `agent.py` with one `Agent` (name, model, instruction). No tools yet.
5. Run `uv run adk web src` and chat with it in the browser.

**You learn:** how an ADK agent is defined, and how `adk web` finds it (`root_agent`).

**Done when:** the agent replies to "hi" in the ADK web UI.

---

## Phase 1: First tool, fetch the paper (about 1 hour)

**Goal:** the agent can read a real paper.

Steps:
1. In `tools.py`, write `fetch_paper(arxiv_id)`. It downloads the PDF and returns the text (use `pypdf` or the arXiv API).
2. Give the tool to the agent: `tools=[fetch_paper]`.
3. Change the instruction to: "Read the paper and explain its core method in 5 bullet points."

**You learn:** a tool is just a Python function. ADK reads its **name, type hints and docstring** to tell the model what it does. A good docstring matters a lot.

**Done when:** you paste an arXiv link and get a correct summary. In the web UI you can see the tool call happen.

---

## Phase 2: Write and run a notebook (about 2 hours)

**Goal:** the agent makes a real `.ipynb` file that runs.

Steps:
1. Add `save_notebook(title, cells)` in `tools.py`. It uses `nbformat` to write a notebook into `outputs/`.
2. Add `run_notebook(path)`. It runs the notebook (with `nbclient`) and returns "ok" or the error message.
3. Tell the agent: write the notebook, run it, and if it fails, fix it and try again (max 3 tries).

**You learn:** how tools return results the model can react to, and how an agent loops on its own until the job is done.

**Done when:** a small paper turns into a notebook that runs without errors.

---

## Phase 3: Connect an MCP server (about 1 hour)

**Goal:** swap our homemade paper fetcher for a ready-made arXiv MCP server.

Steps:
1. Use an existing arXiv MCP server (for example `arxiv-mcp-server`, started with `uvx`).
2. Connect it in `agent.py` with ADK's `MCPToolset`.
3. Now the agent can also **search** arXiv, not just fetch one paper.

**You learn:** what MCP is (a standard plug that lets any agent use tools someone else wrote), how ADK starts the server and lists its tools, and how MCP tools compare with our own tools.

**Done when:** "find the original ResNet paper and make a notebook" works without giving an ID.

---

## Phase 4: Add a skill (about 1 hour)

**Goal:** notebooks always follow the same clean style.

Steps:
1. Write `skills/notebook_style/SKILL.md` with your rules, for example:
   - Sections: Title → The idea in plain words → Toy data → The method → Results → What we skipped.
   - Use only numpy, matplotlib and (if needed) torch.
   - Must run in under 1 minute on a laptop.
   - Short comments, no walls of text.
2. Load the skill into the agent. Check how the current ADK version supports skills; if it has no built-in support, load the file into the instruction ourselves.

**You learn:** a skill is packaged know-how. You write your expertise once, and the agent uses it every time, without stuffing it all into the main prompt.

**Done when:** two notebooks from different papers have the same structure.

---

## Phase 5: Split into sub-agents (about 2–3 hours)

**Goal:** one agent per job, so each one is simpler and better at its job.

Steps:
1. Create four small agents in `sub_agents/`: **Reader**, **Planner**, **Coder**, **Tester**.
2. Join Reader → Planner → Coder in a `SequentialAgent` (runs in order).
3. Put Coder + Tester in a `LoopAgent` so they repeat until the notebook runs.
4. Pass results between agents with `output_key` (for example, Reader saves `paper_summary`, and Planner reads it).

**You learn:** how agents work together, the difference between a "workflow" agent (fixed order) and an LLM agent (decides by itself), and how data moves between them.

**Done when:** the web UI shows each sub-agent taking its turn.

---

## Phase 6: State and callbacks (about 1–2 hours)

**Goal:** add memory and safety checks.

Steps:
1. Use session **state** to remember papers already done, so asking twice reuses the old notebook.
2. Add a **callback** before the code runs that blocks dangerous code (deleting files, network calls).
3. Add a callback that logs how long each step took and how many tokens it used.

**You learn:** how ADK remembers things, and how to control or watch an agent from the outside.

**Done when:** a repeat request is instant, and a notebook with `os.remove` gets blocked.

---

## Phase 7 (optional): Polish

- A simple command: `uv run paper-to-notebook 1706.03762`.
- A few tests with 3 known papers to see if quality drops after changes (a tiny eval).
- A README with an example notebook.

---

## Things to watch out for

| Risk | What we do about it |
|---|---|
| Papers are long and cost many tokens | Send only the abstract and method section to the Planner and Coder. |
| Generated code could do harmful things | Run notebooks in a separate process with a timeout, plus the Phase 6 safety callback. |
| Some papers need big models or GPUs | The Planner must shrink the idea into a toy version. Its "What we skipped" section says what was left out. |
| The fix loop never ends | Cap retries at 3. |

## Rough timeline

About **10–12 hours** in total. One or two phases per evening works well.

## Packages we will add

`google-adk`, `pypdf`, `nbformat`, `nbclient`, `ipykernel`, `numpy`, `matplotlib`
