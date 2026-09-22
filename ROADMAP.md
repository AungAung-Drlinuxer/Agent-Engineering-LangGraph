# 8-Module Study Roadmap

ဒီ roadmap က `Agent Engineering with LangGraph` course ရဲ့ module ရှစ်ခုစလုံးအတွက် လေ့လာမည့် အစီအစဉ် ဖြစ်ပါတယ်။ တစ် module ချင်းစီမှာ ဘာ cover လုပ်မလဲ၊ ပြီးရင် လုပ်နိုင်ရမည့်အရာ၊ နဲ့ self-test checkpoint မေးခွန်းတွေ ပါဝင်ပါတယ်။

## Module 01 — LangGraph Foundations — Graphs, Nodes, State

**Cover လုပ်မည့်အရာ —** Agent logic ကို plain while-loop နဲ့ရေးတာရဲ့ အားနည်းချက် (state ရှုပ်ထွေးမှု၊ retry/fan-out မရေးလွယ်ခြင်း) နဲ့ explicit graph က ဘာကြောင့် ပိုကောင်းတယ်ဆိုတာ။ `StateGraph`, nodes, edges, `START`/`END`, `compile()`, `invoke()`, `stream()` တို့ရဲ့ အခြေခံ။ Chain တစ်ခုနဲ့ agent graph တစ်ခု ကွာခြားချက်။

**ဒီ module ပြီးရင် လုပ်နိုင်ရမည့်အရာ —**
- State type တစ်ခု သတ်မှတပြီး nodes နဲ့ edges ချိတ်တဲ့ graph တစ်ခုကို compile လုပ်နိုင်ခြင်း
- `invoke()` နဲ့ `stream()` နဲ့ run ကြည့်ပြီး step-by-step output ရနိုင်ခြင်း

**Checkpoint self-test —**
1. `StateGraph` မှာ node တစ်ခုက return လုပ်တဲ့ dict က graph state နဲ့ ဘယ်လို ဆက်စပ်လဲ?
2. `invoke()` နဲ့ `stream()` ကွာခြားချက်က ဘာလဲ? ဘယ်အချိန် `stream()` ကို သုံးသင့်လဲ?
3. Chain တစ်ခုနဲ့ agent graph တစ်ခုရဲ့ အဓိက ကွာခြားချက် ၃ ချက်ပြောပါ။

## Module 02 — State, Messages & Reducers

**Cover လုပ်မည့်အရာ —** `TypedDict` နဲ့ Pydantic state schemas၊ `Annotated` reducers (ဥပမာ `add_messages`)၊ channel semantics၊ partial state updates၊ reducer တစ်ခုက update တစ်ခုကို merge လုပ်မလား overwrite လုပ်မလား ဘယ်လို ဆုံးဖြတ်လဲဆိုတာ။

**ဒီ module ပြီးရင် လုပ်နိုင်ရမည့်အရာ —**
- Message history accumulate လုပ်တဲ့ state schema ကို reducer နဲ့ တည်ဆောက်နိုင်ခြင်း
- Reducer မပါတဲ့ field နဲ့ reducer ပါတဲ့ field ရဲ့ update အလုပ်လုပ်ပုံ ကွာခြားချက်ကို predict လုပ်နိုင်ခြင်း

**Checkpoint self-test —**
1. `Annotated[list, add_messages]` ကို သုံးတဲ့ field မှာ node တစ်ခုက message အသစ် return ပြန်ရင် ဘာဖြစ်မလဲ?
2. Reducer မသတ်မှတထားတဲ့ field က update တွေကို ဘယ်လို ဆက်စပ်လဲ?
3. Pydantic state က TypedDict state ထက် ဘယ်အခါ ပိုအသုံးဝင်လဲ?

## Module 03 — Conditional Routing, Loops & Termination

**Cover လုပ်မည့်အရာ —** Conditional edges နဲ့ routing functions၊ retry/refine loops အတွက် cycles၊ `recursion_limit`၊ termination conditions၊ infinite loop တွေကနေ ကာကွယ်တဲ့ guardrails ဒီဇိုင်း၊ loop-based agent design အခြေခံ။

**ဒီ module ပြီးရင် လုပ်နိုင်ရမည့်အရာ —**
- Quality check → refine → re-check ပုံစံက graph တစ်ခုကို conditional edges နဲ့ တည်ဆောက်နိုင်ခြင်း
- Loop ပိတ်မဆုံးတာကို ကာကွယ်ဖို့ အနည်းဆုံး guardrail နှစ်ခု ထည့်နိုင်ခြင်း

