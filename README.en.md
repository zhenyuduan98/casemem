# CaseMem

[中文](README.md)

Auditable long-term memory for agents across scenarios. Support, travel planning, and expense review share one pipeline: isolate by business, promote only evidence-backed requirements, and recall them before the next task.

**Summary:** A reusable long-term memory layer for agents. Conversations and tool results are compressed per session, then promoted into rules only when there is evidence: an explicit lasting instruction, repetition across sessions, or a correction after a tool failure. Memories are isolated by `project_id`, so support preferences, trip constraints, and expense-review policy follow the same consolidation and recall flow without leaking across businesses. A new task recalls the relevant rules first, and the run keeps both an unmemorized draft and a rule-by-rule compliance check. Stack: Python, Agently, and Qwen (`qwen-plus`).

## Problem

Stuffing the whole history into the next request gives the model a transcript, not a rule it can follow. Support asks again for a preference already stated. A trip planner repeats a route a tool already rejected. Expense review asks about the same policy every time.

CaseMem consolidates between tasks. The model writes summaries, merges duplicates, and proposes search terms. The code stores files, filters by business, decides whether evidence is enough to promote a rule, and places the selected rules into the next request.

## One pipeline, several businesses

The simulated data already covers two businesses, not a single trip scenario.

| Business | project_id | Rules that can remain |
|---|---|---|
| Support / tickets | create one on intake, for example `support-agent` | Stated contact preferences, repeated handling requirements, corrections after tool failures |
| Trip planning | `travel-agent` | Nap window, a distant attraction occupying a full day, attaching the route-check result |
| Expense pre-review | `expense-agent` | Small transport receipts are pre-reviewed under the new policy, without asking every time |

A support event uses the same fields as the current materials: `project_id`, `session_id`, `event_id`, role, and text, plus `tool` and `status` when a tool fails. Switching businesses changes the event content. Consolidation, promotion, and recall stay the same.

The current file is `materials/simulated_long_conversation.jsonl`: 36 events in 3 sessions. The Beijing and Xi'an family trips belong to `travel-agent`. The taxi-receipt review belongs to `expense-agent`. Recalling a travel task filters out expense memories.

## What happens on a new task

```text
raw events
  -> compress each session into a summary and candidates
  -> promote only evidence-backed rules
  -> recall by the current business and the current task
  -> draft a reply and check each rule
```

Promotion has three gates:

- The user explicitly said the requirement stays in force
- The same requirement shows up in more than one session
- A tool failure produced a corrected practice

Candidates that miss these gates stay on a watch list and do not become long-term rules.

## Memory tiers

| Tier | What it stores | Across scenarios |
|---|---|---|
| Raw | Original dialogue and tool results | Support wording, route-check output, receipt review |
| Consolidated | One session's summary and candidates | How this case failed and how it was fixed |
| Semantic | Rules that still apply across sessions | Handling preferences, project constraints, review policy |
| Working | The memory pack for the current task | Rules and summaries actually used in this reply |

A background heartbeat compares event ids already processed. It consolidates only when new events exist, and skips the model call when there are none.

## Demo contrast

Travel task: "Take my 5-year-old daughter to Shanghai for two days. She wants Disneyland."

Without memory, both days are booked inside Disneyland and there is no nap window. With this business's memory, the system recalls the nap, morning priority for important sights, a full day for a distant attraction, and the requirement to attach a route check. Each rule is marked met, partial, or missed. The check reports that the nap is still absent from the schedule, and that "stay near Wangfujing" is a Beijing rule that does not apply in Shanghai. Rules keep their evidence, so compliance can be reviewed. Expense memories are not recalled for this task.

## Run

Requires Python 3.12 and an Alibaba Cloud Model Studio key in `DASHSCOPE_API_KEY`. Put the key in the environment or in a root `.env`. Do not commit it.

```bash
pip install -i https://mirrors.aliyun.com/pypi/simple/ agently python-dotenv
```

See `.env.example`:

```text
DASHSCOPE_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
DASHSCOPE_MODEL=qwen-plus
DASHSCOPE_API_KEY=your Model Studio API key
```

Smallest request:

```bash
python demo.py
```

Consolidation and recall:

```bash
python scripts/01_compress_session_memory.py
python scripts/02_promote_long_term_memory.py
python scripts/03_recall_new_session.py
python scripts/04_recall_process_memory.py
python scripts/05_extract_capability_memory.py
```

File memory, heartbeat, and the integrated comparison:

```bash
python scripts/01_raw_only_context.py
python scripts/06_heartbeat_trigger.py
python scripts/07_file_recall.py
python scripts/08_integrated_workspace_case/run.py
```

`06` runs collection, extraction, promotion, and working memory when new raw events exist. `08` clears the demo directory, rebuilds the four tiers, imports them into RecordStore, and prints a draft with memory and a draft without it. Output is written under `.demo_runs/`.

## Layout

```text
demo.py                     memory slots inside one request
materials/                  simulated dialogue and the current process event
scripts/01–05               compress, promote, recall, method cards
scripts/02_collect_raw.py   start of the file-based four tiers
scripts/06_heartbeat_trigger.py
scripts/08_integrated_workspace_case/
```

## Stack

Python 3.12, Agently 4.1, Qwen `qwen-plus` through the DashScope OpenAI-compatible API, local JSON/JSONL, and RecordStore.
