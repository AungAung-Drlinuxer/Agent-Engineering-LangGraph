# ဖြေရှင်းချက်များ — Human-in-the-Loop Approval & Interrupts

## လေ့ကျင့်ခန်း ၁ — ပထမ Approval Gate

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt
from typing import TypedDict

class State(TypedDict):
    action: str
    message: str

def plan(state: State) -> dict:
    # A real agent would call an LLM here; we use a fixed plan
    return {"action": "Send email to team@example.com"}

def approve(state: State) -> dict:
    # Pause the run and expose the action for review
    interrupt({"action": state["action"]})
    return {}

def execute(state: State) -> dict:
    return {"message": "action executed"}

builder = StateGraph(State)
builder.add_node("plan", plan)
builder.add_node("approve", approve)
builder.add_node("execute", execute)
builder.add_edge(START, "plan")
builder.add_edge("plan", "approve")
builder.add_edge("approve", "execute")
builder.add_edge("execute", END)

graph = builder.compile(checkpointer=MemorySaver())
config = {"configurable": {"thread_id": "ex1"}}

result = graph.invoke({"action": "", "message": ""}, config)
print("__interrupt__" in result, result.get("__interrupt__"))
# Expected output: True followed by the interrupt payload containing the action
```

**အဓိကအယူအဆ** — Checkpointer တစ်ခုနဲ့ compile လုပ်ထားမှ `interrupt()` က run ကို checkpoint ထဲ သိမ်းပြီး ရပ်စေနိုင်ပါတယ်။

## လေ့ကျင့်ခန်း ၂ — State ကို Inspect လုပ်ခြင်း

```python
# Reuse the graph, config, and first invoke from Exercise 1
snapshot = graph.get_state(config)

print("State values:", snapshot.values)
print("Waiting at:", snapshot.next)
# Expected output: State values: {'action': 'Send email to team@example.com',
# 'message': ''}
# Waiting at: ('approve',)
```

**အဓိကအယူအဆ** — Interrupt ဖြစ်နေတဲ့ run က checkpointer ထဲမှာ ရှိနေဆဲဖြစ်ပြီး `get_state` က reviewer အတွက် လိုအပ်တဲ့ အချက်အလက်တွေကို ဖတ်ခွင့်ပေးပါတယ်။

## လေ့ကျင့်ခန်း ၃ — Resume နဲ့ Execute

```python
from langgraph.types import Command

def approve_with_decision(state: State) -> dict:
    # The return value of interrupt() is the resume payload
    decision = interrupt({"action": state["action"]})
    return {"message": ""}  # decision handled in execute node

def execute_with_decision(state: State) -> dict:
    # Re-check via interrupt result passed through state is complex;
    # simplest approach: read decision inside approve node
    return {"message": "action executed"}

def approve_returning(state: State) -> dict:
    decision = interrupt({"action": state["action"]})
    if decision["approved"]:
        return {"message": "action executed"}
    return {"message": "action rejected"}

# Build a two-node graph for clarity
b2 = StateGraph(State)
b2.add_node("plan", plan)
b2.add_node("approve", approve_returning)
b2.add_edge(START, "plan")
b2.add_edge("plan", "approve")
b2.add_edge("approve", END)
g2 = b2.compile(checkpointer=MemorySaver())

cfg_approve = {"configurable": {"thread_id": "ex3a"}}
g2.invoke({"action": "", "message": ""}, cfg_approve)
r1 = g2.invoke(Command(resume={"approved": True}), cfg_approve)
print(r1["message"])

