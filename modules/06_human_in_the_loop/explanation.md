# Human-in-the-Loop Approval & Interrupts — ရှင်းလင်းချက်

ဒီသင်ခန်းစာမှာ LangGraph ရဲ့ human-in-the-loop အခြေခံ pattern ၄ ခုကို အဆင့်ဆင့် လေ့လာပါမယ် — (၁) `interrupt()` နဲ့ approval gate၊ (၂) state ကို inspect/edit လုပ်ခြင်း၊ (၃) `Command` နဲ့ resume လုပ်ခြင်း၊ (၄) timeout/cancellation နဲ့ audit trail နဲ့ escalation mapping။

## ၁ — interrupt() နဲ့ Approval Gate

### ဘာကို ဆိုလိုတာလဲ

`interrupt()` ဆိုတာ LangGraph က provide လုပ်တဲ့ function တစ်ခုပါ။ Graph လည်ပတ်နေစဉ်မှာ node တစ်ခုက ဒီ function ကို ခေါ်လိုက်ရင် run က အလိုအလျောက် ရပ်သွားပြီး "interrupted" အနေအထားမှာ checkpoint နဲ့အတူ သိမ်းထားပါတယ်။ လူက action တစ်ခုကို အတည်ပြုမှ state ကို resume လုပ်လို့ရပါတယ်။

### ဘာကြောင့် လဲ

Agent တွေဟာ တခါတရံမှာ ပြန်လှန်ဖျက်လို့မရတဲ့ action (ငွေပေးချေမှု၊ email ပို့၊ production database ပြောင်းလဲမှု) တွေကို လုပ်ဖို့ ကြိုးစားနိုင်ပါတယ်။ LLM output က မှန်ကန်မှုကို အာမခံနိုင်စွာ မသက်သက်တဲ့အတွက် အရေးကြီးတဲ့ ဆုံးဖြတ်ချက်တွေမှာ လူသူက တဆင့် စစ်ဆေးပေးတာက စိတ်ချရမှုနဲ့ တာဝန်ခံမှု ရှိစေပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

၁။ Graph ကို checkpointer တစ်ခု (ဥပမာ `MemorySaver`) နဲ့ compile လုပ်ပါ — interrupt က persistence မှာ မူတည်လို့ပါ။
၂။ အန္တရာယ်ရှိတဲ့ node ထဲမှာ `interrupt()` ကို ခေါ်ပြီး စစ်ဆေးဖို့ data (ဥပမာ action plan) ကို argument အနေနဲ့ ပေးပါ။
၃။ Run က `__interrupt__` key ပါတဲ့ state နဲ့ ပြန်လာပါတယ်။
၄။ လူက အတည်ပြုပြီးရင် `Command(resume=...)` နဲ့ invoke ပြန်လုပ်ပါ — node က `interrupt()` ရပ်နေတဲ့နေရာကနေ ဆက်လည်ပတ်ပါမယ်။

### ဥပမာ

```python
# Requires: pip install langgraph
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt
from typing import TypedDict

class State(TypedDict):
    query: str
    action: str

def propose_action(state: State) -> dict:
    # In a real agent this could be an LLM-produced plan
    return {"action": f"Delete file: /data/{state['query']}.csv"}

def human_gate(state: State) -> dict:
    # Pause here and show the proposed action to a human
    decision = interrupt({"action": state["action"]})
    print("Human decision:", decision)
    return {}

builder = StateGraph(State)
builder.add_node("propose", propose_action)
builder.add_node("gate", human_gate)
builder.add_edge(START, "propose")
builder.add_edge("propose", "gate")
builder.add_edge("gate", END)

checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "demo-1"}}
result = graph.invoke({"query": "sales"}, config)
print(result)
# Expected output: the run stops at the gate; result contains
# "__interrupt__" with the payload {"action": "Delete file: /data/sales.csv"}
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Production system တွေမှာ "dry-run ပြီးမှ approve" ပုံစံက အန္တရာယ်ကို သိသိသာသာ လျှော့ချပေးပါတယ်။ Gate တစ်ခုတည်ဆောက်တာက node တစ်ခုထဲမှာ `interrupt()` တစ်ကြောင်း ထည့်တာလောက်ပဲ ရှုပ်ထွေးမှု သက်သက်နည်းပါတယ်။

## ၂ — Interrupt ခဏရပ်ချိန်မှာ State ကို Inspect နဲ့ Edit လုပ်ခြင်း

### ဘာကို ဆိုလိုတာလဲ

Run တစ်ခုက interrupt ဖြစ်နေတဲ့အခါ graph က ဆက်မလည်ပတ်ပါဘူး — ဒါပေမဲ့ state က checkpointer ထဲမှာ ရှိနေဆဲဖြစ်ပါတယ်။ ကျွန်တော်တို့က `graph.get_state(config)` နဲ့ အခုထိက state ကို ဖတ်နိုင်ပြီး၊ `graph.update_state(config, ...)` နဲ့ တန်ဖိုးတွေကို ပြင်နိုင်ပါတယ်။

### ဘာကြောင့် လဲ

Approval မလုပ်ခင်မှာ reviewer က action ရဲ့ အသေးစိတ်ကို ကြည့်ဖို့ လိုပါတယ်။ တခါတရံမှာ LLM က မှားနေတဲ့ argument တွေကို လူက တိုက်ရိုက်ပြင်ပြီးမှ approve လုပ်တာက လုပ်ငန်းစွမ်းအင် ပိုကောင်းပါတယ် — reject ပြီး အလုံးစုံ ပြန် run တာထက် မြန်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

`get_state(config)` က `StateSnapshot` ပြန်ပါတယ် — `.values` က state dict ဖြစ်ပြီး `.next` က နောက်ထပ် run ဖို့ စောင့်နေတဲ့ node တွေပါ။ `update_state(config, {"key": "new_value"})` က checkpoint ထဲက state ကို ပြင်ဆင်ပေးပြီး resume လုပ်တဲ့အခါ node တွေက ပြင်ပြီးတဲ့ တန်ဖိုးတွေကိုပဲ မြင်ပါလိမ့်မယ်။

### ဥပမာ

```python
# Continuing from the previous example's graph and config
snapshot = graph.get_state(config)
print("Current action:", snapshot.values["action"])
print("Waiting at node:", snapshot.next)

