# Ask-the-Web Agent — Build-It-Yourself Roadmap

A self-paced roadmap to rebuild the tool-calling web-search agent **from scratch**, in
your own UV + Python project, **CLI-first**. Use the cohort's `ask_the_web_agent_solution.ipynb`
/ `chainlit_app.py` only as a reference when you're truly stuck.

> **Goal:** understand *why* each piece exists by building it yourself, smallest → largest.
> A local LLM (Ollama) reasons about a question, decides which **tool** to call
> (`calculate` or `web_search`), runs it, and answers — the ReAct loop.

---

## 0. Prerequisites & Project Setup

**Install Ollama and pull the model:**
```bash
ollama pull llama3.2:3b
ollama list          # confirm the model is present
```

**Create the project with UV:**
```bash
uv init ask-the-web-agent
cd ask-the-web-agent
uv add langchain langchain-core langchain-ollama langgraph ddgs
# (add later, only when you reach those milestones)
# uv add chainlit
```

**Suggested layout (build it up as you go):**
```
ask-the-web-agent/
├── pyproject.toml
├── agent/
│   ├── __init__.py
│   ├── tools.py        # your @tool functions
│   ├── core.py         # llm + create_react_agent + answer extraction
│   └── cli.py          # the CLI entry point  ← FIRST deliverable
└── app.py              # Chainlit UI          ← LAST, optional
```

**Definition of done for this milestone:** `uv run python -c "import langchain_ollama; print('ok')"` prints `ok`.

---

## 1. Talk to the LLM (no tools yet)

Prove you can get a response from the local model.

- Create a `ChatOllama(model="llama3.2:3b")`.
- Call `.invoke("Say hello in one sentence.")` and print the result.

**Concepts:** the chat model object; messages in / message out. Local inference is slow —
expect seconds per call. That's normal.

**Done when:** running your script prints a real sentence from the model.

> 💡 Tip: set a `timeout` (e.g. 120s) when you create the model so a stuck call doesn't hang forever.

---

## 2. Write ONE tool: `calculate`

Create `agent/tools.py`.

- Write `calculate(expression: str) -> str` decorated with `@tool` from `langchain_core.tools`.
- Give it a **clear docstring** — e.g. *"Evaluate a simple math expression like '5*5' or '10+3'."*
- Guard the input: only allow digits and `+ - * / ( ) .` before evaluating. Return a friendly
  error string on bad input instead of raising.

**Concepts:** the **docstring IS the instruction** — it's what the LLM reads to decide *when*
and *how* to call your tool. The type hints define the arguments it must produce.

**Done when:** `calculate.invoke({"expression": "25*4"})` returns `"100"`.

---

## 3. Wire up the agent (one tool)

Create `agent/core.py`.

- `llm = ChatOllama(model="llama3.2:3b")`
- Write a short `system_prompt` telling the assistant it has a `calculate` tool for math.
- `agent = create_react_agent(llm, [calculate], prompt=system_prompt)`
  (from `langgraph.prebuilt`).
- Invoke it: `agent.invoke({"messages": [HumanMessage(content="what is 25*4?")]})`
- Print the **full** returned message list and read it.

**Concepts:** the **ReAct loop** — the model reasons, emits a tool call, the tool runs, the
result feeds back, the model answers. Seeing the raw messages here is the whole point: you'll
spot the tool-call message and the final answer message.

**Done when:** you can see in the message trace that the model *called* `calculate` rather than
doing the math itself.

> 💡 Limit steps (e.g. `max_iterations` / recursion limit) so a bad run can't loop forever.

---

## 4. Add the second tool: `web_search`

Back in `agent/tools.py`.

- Write `web_search(query: str) -> str` using `ddgs`:
  ```python
  with DDGS() as ddgs:
      results = list(ddgs.text(query, max_results=5))
  ```
- Return a short formatted list of titles + URLs (or "No results found.").
- Add it to the agent's tool list: `create_react_agent(llm, [calculate, web_search], ...)`
  and update the system prompt to mention both tools.

**Concepts:** **tool selection** — with two tools, the model must choose. Ask a math question
(should pick `calculate`) and a current-events question (should pick `web_search`) and confirm
it routes correctly.

**Done when:** "What's 12*9?" uses `calculate` and "Who won the latest F1 race?" uses `web_search`.

---

## 5. Extract the clean final answer

The raw message list contains intermediate tool-call JSON — not what you'd show a user.

- Write a helper `get_final_answer(messages) -> str` that walks the messages **in reverse**,
  finds the last `AIMessage` with real text content, and **skips** content that looks like raw
  tool-call schema (e.g. starts with `{` and contains `"name"` / `"parameters"`).
- Fall back to a friendly message if nothing usable is found.

**Concepts:** the agent's output is a *transcript*, not a single answer — you post-process it.

**Done when:** your function returns only the human-readable answer string.

---

## 6. CLI — your first real deliverable

Create `agent/cli.py`.

- Read the user's question (start simple: `input()` in a loop, or `sys.argv`).
- Pass it to the agent, run `get_final_answer`, print the result.
- Add a `[project.scripts]` entry in `pyproject.toml` so `uv run ask-web "your question"` works.

**Suggested loop shape:**
```
print a greeting
while True:
    q = input("> ")
    if q in {"exit", "quit"}: break
    answer = run_agent(q)
    print(answer)
```

**Done when:** `uv run ask-web` gives you an interactive prompt that answers math *and*
web-lookup questions from the terminal.

> 🎯 This is the milestone to stop at first. A working CLI agent = you understand the whole system.

---

## 7. (Optional) Wrap it in a Chainlit chat UI

Only after the CLI works. The agent logic is identical — you're just swapping the interface.

- `uv add chainlit`
- Create `app.py` with:
  - `@cl.on_chat_start` → send a welcome message.
  - `@cl.on_message` → `await agent.ainvoke(...)` (note the **async** variant), run
    `get_final_answer`, `await cl.Message(content=...).send()`.
- Run it: `uv run chainlit run app.py`.

**Concepts:** same agent, async invocation, event-handler-based UI. Compare your result to the
cohort's `chainlit_app.py` — they should be structurally the same.

**Done when:** you can chat with your agent in the browser.

---

## Stretch goals (make it yours)

- Add a third tool (e.g. a unit converter, or fetch + summarize a URL's content).
- Stream tokens to the CLI/UI instead of waiting for the full answer.
- Add citations: have `web_search` return URLs and make the agent cite them.
- Swap the model (`gemma3`, a bigger llama) and compare tool-calling reliability.
- Add `max_iterations` / timeout handling and a "thinking…" indicator.

---

## When you're stuck

1. Re-read your tool **docstrings** — vague docstrings = bad tool selection.
2. Print the **full message list** to see what the agent actually did.
3. Confirm Ollama is alive: `ollama run llama3.2:3b` and type a prompt.
4. Only then peek at `ask_the_web_agent_solution.ipynb` for the specific piece you're missing.

Happy building. 🚀