**Checkpoint self-test —**
1. Conditional edge တစ်ခုက routing function ရဲ့ return value ကို ဘယ်လို အသုံးချလဲ?
2. `recursion_limit` ကရောက်ရင် graph မှာ ဘာဖြစ်လဲ? ဘယ်လို handle လုပ်သင့်လဲ?
3. Retry loop တစ်ခုမှာ အနည်းဆုံး ဘယ် guardrails တွေ ထည့်သင့်လဲ?

## Module 04 — Tool-Calling Agents

**Cover လုပ်မည့်အရာ —** `bind_tools` နဲ့ Pydantic/JSON schema tools၊ `ToolNode`၊ parallel tool calls၊ tool error contracts နဲ့ retries၊ tool arguments validate လုပ်ခြင်း၊ tool permissions ကို ကျဉ်းချင်းခြင်း။

**ဒီ module ပြီးရင် လုပ်နိုင်ရမည့်အရာ —**
- Pydantic schema နဲ့ tool တွေ သတ်မှတပြီး agent တစ်ကောင်ကို `ToolNode` နဲ့ တည်ဆောက်နိုင်ခြင်း
- Tool failure ဖြစ်ခဲ့ရင် agent ကို graceful ပြန်ဖြေစေတဲ့ error handling ရေးနိုင်ခြင်း

**Checkpoint self-test —**
1. `bind_tools` က model ကို ဘယ် information ပို့လဲ? Tool function ကို ဘယ်အချိန် run လဲ?
2. Tool တစ်ခု exception ထုတ်လိုက်ရင် `ToolNode` မှာ default အလုပ်လုပ်ပုံက ဘာလဲ?
3. Tool permission narrowing ဆိုတာ ဘာလဲ? ဘာကြောင့် လိုအပ်လဲ?

## Module 05 — Persistence, Checkpointing & Time Travel

**Cover လုပ်မည့်အရာ —** Checkpointer backends (Memory, Sqlite, Postgres)၊ `thread_id` semantics၊ failure နောက် resume လုပ်ခြင်း၊ past checkpoint က replay လုပ်ခြင်း၊ long-term store နဲ့ cross-thread memory။

**ဒီ module ပြီးရင် လုပ်နိုင်ရမည့်အရာ —**
- Postgres checkpointer နဲ့ durable agent တစ်ခုကို deploy အလွယ်သဏ္ဌာန် တည်ဆောက်နိုင်ခြင်း
- တူညီတဲ့ `thread_id` နဲ့ ပြန်ခေါ်တဲ့အခါ conversation state ပြန်ရပုံကို ရှင်းပြနိုင်ခြင်း

**Checkpoint self-test —**
1. `thread_id` က checkpoint တွေကို ဘယ်လို ဖွဲ့စည်းလဲ? အခြား thread ရဲ့ state နဲ့ ရောက်နိုင်လဲ?
2. Process crash ဖြစ်ပြီးနောက် Postgres checkpointer က resume လုပ်ပုံက ဘယ်လိုအလုပ်လုပ်လဲ?
3. Time travel (past checkpoint က replay) ကို ဘယ် use case တွေမှာ သုံးသင့်လဲ?

## Module 06 — Human-in-the-Loop Approval & Interrupts

**Cover လုပ်မည့်အရာ —** `interrupt()` နဲ့ approval gates၊ state ကို ဆက်မလုပ်ခင် inspect/edit လုပ်ခြင်း၊ `Command` နဲ့ resume လုပ်ခြင်း၊ timeout နဲ့ cancellation handling၊ approvals ရဲ့ audit trail။

**ဒီ module ပြီးရင် လုပ်နိုင်ရမည့်အရာ —**
- အန္တရာယ်ရှိတဲ့ action တွေကို pre-execution မှာ approve စေတဲ့ agent တစ်ခုကို တည်ဆောက်နိုင်ခြင်း
- Interrupt state ကို edit လုပ်ပြီး `Command` နဲ့ resume လုပ်နိုင်ခြင်း၊ approval record တွေကို log လုပ်ခြင်း

**Checkpoint self-test —**
1. `interrupt()` က graph run ကို ဘယ်လိုရပ်တန့်စေလဲ? State က ဘယ်နေရာမှာ လုံခြုံစွာ သိမ်းထားလဲ?
2. Resume လုပ်ချင်ရင် `Command` ထဲ ဘာတွေ ထည့်ပေးလို့ရလဲ?
3. Approval timeout ကို ဘယ်လို design လုပ်သင့်လဲ? Timeout ကြားတဲ့အခါ ဘာဖြစ်သင့်လဲ?

## Module 07 — Multi-Agent Patterns

**Cover လုပ်မည့်အရာ —** Agent တစ်ကောင် မလုံလောက်တဲ့အခါ။ Supervisor routing၊ handoff/swarm patterns၊ parallel fan-out နဲ့ fan-in၊ reusable components အဖြစ် subgraphs၊ state isolation နဲ့ sharing။