# A reviewer edits the path before approving
graph.update_state(config, {"action": "Delete file: /data/sales_OLD.csv"})

updated = graph.get_state(config)
print("After edit:", updated.values["action"])
# Expected output:
# Current action: Delete file: /data/sales.csv
# Waiting at node: ('gate',)
# After edit: Delete file: /data/sales_OLD.csv
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Review UI တွေမှာ ဒီ pattern က အသုံးဝင်ပါတယ် — "propose → လူက ကြည့်ပြီး ဖောင်ထဲမှာ တည်းဖြတ် → approve" flow တွေက LangGraph ရဲ့ inspect/edit API အပေါ် တိုက်ရိုက် မူတည်ပါတယ်။ ပြင်တဲ့ အခါကြားမှုကိုလည်း update တစ်ခုချင်းစီက checkpoint history ထဲမှာ မှတ်တမ်းတင်စေပါတယ်။

## ၃ — Command နဲ့ Resume / Reject လုပ်ခြင်း

### ဘာကို ဆိုလိုတာလဲ

`Command` ဆိုတာ LangGraph object တစ်ခုဖြစ်ပြီး `resume` value တစ်ခုပါပါတယ်။ Graph ကို `invoke(Command(resume=...), config)` လို့ ခေါ်လိုက်ရင် ရပ်နေတဲ့ `interrupt()` နေရာကနေ ဆက်လည်ပတ်ပြီး၊ resume value ကို `interrupt()` ရဲ့ return value အနေနဲ့ ရပါတယ်။

### ဘာကြောင့် လဲ

Workflow က "approve / reject / feedback ပြင်ဆင်" ဆိုတဲ့ ရွေးချယ်မှုတွေကို လက်ခံဖို့ လိုပါတယ်။ Plain dict ထပ် invoke လုပ်တာနဲ့ မတူဘဲ `Command(resume=...)` က "ဒီ run ကို ဆက်ချင်တယ်၊ ဒီအဖြေနဲ့" ဆိုတဲ့ ရည်ရွယ်ချက်ကို တိကျစွာ ပြောပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

၁။ ပထမ invoke က interrupt ဖြစ်ပြီး ပြန်လာပါတယ်။
၂။ Resume ချင်ရင် `graph.invoke(Command(resume={"approved": True}), config)` လုပ်ပါ — တူညီတဲ့ `thread_id` ကို သုံးရပါမယ်။
၃။ Reject လုပ်ချင်ရင် resume value ထဲမှာ flag တစ်ခု ပြန်ပေးပြီး node logic က END ဆီသွားအောင် ရေးပါ။
၄။ State အသစ် တစ်ခုလုံးကို ပြန်စချင်ရင် `thread_id` အသစ်နဲ့ invoke လုပ်ပါ။

### ဥပမာ

