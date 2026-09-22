# Agent Engineering with LangGraph — 8-Module Self-Study Course

ဒီ repository က production-grade AI agents ကို LangGraph နဲ့ ဆောက်တဲ့ self-study course ရဲ့ အထိပ်မှာ ရှိတဲ့ curriculum ဖြစ်ပါတယ်။

> **မှတ်ချက် —** ဒီ course က ကိုယ်ပိုင်ရေးသားတဲ့ original self-study curriculum တစ်ခုဖြစ်ပြီး၊ technical အချက်အလက်အားလုံးကို **official open documentation** (LangGraph docs, Langfuse docs, Python stdlib docs) မှာတစ်ခုတည်း ကိုးကားပြီး ရေးသားထားတာ ဖြစ်ပါတယ်။ မည်သည့် commercial course ၏ သင်ခန်းစာ သို့မဟုတ် စာသားကိုမှ မကူးယူထားပါ။ ဒီ course က ရှေ့မှာသင်ပြီးသွားတဲ့ Python fundamentals course နဲ့ production AI-engineering track (FastAPI/Celery/pgvector/RAG/evals/deployment) နှစ်ခုရဲ့ ဆက်လက်အနေဖြင့် ရေးခဲ့တာလည်း ဖြစ်ပါတယ်။

## ဒီ course မှာ ဘာသင်မလဲ

- Agent logic ကို while-loop သာမက explicit **graph** အနေနဲ့ ဘာကြောင့် model လုပ်သင့်တယ်ဆိုတာ — StateGraph, nodes, edges, `compile()`, `invoke()`/`stream()`
- **State design** — TypedDict/Pydantic schemas, `Annotated` reducers, `add_messages`, partial state updates ရဲ့ merge-or-overwrite စည်းမျဉ်းတွေ
- **Conditional routing နဲ့ loops** — retry/refine cycles, `recursion_limit`, infinite loop တွေကို ကာကွယ်တဲ့ guardrails
- **Tool-calling agents** — `bind_tools`, `ToolNode`, parallel tool calls, tool error contracts, argument validation, tool permission narrowing
- **Persistence** — Memory/Sqlite/Postgres checkpointer backends, `thread_id` semantics, failure နောက် resume လုပ်ခြင်း၊ past checkpoint က replay
- **Human-in-the-loop** — `interrupt()` approval gates, state inspect/edit, `Command` နဲ့ resume, timeout, approval audit trail
- **Multi-agent patterns** — Supervisor, Swarm/handoff, parallel fan-out/fan-in, subgraphs နဲ့ state isolation
- **Evals & observability** — outcome vs trajectory evals, dataset design, Langfuse tracing, CI regression gates, cost/latency တိုင်းတာခြင်း

## Modules

- `01_langgraph_foundations/` — **LangGraph Foundations — Graphs, Nodes, State** — Agent logic ကို explicit graph နဲ့ model လုပ်ခြင်း၊ StateGraph, nodes, edges, `compile()`, `invoke`/`stream` အခြေခံ။
- `02_state_and_reducers/` — **State, Messages & Reducers** — State schema ဒီဇိုင်း၊ reducers နဲ့ merge/overwrite စည်းမျဉ်းတွေကို နားလည်ခြင်း။
- `03_conditional_routing/` — **Conditional Routing, Loops & Termination** — Conditional edges, retry loops, `recursion_limit` နဲ့ safe termination။
- `04_tool_calling_agent/` — **Tool-Calling Agents** — LLM ကို tools ချိတ်ပေးခြင်း၊ parallel calls, error handling နဲ့ tool permissions။
- `05_persistence_checkpointing/` — **Persistence, Checkpointing & Time Travel** — Checkpointer backends, `thread_id`, resume/replay နဲ့ long-term memory store။
- `06_human_in_the_loop/` — **Human-in-the-Loop Approval & Interrupts** — `interrupt()`, state edit, `Command` resume, approval audit trail။
- `07_multi_agent_patterns/` — **Multi-Agent Patterns** — Supervisor, Swarm, subgraphs နဲ့ state isolation/sharing။
- `08_agent_evals_observability/` — **Agent Evaluation & Observability** — Outcome/trajectory evals, Langfuse tracing, CI regression gates, cost/latency measurement။

## Folder ဖွဲ့စည်းပုံ

ဒီ course ရဲ့ module တိုင်းမှာ ဖိုင်လေးခု တစ်ချောင်းစီ ပါဝင်ပါတယ် —

- `README.md` — module overview နဲ့ learning objectives
- `explanation.md` — သဘောတရားရှင်းလင်းချက် (Burmese) + ဥပမာ code များ
- `exercise.md` — လက်တွေ့စာမေးပွဲများ
- `solution.md` — အဖြေများ နဲ့ ရှင်းလင်းချက်များ

## လေ့လာပုံ နည်းလမ်း

1. ဒီ top-level README ကို အရင်ဖတ်ပြီး roadmap (ROADMAP.md) နဲ့ ကြည့်ပါ။
2. တစ် module ချင်းစီအတွက် `explanation.md` ကို ဖတ်ပါ — ကုဒ်ဥပမာတွေကို ကိုယ်တိုင် run ကြည့်ပါ။
3. `exercise.md` ထဲက စာမေးပွဲတွေကို မကြည့်ဘဲ ကိုယ်တိုင်ဖြေပါ။
4. `solution.md` နဲ့ နှိုင်းယှဉ်ပြီး၊ ကွာခြားချက်ရှိရင် documentation ကို ပြန်ဖတ်ပါ။
5. Module တိုင်းရဲ့ mini agent လေးကို ကိုယ်ပိုင် topic နဲ့ တည်ဆောက်ကြည့်ပါ — ဒါက အရေးကြီးဆုံး အဆင့်ဖြစ်ပါတယ်။

## လိုအပ်ချက်များ (Prerequisites)

ဒီ course က beginner level မဟုတ်ဘဲ **beginner-to-intermediate** level ဖြစ်ပါတယ်။ အောက်ပါအချက်တွေ ရှိနေဖို့ လိုပါတယ် —

- **Python fundamentals** — functions, classes, decorators, `typing` (TypedDict, Annotated, Optional)
- **Typed models** — Pydantic BaseModel နဲ့ validation အခြေခံ
- **LLM API basics** — chat completions, messages, tool calling ကို API တစ်ခုမှတစ်ခုချင်း ခေါ်ပုံ
- **RAG concepts** — embedding, vector store, retrieval pipeline အခြေခံ (ရှေ့ course ထဲက)
- Python 3.9+ နဲ့ `langgraph` package ကို install လုပ်နိုင်တဲ့ environment

## ကိုးကား

Technical အချက်အလက်အားလုံးကို အောက်ပါ official open documentation တွေက ကိုးကားပြီး ကိုယ်ပိုင်ရေးသားထားပါတယ် —

- LangGraph documentation: https://langchain-ai.github.io/langgraph/
- LangGraph how-to guides: https://langchain-ai.github.io/langgraph/how-tos/
- Langfuse documentation: https://langfuse.com/docs
- Python typing documentation: https://docs.python.org/3/library/typing.html
- Pydantic documentation: https://docs.pydantic.dev/