cfg_reject = {"configurable": {"thread_id": "ex3b"}}
g2.invoke({"action": "", "message": ""}, cfg_reject)
r2 = g2.invoke(Command(resume={"approved": False}), cfg_reject)
print(r2["message"])
# Expected output:
# action executed
# action rejected
```

**အဓိကအယူအဆ** — `Command(resume=...)` ရဲ့ payload က `interrupt()` ရဲ့ return value ဖြစ်ပြီး၊ တူညီတဲ့ `thread_id` နဲ့မှ ရပ်နေတဲ့ run ကို ဆက်ချင်ရနိုင်ပါတယ်။

## လေ့ကျင့်ခန်း ၄ — Edit Before Approve

Interrupt ဖြစ်နေတဲ့အခါမှာ `graph.update_state(config, ...)` ကို အသုံးပြုပြီး state ထဲက `action` တန်ဖိုးကို တိုက်ရိုက် ပြင်ဆင်နိုင်ပါတယ်။ အောက်မှာ email ပို့တဲ့ action ကို execute လုပ်မယ့် node တစ်ခု၊ human approval အတွက် `interrupt()` တစ်ခု ပါဝင်တဲ့ graph အပြည့်အစုံကို ဖန်တီးပြီး interrupt ရပ်နေချိန်မှာ email address ကို `update_state` နဲ့ ပြင်ပြီးမှ resume လုပ်ပြီး ပြင်ပြီးတဲ့တန်ဖိုးနဲ့ execute node က အလုပ်လုပ်သလားဆိုတာကို စစ်ဆေးပြထားပါတယ်။

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt, Command
from typing import TypedDict


# Define the state schema
class State(TypedDict):
    action: dict
    result: str


def execute_node(state: State) -> dict:
    # Read the (possibly edited) action from the state
    action = state["action"]
    recipient = action["to"]
    subject = action["subject"]

    # Build the result using the current action values
    result = f"Email sent to {recipient} with subject '{subject}'"
    print(f"[execute_node] {result}")

    return {"result": result}


def approval_node(state: State) -> dict:
    # Pause execution here and ask a human for approval
    decision = interrupt(
        {"question": "Approve this action?", "action": state["action"]}
    )
    print(f"[approval_node] Human decision: {decision}")
    return {}


# Build the graph
builder = StateGraph(State)
builder.add_node("execute", execute_node)
builder.add_node("approve", approval_node)
builder.add_edge(START, "approve")
builder.add_edge("approve", "execute")
builder.add_edge("execute", END)

checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# Initial action with a wrong email address
initial_action = {"to": "wrong@example.com", "subject": "Status report"}

config = {"configurable": {"thread_id": "thread-1"}}

# First invocation: hits the interrupt inside approval_node
result = graph.invoke({"action": initial_action}, config=config)
print(f"Interrupted? {result.get('__interrupt__') is not None}")

# Edit the action BEFORE approving/resuming: fix the email address
edited_action = {"to": "correct@example.com", "subject": "Status report"}
graph.update_state(config, {"action": edited_action})

# Verify the state now holds the corrected action
snapshot = graph.get_state(config)
print(f"State action after edit: {snapshot.values['action']}")

# Resume with an approval decision
final = graph.invoke(Command(resume="approve"), config=config)

# The execute node used the EDITED action, not the original one
print(f"Final result: {final['result']}")
assert "correct@example.com" in final["result"], "Edited action was not used!"
print("SUCCESS: the edited action was used after resuming.")
```

**အဓိကအယူအဆ** — Interrupt ရပ်နေချိန်မှာ resume မလုပ်ခင် `graph.update_state(config, {"action": ...})` ကို ခေါ်ပြီး state ကို ပြင်ဆင်ထားရင် execute node က resume လုပ်တဲ့အခါ ပြင်ပြီးတဲ့ action တန်ဖိုးကိုသာ အသုံးပြုမှာ ဖြစ်ပါတယ်။

## လေ့ကျင့်ခန်း ၅ — Timeout Auto-Reject

ဒီလေ့ကျင့်ခန်းမှာ pending interrupt တစ်ခုရဲ့ သက်တမ်းကျော်မှုကို အလိုအလျောက် reject လုပ်တဲ့ logic ကို ရေးကြပါမယ်။ Human approval interrupt ကို စောင့်နေတဲ့ run တစ်ခုအတွက် `created_at` timestamp ကို state ထဲမှာ သိမ်းထားပြီး၊ helper function တစ်ခုက လက်ရှိအချိန်နဲ့ နှိုင်းယှဉ်ပြီး သက်တမ်းကျရင် `Command(resume={"approved": False, "reason": "timeout"})` နဲ့ run ကို auto-reject လုပ်ပါမယ်။