```python
from langgraph.types import Command

def human_gate(state: State) -> dict:
    decision = interrupt({"action": state["action"]})
    if decision["approved"]:
        return {"action": state["action"] + " [EXECUTED]"}
    return {"action": state["action"] + " [REJECTED]"}

# ... build and compile the graph as before ...

# First run pauses at the gate (see example 1)
graph.invoke({"query": "sales"}, config)

# Human approves; the gate node receives the decision
result = graph.invoke(Command(resume={"approved": True}), config)
print(result["action"])
# Expected output: Delete file: /data/sales.csv [EXECUTED]

# With a rejection the same node would print:
# Delete file: /data/sales.csv [REJECTED]
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

`thread_id` တစ်ခုချင်းစီက approval စောင့်နေတဲ့ run တစ်ခုစီကို ကိုယ်စားပြုပါတယ်။ ဒါကြောင့် web app တစ်ခုက တစ်ယောက်ချင်း run တွေကို ခွဲခြားပြီး notification ပို့တာ၊ approval လက်ခံတာတွေကို တိုက်ရိုက် ချိတ်ဆက်နိုင်ပါတယ်။

## ၄ — Timeout, Cancellation, Audit Trail နဲ့ Escalation

### ဘာကို ဆိုလိုတာလဲ

Timeout ဆိုတာ approval တစ်ခုက သတ်မှတ်ချိန်အတွင်း မရှိလာရင် လုပ်ဆောင်ချက်တွေ ရွေးရတာပါ — ဥပမာ အလိုအလျောက် reject လုပ်တာ သို့မဟုတ် manager tier တစ်ခုကို escalate လုပ်တာပါ။ Audit trail က approval တိုင်းရဲ့ ဘယ်သူ၊ ဘယ်အချိန်၊ ဘာကြောင့် ဆုံးဖြတ်လဲဆိုတာ မှတ်တမ်းတင်တာပါ။

### ဘာကြောင့် လဲ

Production မှာ approval တွေက အချိန်မရွေး လာမှာ မဟုတ်ပါဘူး။ Pending run တွေက အမြဲတမ်း စောင့်မနေနိုင်လို့ timeout policy လိုပါတယ်။ Compliance အတွက်လည်း "ဘယ်သူ့ကို approve ချင်းလဲ" ဆိုတာ နောက်ပိုင်း စစ်ဆေးနိုင်ဖို့ မှတ်တမ်း လိုအပ်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

Timeout ကိ်ု အလွယ်ဆုံး လက်တွေ့လုပ်နည်းက — interrupt မဖြစ်ခင် node ထဲမှာ အချိန်ကို state ထဲ သိမ်းပြီး၊ pending run တွေကို scheduled job တစ်ခုက `get_state` နဲ့ စစ်ပြီး သက်တမ်းကျရင် `Command(resume={"approved": False, "reason": "timeout"})` နဲ့ အလိုအလျောက် reject လုပ်တာပါ။ Audit trail အတွက် state ထဲ `approvals` list တစ်ခု ထည့်ပြီး တစ်ခြောက် approve/reject တိုင်း record တစ်ခု တွင်းထည့်ပါ။ Record ထဲမှာ reviewer id၊ decision၊ timestamp နဲ့ note တွေ ပါဝင်စေပါ။

### ဥပမာ

```python
import time
from typing import Annotated
import operator

class AuditState(TypedDict):
    action: str
    approvals: Annotated[list, operator.add]  # append-only audit log

def human_gate(state: AuditState) -> dict:
    decision = interrupt({"action": state["action"]})
    record = {
        "action": state["action"],
        "decision": decision["approved"],
        "reviewer": decision.get("reviewer", "system"),
        "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
    }
    return {"approvals": [record]}

def build_audit_graph():
    builder = StateGraph(AuditState)
    builder.add_node("gate", human_gate)
    builder.add_edge(START, "gate")
    builder.add_edge("gate", END)
    return builder.compile(checkpointer=MemorySaver())

g = build_audit_graph()
cfg = {"configurable": {"thread_id": "audit-1"}}
g.invoke({"action": "Pay invoice #42", "approvals": []}, cfg)
final = g.invoke(Command(resume={"approved": False, "reviewer": "alice"}), cfg)
print(final["approvals"][0])
# Expected output: a dict with decision=False, reviewer="alice",
# a timestamp string, and the action "Pay invoice #42"
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Ticket / escalation workflow တွေမှာ ဒီ pattern တွေက တိုက်ရိုက် အသုံးဝင်ပါတယ် — pending approval တစ်ခုကို Jira/ServiceNow ticket တစ်ခုအဖြစ် ဖွင့်ပြီး၊ reviewer က ticket ပိတ်တဲ့အခါ LangGraph run ကို `Command` နဲ့ resume လုပ်တာပါ။ Audit log က compliance review နဲ့ post-incident analysis နှစ်ခုစလုံးအတွက် အခြေခံ data ဖြစ်ပါတယ်။

## အနှစ်ချုပ်

- `interrupt()` က node တစ်ခုထဲမှာ approval gate ထည့်ပေးပြီး run ကို checkpoint နဲ့ ရပ်စေပါတယ် — checkpointer လိုအပ်ပါတယ်။
- `get_state` နဲ့ `update_state` က pending run ရဲ့ state ကို ကြည့်ရှုပြီး တည်းဖြတ်ခွင့်ပေးပါတယ်။
- `Command(resume=...)` က တူညီတဲ့ `thread_id` နဲ့ run ကို ရပ်နေတဲ့နေရာကနေ ဆက်လည်ပတ်စေပါတယ်။
- Timeout policy တွေက scheduled job တစ်ခုကနေ pending run တွေကို စစ်ပြီး auto-reject သို့မဟုတ် escalate လုပ်နိုင်ပါတယ်။
- `approvals` list တစ်ခု state ထဲထည့်တာက audit trail အတွက် ရိုးရိုးရှင်းရှင်းနဲ့ ခိုင်မာတဲ့ နည်းလမ်းဖြစ်ပါတယ်။
