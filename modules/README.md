# Agent Engineering (LangGraph)

ဒီပတ်မှာ production-grade AI agent တွေကို LangGraph နဲ့ တည်ဆောက်၊ စီမံခန့်ခွဲ၊ စစ်ဆေးနည်းတွေကို လေ့လာပါမယ်။

## ဒီပတ်မှာ ဘာသင်မလဲ

- Agent logic ကို `while` loop သုံးတဲ့ နည်းနဲ့မဟုတ်ဘဲ explicit graph အနေနဲ့ မကြာခဏ ဘာကြောင့် သတ်မှတ်သင့်တာလဲ
- StateGraph၊ nodes၊ edges၊ `compile()`၊ `invoke()`/`stream()` အသုံးချနည်း
- State schema တွေနဲ့ reducers က merge လား overwrite လား ဆုံးဖြတ်ပုံ
- Conditional routing၊ retry loops၊ recursion limit နဲ့ infinite loop တားဆီးနည်း
- Tool-calling agents — `bind_tools`, `ToolNode`, error handling နဲ့ permissions
- Checkpointing၊ thread_id၊ time travel နဲ့ long-term memory
- Human-in-the-loop approval gates နဲ့ `interrupt()`
- Multi-agent patterns (Supervisor, Swarm, Subgraphs) နဲ့ agent evaluation/observability

## Modules

- `01_langgraph_foundations/` — **LangGraph Foundations** — Graphs, nodes, state တွေရဲ့ အခြေခံသဘောတရားနဲ့ chain နဲ့ state machine ကွာခြားချက်ကို သင်ပါမယ်။
- `02_state_and_reducers/` — **State, Messages & Reducers** — TypedDict/Pydantic state schemas နဲ့ `add_messages` တို့လို reducer တွေက state update ကို ဘယ်လို ထိန်းချုပ်လဲ သင်ပါမယ်။
- `03_conditional_routing/` — **Conditional Routing, Loops & Termination** — Conditional edges၊ retry cycles၊ `recursion_limit` နဲ့ လုံခြုံတဲ့ termination conditions တွေကို လက်တွေ့ သင်ပါမယ်။
- `04_tool_calling_agent/` — **Tool-Calling Agents** — `bind_tools`, `ToolNode`, parallel tool calls၊ error contracts နဲ့ tool permission narrowing တွေကို သင်ပါမယ်။
- `05_persistence_checkpointing/` — **Persistence, Checkpointing & Time Travel** — Checkpointer backends (memory, sqlite, postgres)၊ thread_id နဲ့ checkpoint replay တွေကို သင်ပါမယ်။
- `06_human_in_the_loop/` — **Human-in-the-Loop Approval & Interrupts** — `interrupt()` approval gates၊ state inspect/edit၊ resume with `Command` နဲ့ audit trail တွေကို သင်ပါမယ်။
- `07_multi_agent_patterns/` — **Multi-Agent Patterns** — Supervisor routing၊ swarm handoffs၊ fan-out/fan-in နဲ့ subgraph state isolation တွေကို သင်ပါမယ်။
- `08_agent_evals_observability/` — **Agent Evaluation & Observability** — Outcome vs trajectory evals၊ Langfuse tracing နဲ့ CI regression gates တွေကို သင်ပါမယ်။

## ဒီပတ်ရဲ့ ရည်မှန်းချက်

ဒီပတ် ပြီးတဲ့အခါ သင်ဟာ —

- Agent logic ကို LangGraph StateGraph နဲ့ ဖွဲ့စည်းပြီး node/edge တွေကနေ ရှင်းလင်းတဲ့ control flow နဲ့ ရေးနိုင်မယ်
- State schema နဲ့ reducer သတ်မှတ်ပြီး message history တွေကို မှန်ကန်စွာ စီမံနိုင်မယ်
- Retry/refine loops ပါတဲ့ tool-calling agent တစ်ခုကို safety guardrails နဲ့ တည်ဆောက်နိုင်မယ်
- Checkpointing သုံးပြီး agent run တွေကို pause/resume လုပ်နိုင်ပြီး human approval ထည့်နိုင်မယ်
- Multi-agent system တစ်ခုကို ဒီဇိုင်းချပြီး tracing နဲ့ evals နဲ့ ဂုဏ်သတ္တိ စစ်ဆေးနိုင်မယ်

## လေ့လာရန် အစီအစဉ်

အောက်ပအတိုင်း အစဉ်လိုက် လေ့လာဖို့ အကြံပြုပါတယ် (၃ ရက် = ၇ ရက်):

1. **Day 1–2:** `01_langgraph_foundations/` — graph အခြေခံ သဘောတရား
2. **Day 2–3:** `02_state_and_reducers/` — state နဲ့ reducer စနစ်
3. **Day 3–4:** `03_conditional_routing/` — routing နဲ့ loops
4. **Day 4:** `04_tool_calling_agent/` — tool-calling agent တည်ဆောက်ခြင်း
5. **Day 5:** `05_persistence_checkpointing/` — checkpointing နဲ့ time travel
6. **Day 5–6:** `06_human_in_the_loop/` — approval workflow တွေ
7. **Day 6:** `07_multi_agent_patterns/` — multi-agent patterns
8. **Day 7:** `08_agent_evals_observability/` — evals နဲ့ observability၊ ပြီးရင် ဒီပတ်ရဲ့ project ကို အဆုံးသတ်

## Checkpoint

သင့်ကိုယ်သင် စစ်ဆေးကြည့်ပါ —

1. LangGraph ရဲ့ explicit graph model က `while` loop တစ်ခုထက် ဘာကြောင့် ပို maintainable ဖြစ်တာလဲ? ဥပမာ နှစ်ခု သုံးပြီး ရှင်းပြပါ။
2. `Annotated[list, add_messages]` reducer က state update တစ်ခုမှာ messages ကို append လား overwrite လား? ဘာကြောင့်လဲ။
3. Retry loop တစ်ခုက `recursion_limit` ကို ကျော်သွားရင် LangGraph က ဘယ်လို ဆက်လက်ဆောင်ရွက်မလဲ? ဘယ်လို ကာကွယ်မလဲ?
4. `interrupt()` နဲ့ checkpointing က ဘယ်လို ပေါင်းစပ်ပြီး human approval gate တစ်ခုကို ဖန်တီးပေးလဲ?
