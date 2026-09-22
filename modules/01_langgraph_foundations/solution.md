# ဖြေရှင်းချက်များ — LangGraph Foundations

## လေ့ကျင့်ခန်း ၁ — ရိုးရိုး graph တစ်ခု တည်ဆောက်ခြင်း

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

# Shared state schema for the whole graph
class State(TypedDict):
    name: str
    greeting: str

# A node receives full state and returns a partial update
def make_greeting(state: State):
    return {"greeting": f"Hello, {state['name']}!"}

builder = StateGraph(State)
builder.add_node("make_greeting", make_greeting)
builder.add_edge(START, "make_greeting")
builder.add_edge("make_greeting", END)

app = builder.compile()
result = app.invoke({"name": "Aung", "greeting": ""})
print(result["greeting"])
# Expected output: Hello, Aung!
```

**အဓိကအယူအဆ** — Node တစ်ခုဆိုတာ state လက်ခံပြီး update တစ်စိတ်စိတ် ပြန်ပေးသော function သာဖြစ်သည်။

## လေ့ကျင့်ခန်း ၂ — Node နှစ်ခု တန်းဆက်ခြင်း

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    value: int
    steps: list[str]

def double(state: State):
    # Return a NEW list instead of mutating the old one
    return {"value": state["value"] * 2, "steps": state["steps"] + ["double"]}

def add_ten(state: State):
    return {"value": state["value"] + 10, "steps": state["steps"] + ["add_ten"]}

builder = StateGraph(State)
builder.add_node("double", double)
builder.add_node("add_ten", add_ten)
builder.add_edge(START, "double")
builder.add_edge("double", "add_ten")
builder.add_edge("add_ten", END)

app = builder.compile()
print(app.invoke({"value": 5, "steps": []}))
# Expected output: {'value': 20, 'steps': ['double', 'add_ten']}
```

**အဓိကအယူအဆ** — Edge တွေက node တွေရဲ့ အလုပ်လုပ်ရမယ့် အစီအစဉ်ကို သတ်မှတ်ပေးသည်။

## လေ့ကျင့်ခန်း ၃ — stream() ဖြင့် အဆင့်ဆင့် ကြည့်ခြင်း

```python
# Reuse the two-node graph from exercise 2, then stream instead of invoke
for chunk in app.stream({"value": 5, "steps": []}):
    print(chunk)
# Expected output:
# {'double': {'value': 10, 'steps': ['double']}}
# {'add_ten': {'value': 20, 'steps': ['double', 'add_ten']}}
```

**အဓိကအယူအဆ** — `stream()` က node တစ်ခုပြီးတိုင်း ရလဒ်အသစ်ကို တန်းပြေးမထုတ်ဘဲ အဆင့်ဆင့် ထုတ်ပေးသည်။

## လေ့ကျင့်ခန်း ၄ — Conditional edge ဖြင့် branch လုပ်ခြင်း

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    count: int

def increment(state: State):
    return {"count": state["count"] + 1}

def decide(state: State):
    # Loop back while count is below 5, otherwise finish
    if state["count"] < 5:
        return "increment"
    return END

builder = StateGraph(State)
builder.add_node("increment", increment)
builder.add_edge(START, "increment")
builder.add_conditional_edges("increment", decide)

app = builder.compile()
print(app.invoke({"count": 0}))
# Expected output: {'count': 5}
```

**အဓိကအယူအဆ** — Conditional edge က state အပေါ် အခြေခံပြီး graph ရဲ့ လမ်းကြောင်းကို ရွေးချယ်ပေးသည်၊ ဒါက cycle ဖန်တီးပေးသည်။

## လေ့ကျင့်ခန်း ၅ — Branch နှစ်ခု ရွေးခြင်း

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    number: int
    label: str

def even(state: State):
    return {"label": "even"}

def odd(state: State):
    return {"label": "odd"}

def router(state: State):
    # Choose which node to visit based on the number
    if state["number"] % 2 == 0:
        return "even"
    return "odd"

builder = StateGraph(State)
builder.add_node("even", even)
builder.add_node("odd", odd)
builder.add_conditional_edges(START, router)
builder.add_edge("even", END)
builder.add_edge("odd", END)

app = builder.compile()
print(app.invoke({"number": 4, "label": ""}))
print(app.invoke({"number": 5, "label": ""}))
# Expected output:
# {'number': 4, 'label': 'even'}
# {'number': 5, 'label': 'odd'}
```

**အဓိကအယူအဆ** — `START` ကနေတောင် တိုက်ရိုက် node တစ်ခုထဲ မသွားဘဲ router တစ်ခ်ဖြင့် လမ်းကြောင်း ရွေးနိုင်သည်။

## လေ့ကျင့်ခန်း ၆ — Chain နှင့် state machine နိုင်းယှဉ်ခြင်း

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    history: list[str]
    rounds: int

def call_model(state: State):
    return {"history": state["history"] + ["model"], "rounds": state["rounds"]}

def call_tool(state: State):
    return {"history": state["history"] + ["tool"]}

def check(state: State):
    return {"history": state["history"] + ["done"]}

# After the tool runs, decide: loop back to model or go to check
def after_tool(state: State):
    if state["rounds"] < 1:
        return "call_model"
    return "check"

builder = StateGraph(State)
builder.add_node("call_model", call_model)
builder.add_node("call_tool", call_tool)
builder.add_node("check", check)
builder.add_edge(START, "call_model")
builder.add_edge("call_model", "call_tool")
builder.add_conditional_edges("call_tool", after_tool)
builder.add_edge("check", END)

app = builder.compile()
result = app.invoke({"history": [], "rounds": 0})
print(result["history"])
# Expected output: ['model', 'tool', 'model', 'tool', 'done']

# Note: a plain chain (A -> B -> C) cannot express this cycle;
# a state machine with conditional edges is required.
```

**အဓိကအယူအဆ** — Chain တစ်ခုတည်းနဲ့ `call_tool` ကနေ `call_model` ကို ပြန်သွားတဲ့ cycle ကို ဖော်ပြလို့ မရနိုင်တဲ့အတွက် state machine ဖြစ်တဲ့ LangGraph graph ကို အသုံးပြုရသည်။