```python
import time
from langgraph.graph import StateGraph, START, END
from langgraph.types import Command, interrupt
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict

# Timeout window: 300 seconds (5 minutes)
TIMEOUT_SECONDS = 300


class State(TypedDict):
    created_at: float
    approved: bool
    reason: str


def request_approval(state: State):
    # Store the timestamp when the interrupt was created
    created_at = state.get("created_at", time.time())
    # Interrupt here and wait for a human decision
    decision = interrupt({"question": "Approve this request?"})
    return {
        "created_at": created_at,
        "approved": decision.get("approved", False),
        "reason": decision.get("reason", "manual"),
    }


def finalize(state: State):
    print("approved:", state["approved"], "| reason:", state["reason"])
    return state


builder = StateGraph(State)
builder.add_node("request_approval", request_approval)
builder.add_node("finalize", finalize)
builder.add_edge(START, "request_approval")
builder.add_edge("request_approval", "finalize")
builder.add_edge("finalize", END)

checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "order-42"}}
graph.invoke({"created_at": time.time()}, config)


def timeout_helper(config):
    """Auto-reject a pending run if it has exceeded its lifetime."""
    snap = graph.get_state(config)
    # An empty .next means the run already finished
    if not snap.next:
        return "run-finished"
    created_at = snap.values.get("created_at", 0)
    if time.time() - created_at > TIMEOUT_SECONDS:
        # Resume the interrupted run with an automatic rejection
        graph.invoke(
            Command(resume={"approved": False, "reason": "timeout"}), config
        )
        return "timeout-reject"
    return "still-pending"


# Simulate the pending request expiring: simulate an old timestamp
snap = graph.get_state(config)
graph.update_state(config, {"created_at": time.time() - 999})
result = timeout_helper(config)
print(result)  # timeout-reject

final = graph.get_state(config)
print(final.values["approved"], final.values["reason"])  # False timeout
```

`timeout_helper` ကို ခေါ်တဲ့အခါ `get_state(config)` နဲ့ run ရဲ့ အခြေအနေကို အရင်စစ်ပြီး `.next` ဗလာဖြစ်နေရင် run က ပြီးသွားပြီလို့ သတ်မှတ်ပါတယ်။ သက်တမ်း ကျော်လွန်နေရင် `Command(resume=...)` နဲ့ interrupt ကို `"approved": False` နဲ့ `"reason": "timeout"` ထည့်ပြီး ဆက်လက်လုပ်ဆောင်စေတာကြောင့် audit record ထဲမှာ timeout အကြောင်းပြချက် ပါဝင်ပါမယ်။

**အဓိကအယူအဆ** — Pending interrupt တစ်ခုရဲ့ သက်တမ်းကို `created_at` timestamp နဲ့ တိုင်းတာပြီး helper function က `get_state(config).next` ကို စစ်ဆေးခြင်းဖြင့် run ပြီးမြောက်ခဲ့ပြီလားဆိုတာကို ခွဲခြားကာ သက်တမ်းကျော်လျှင် `Command(resume={"approved": False, "reason": "timeout"})` ဖြင့် အလိုအလျောက် reject လုပ်ပါတယ်။

## လေ့ကျင့်ခန်း ၆ — Audit Trail နဲ့ Escalation

```python
import operator
from datetime import datetime, timezone
from typing import Annotated, TypedDict


class State(TypedDict):
    messages: Annotated[list, operator.add]
    approvals: Annotated[list, operator.add]


def approve(state: State) -> dict:
    # Simulate a reviewer decision
    reviewer_decision = "reject"  # change to "approve" to test the other path
    if reviewer_decision == "reject":
        approvals = [{
            "decision": "reject",
            "reviewer": "reviewer_01",
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "reason": "risk threshold exceeded",
        }]
        return {"approvals": approvals, "messages": ["reviewer rejected the request"]}
    approvals = [{
        "decision": "approve",
        "reviewer": "reviewer_01",
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "reason": "within policy limits",
    }]
    return {"approvals": approvals, "messages": ["reviewer approved the request"]}


def escalate(state: State) -> dict:
    # Notify that a human manager must intervene
    return {"messages": ["escalated to manager"]}


def execute(state: State) -> dict:
    # Normal execution path when approval succeeds
    return {"messages": ["request executed"]}


def route_after_approve(state: State) -> str:
    # Choose next node based on the last audit record
    last = state["approvals"][-1]
    return "escalate" if last["decision"] == "reject" else "execute"


from langgraph.graph import StateGraph, END

graph = StateGraph(State)
graph.add_node("approve", approve)
graph.add_node("escalate", escalate)
graph.add_node("execute", execute)
graph.set_entry_point("approve")
graph.add_conditional_edges("approve", route_after_approve, {
    "escalate": "escalate",
    "execute": "execute",
})
graph.add_edge("escalate", END)
graph.add_edge("execute", END)

app = graph.compile()
result = app.invoke({"messages": [], "approvals": []})
print(result)
```

**အဓိကအယူအဆ** — `operator.add` reducer နဲ့ append-only ဖြစ်တဲ့ approvals log တစ်ခုက တစ်ခြောက် approve/reject တိုင်းကို record တွင်းထည့်ပြီး `add_conditional_edges` က reject ဖြစ်ရင် `escalate` node ဆီ approve ဖြစ်ရင် `execute` node ဆီ လမ်းခွဲပေးလို့ ဖြစ်ပါမယ်။
