# Larkspur Disruption Agent — Repo Overview

## What it is

A hands-on exercise where you build a **multi-turn tool-calling agent** for Larkspur Airlines. Claude handles disrupted passengers by calling a suite of nine tools (look up bookings, check policy, rebook, issue vouchers, escalate). You write `agent.py`; everything else is given.

---

## Repo structure

```mermaid
graph TD
    A[agent.py<br/>✏️ YOUR file] -->|calls| B[support/tools.py<br/>9 tool implementations]
    A -->|via| C[support/session.py<br/>new_session → client + tracer]
    B -->|reads/writes| D[support/mock_backend.py<br/>bookings, flights, policy data]
    D -->|reads| E[data/americas/<br/>bookings.json, flights.csv,<br/>disruption_policy.json]

    F[run.py] -->|drives| A
    G[verify.py<br/>the gate] -->|reads| H[.workshop/last_trace.json]
    C -->|writes| H

    I[support/mcp_server.py] -.->|optional: Build 2.2| A
```

---

## The agent loop (and its bugs)

```mermaid
sequenceDiagram
    participant User
    participant run_agent
    participant Claude API
    participant tools

    User->>run_agent: PNR + last_name + message
    run_agent->>Claude API: messages=[user_msg], tools=[9 schemas]
    Claude API-->>run_agent: response (stop_reason=tool_use)

    loop while stop_reason == tool_use AND turns < 8
        Note over run_agent: ✏️ BUG 1: appends text_of(response)<br/>instead of response.content<br/>→ strips tool_use blocks → API error
        run_agent->>run_agent: messages.append(assistant: text_of(response))
        run_agent->>tools: tool_results(response)
        tools-->>run_agent: [{tool_result: ...}]
        run_agent->>Claude API: messages=[..., tool_results]
        Claude API-->>run_agent: next response
        run_agent->>run_agent: answer = text_of(response)  ← captures PREVIOUS turn's text
        Note over run_agent: ✏️ BUG 2: answer is set BEFORE<br/>the new API call fires,<br/>so the last turn's text is never captured
    end

    run_agent-->>User: return answer  ← wrong: should be text_of(response)
```

---

## The six editable places (`grep -n '✏' agent.py`)

| Build | Step | Mark | What you write |
|-------|------|------|----------------|
| 1 | 1.2 | `run_agent()` | Fix the tool loop — append `response.content`, return `text_of(response)` |
| 1 | 1.3 | `build_tools()` | Fix tool descriptions so Claude routes correctly |
| 2 | 2.1 | `EXTRA_TOOLS` + `LOCAL_TOOLS` | Add your own tool schema + implementation |
| 2 | 2.2 | `tool_list()` | Route some tools over MCP instead |
| 3 | 3.1 | `evals/cases.json` | Write eval cases (via `/case` command) |
| 4 | 4.1 | `TONE_ADDENDUM` | Add intelligence/tone lane to system prompt |

---

## Current bugs to diagnose

**Bug 1 — `run_agent()` line 71** (`messages.append` inside loop):

```python
# Wrong — strips tool_use blocks, API rejects the conversation
messages.append({"role": "assistant", "content": text_of(response)})

# Should be
messages.append({"role": "assistant", "content": response.content})
```

**Bug 2 — `run_agent()` return value** (line 80):

```python
# Wrong — captures text before the next API call, so last turn is lost
answer = text_of(response)   # set here, THEN new response overwrites it
response = client.messages.create(...)
...
return answer  # one turn behind

# Should be
return text_of(response)  # after the loop, captures the final turn
```

**Bug 3 — `search_alternatives` description** (line 130):

```python
"description": "search",  # 6 characters — Claude has no routing signal
```

The gate for step 1.3 will fail here. Fix: write a full description of when Claude should call this tool and what it needs to already know before calling it.

---

## Build order and gates

```mermaid
flowchart LR
    B1["Build 1\n1.2 loop\n1.3 tools\n1.4 all 5 shapes"] -->
    B2["Build 2\n2.1 your tool\n2.2 over MCP"] -->
    B3["Build 3\n3.1 eval cases"] -->
    B4["Build 4\n4.1 tone lane"]

    style B1 fill:#ffd,stroke:#aaa
    style B2 fill:#dff,stroke:#aaa
    style B3 fill:#dfd,stroke:#aaa
    style B4 fill:#fdf,stroke:#aaa
```

Each step has a gate: `python3 verify.py <step>` — it checks the **wire** (what actually went to the API), not your code shape. Any implementation that produces the right behavior passes.

---

## The nine tools

| Tool | Kind | Description quality |
|------|------|---------------------|
| `lookup_booking` | Read | Good — both args explained |
| `get_flight_status` | Read | Good — date format specified |
| `search_alternatives` | Read | **Bad** — description is just `"search"` |
| `check_policy` | Read | Good — explains what it re-derives |
| `hold_seat` | Write | Minimal but adequate |
| `confirm_rebooking` | Write | Good — explains token requirement |
| `issue_voucher` | Write | Good — explains auto-approve threshold |
| `escalate_to_human` | Write | Good — frames escalation as correct outcome |
| `send_confirmation` | Write | Minimal |

---

## Diagnostic commands

| Command | Purpose |
|---------|---------|
| `python3 run.py K7PQ2M --trace` | See every API turn, tool calls, and results |
| `python3 run.py --show-tools` | What Claude actually reads about each tool |
| `python3 run.py --tool-tax` | Token cost of your tool list |
| `python3 run.py --all --trace` | Run all 5 Stage-1 shapes |
| `python3 verify.py 1.2` | Check if your loop passes the gate |
| `python3 verify.py` | Status board — what's banked |

**Start here:** `python3 run.py K7PQ2M --trace` — read the wire. The trace shows Bug 1 and Bug 2 before any code tour does.
