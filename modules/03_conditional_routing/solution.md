# အဖြေများ — Conditional Routing, Loops & Termination

## လေ့ကျင့်ခန်း ၁ — အခြေခံ Conditional Edge

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    score: int
    result: str

def grade_node(state: State) -> dict:
    # Entry node; grading decision is made in the router
    return {}

def route_by_score(state: State) -> str:
    # Route to pass_node or fail_node depending on the score
    if state["score"] >= 50:
        return "pass_node"
    return "fail_node"

def pass_node(state: State) -> dict:
    return {"result": "passed"}

def fail_node(state: State) -> dict:
    return {"result": "failed"}

builder = StateGraph(State)
builder.add_node("grade_node", grade_node)
builder.add_node("pass_node", pass_node)
builder.add_node("fail_node", fail_node)
builder.add_edge(START, "grade_node")
builder.add_conditional_edges("grade_node", route_by_score)
builder.add_edge("pass_node", END)
builder.add_edge("fail_node", END)
graph = builder.compile()

print(graph.invoke({"score": 80, "result": ""})["result"])
print(graph.invoke({"score": 30, "result": ""})["result"])
# Expected output:
# passed
# failed
```

**အဓိကအယူအဆ** — Router function တစ်ခုက state ကိုကြည့်ပြီး နောက် node နာမည်ကို return လုပ်တာနဲ့ conditional routing က ရိုးရိုးသားသား အလုပ်လုပ်ပါတယ်။

## လေ့ကျင့်ခန်း ၂ — Retry Cycle ထည့်ခြင်း

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    attempts: int
    is_valid: bool

def generate_node(state: State) -> dict:
    # Increment attempts; become valid on the second attempt
    attempts = state["attempts"] + 1
    is_valid = attempts >= 2
    return {"attempts": attempts, "is_valid": is_valid}

def route_validity(state: State) -> str:
    # Loop back to generate until valid
    if state["is_valid"]:
        return END
    return "generate"

builder = StateGraph(State)
builder.add_node("generate", generate_node)
builder.add_edge(START, "generate")
builder.add_conditional_edges("generate", route_validity, [END, "generate"])
graph = builder.compile()

result = graph.invoke({"attempts": 0, "is_valid": False})
print("attempts:", result["attempts"], "valid:", result["is_valid"])
# Expected output: attempts: 2 valid: True
```

**အဓိကအယူအဆ** — Conditional edge တစ်ခုပဲ နောက်ကို ရှေ့ဆီပြန်ညွှန်းရင် graph ထဲမှာ retry cycle ဖြစ်ပါတယ်။

## လေ့ကျင့်ခန်း ၃ — recursion_limit ကို စမ်းကြည့်ခြင်း

```python
from langgraph.graph import StateGraph, START, END
from langgraph.errors import GraphRecursionError
from typing import TypedDict

class State(TypedDict):
    count: int

def loop_node(state: State) -> dict:
    # Never-ending node; the router always comes back here
    return {"count": state["count"] + 1}

def route_forever(state: State) -> str:
    # Always returns to loop_node -> infinite loop
    return "loop_node"

builder = StateGraph(State)
builder.add_node("loop_node", loop_node)
builder.add_edge(START, "loop_node")
builder.add_conditional_edges("loop_node", route_forever)
graph = builder.compile()

try:
    graph.invoke({"count": 0}, config={"recursion_limit": 5})
except GraphRecursionError:
    print("recursion limit reached")
# Expected output: recursion limit reached
```

**အဓိကအယူအဆ** — `recursion_limit` က infinite loop ကို error အဖြစ် ရပ်တန့်စေတဲ့ safety net ဖြစ်ပြီး သူ့ကို try/except နဲ့ ကိုင်တွယ်ရပါတယ်။

## လေ့ကျင့်ခန်း ၄ — Counter-based Guardrail

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    attempts: int

MAX_ATTEMPTS = 4

def refine_node(state: State) -> dict:
    # Each pass increments the attempt counter
    return {"attempts": state["attempts"] + 1}

def route_attempts(state: State) -> str:
    # Exit the loop once the counter guardrail is reached
    if state["attempts"] >= MAX_ATTEMPTS:
        return END
    return "refine"