**ဒီ module ပြီးရင် လုပ်နိုင်ရမည့်အရာ —**
- Supervisor pattern နဲ့ specialized agents သုံးကောင်ကို လမ်းညွှန်စေတဲ့ system တစ်ခု တည်ဆောက်နိုင်ခြင်း
- Subgraph နဲ့ state mapping လုပ်ပြီး သီးခြားရှိတဲ့ agent တစ်ခုကို parent graph ထဲ ပေါင်းစည်းနိုင်ခြင်း

**Checkpoint self-test —**
1. Supervisor pattern နဲ့ swarm/handoff pattern ရဲ့ ကွာခြားချက်က ဘာလဲ? ဘယ်အခါ ဘယ်ဟာ ပိုသင့်တော်လဲ?
2. Fan-out/fan-in ကို LangGraph မှာ ဘယ်လို implement လုပ်လဲ?
3. Subgraph ရဲ့ state က parent state နဲ့ ဘယ်လို isolate / map လုပ်လဲ?

## Module 08 — Agent Evaluation & Observability

**Cover လုပ်မည့်အရာ —** Outcome evals vs trajectory evals၊ agents အတွက် dataset design၊ Langfuse နဲ့ node/tool call တိုင်းကို tracing လုပ်ခြင်း၊ CI ထဲ regression gates၊ run တစ်ခုချင်းစီရဲ့ cost နဲ့ latency တိုင်းတာခြင်း။

**ဒီ module ပြီးရင် လုပ်နိုင်ရမည့်အရာ —**
- Agent run တွေကို Langfuse မှာ trace လုပ်ပြီး per-node latency/cost ကို ကြည့်နိုင်ခြင်း
- Eval dataset တစ်ခု တည်ဆောက်ပြီး CI မှာ regression gate တစ်ခု ထည့်နိုင်ခြင်း

**Checkpoint self-test —**
1. Outcome eval တစ်ခုက trajectory eval တစ်ခုထက် ဘယ်အခါ ပိုသင့်တော်လဲ?
2. Agent eval dataset တစ်ခုမှာ ဘယ် fields တွေ ထည့်သင့်လဲ?
3. CI regression gate တစ်ခုက fail ဖြစ်သွားရင် နောက်အဆင့်မှာ ဘာတွေ လုပ်သင့်လဲ?

## နေ့စဉ် လေ့လာပုံ အကြံပြုချက်

- **တစ်နေ့ကို 1.5-2 နာရီ** သီးသန့်ထားပါ — စာဖတ်ချိန်၊ code run ချိန်၊ exercise ဖြေချိန် ခွဲသုံးပါ။
- Module တစ်ခုကို **3-4 ရက်** သုံးပါ — ရက် ၂ ရက် explanation၊ ရက် ၁-၂ ရက် exercise နဲ့ mini agent။
- အားလုံးပေါင်းရင် **4 ပတ်ခန့်** ကုန်မယ်လို့ မျှော်မှန်းပါ — module တစ်ခုချင်းစီကို တိတ်တဆိုင်မ ပြေးဖို့ထက် လက်တွေ့လုပ်ဖို့က ပိုအရေးကြီးပါတယ်။
- ရက်သတ္တပတ်တိုင်းမှာ အရင်ပတ်က module တစ်ခုရဲ့ checkpoint questions တွေကို ပြန်ဖြေကြည့်ပါ။

## Capstone — production agent

Course အားလုံးပြီးဆုံးရင် **IT-ops agent** တစ်ကောင်ကို တည်ဆောက်ပါ —

- **Typed tools** — Pydantic schema နဲ့ IT-ops actions (log query, service status check, restart request) — Module 04
- **Postgres checkpointer** — durable execution၊ failure နောက် resume — Module 05
- **HITL approval** — အန္တရာယ်ရှိတဲ့ action (ဥပမာ service restart) မတိုင်ခင် `interrupt()` approval gate — Module 06
- **Tracing + eval gate** — Langfuse tracing၊ outcome eval dataset၊ CI regression gate — Modules 07-08
- **Cost/latency budget** — run တစ်ခုချင်း cost နဲ့ latency တိုင်းတာပြီး သတ်မှတထားတဲ့ budget ထဲ နေအောင် tune လုပ်ခြင်း — Module 08

ဒီ capstone က ရှစ်မော်ဂျူးစလုံးရဲ့ အဓိက skills တွေကို တစ်နေရာတည်း ပေါင်းစပ်ပြတဲ့ အမြင့်ဆုံး အဆင့်ဖြစ်ပါတယ်။