builder = StateGraph(State)
builder.add_node("refine", refine_node)
builder.add_edge(START, "refine")
builder.add_conditional_edges("refine", route_attempts, [END, "refine"])
graph = builder.compile()

result = graph.invoke({"attempts": 0})
print("attempts:", result["attempts"])
# Expected output: attempts: 4
```

**အဓိကအယူအဆ** — State ထဲက counter တစ်ခုက loop ရဲ့ အကြိမ်အရေအတွက်ကို recursion_limit မှီရင်ပိုပြီး တိကျစွာ ကန့်သတ်ပေးပါတယ်။

## လေ့ကျင့်ခန်း ၅ — Deterministic Fallback

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    attempts: int
    final: str

MAX_ATTEMPTS = 3

def refine_node(state: State) -> dict:
    # Simulated refinement that never produces a good result
    return {"attempts": state["attempts"] + 1}

def route_attempts(state: State) -> str:
    # Send the run to the fallback node once attempts run out
    if state["attempts"] >= MAX_ATTEMPTS:
        return "fallback"
    return "refine"

def fallback_node(state: State) -> dict:
    # Deterministic fallback: always the same safe default value
    return {"final": "default"}

builder = StateGraph(State)
builder.add_node("refine", refine_node)
builder.add_node("fallback", fallback_node)
builder.add_edge(START, "refine")
builder.add_conditional_edges("refine", route_attempts, ["refine", "fallback"])
builder.add_edge("fallback", END)
graph = builder.compile()

result = graph.invoke({"attempts": 0, "final": ""})
print("final:", result["final"], "attempts:", result["attempts"])
# Expected output: final: default attempts: 3
```

**အဓိကအယူအဆ** — Retry တွေ ကုန်သွားရင် ဘာမှမှန်းမသိတဲ့ ရလဒ်အစား သေချာတဲ့ default တန်ဖိုးတစ်ခု ပြန်ပေးတာက system ကို ယုံကြည်စိတ်ချရစေပါတယ်။

## လေ့ကျင့်ခန်း ၆ — ပေါင်းစပ်ခြင်း (မိုင်းလုံးဝင်ပုံစံ)

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

# Shared state: the draft text and how many attempts have been made
class State(TypedDict):
    text: str
    attempts: int

# Simulates a model that only produces a "good" text on the 4th call,
# so with a max of 3 attempts the fallback is always reached here.
def draft(state: State) -> dict:
    attempts = state["attempts"] + 1
    if attempts >= 4:
        text = "This draft looks good and is ready."
    else:
        text = "This draft is still too rough."
    print(f"[draft] attempt {attempts}: {text}")
    return {"text": text, "attempts": attempts}

# Router: check quality first, then the max-attempts guardrail
def review(state: State) -> str:
    if "good" in state["text"]:
        return "end"
    if state["attempts"] >= 3:
        return "fallback"
    return "retry"

# Deterministic fallback when quality never passes within the budget
def fallback(state: State) -> dict:
    print("[fallback] attempts exhausted, returning partial result")
    return {"text": "partial result"}

# Build the graph
builder = StateGraph(State)
builder.add_node("draft", draft)
builder.add_node("fallback", fallback)

builder.add_edge(START, "draft")
builder.add_conditional_edges(
    "draft",
    review,
    {"end": END, "retry": "draft", "fallback": "fallback"},
)
builder.add_edge("fallback", END)

graph = builder.compile()

# Run the workflow
result = graph.invoke({"text": "", "attempts": 0})
print("Final result:", result["text"])
```

**အဓိကအယူအဆ** — စွမ်းဆောင်ရည်စစ်ဆေးမှုကို အရင်လုပ်ပြီး ကြိုးပမ်းမှုအရေအတွက် ကန့်သတ်ချက်ကို ဒုတိယအဆင့်စစ်ဆေးခြင်းဖြင့် ကြိုးပမ်းမှု စက်ဝန်းနှင့် အရည်အသွေးမမှီသောအခါ သတ်မှတ်ပြန်လှန်ဆန်းခြင်း (fallback) နှစ်မျိုးလုံးကို တစ်ပြိုင်တည်း စီမံနိုင်ပြီး အချိန်အနန္တ loop ကို ကာကွယ်ပေးပါသည်။

