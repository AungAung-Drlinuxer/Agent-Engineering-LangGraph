# အဖြေများ — Persistence, Checkpointing & Time Travel

## လေ့ကျင့်ခန်း ၁ — InMemorySaver နဲ့ state ဆက်လက်တင်ရန်

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

class State(dict):
    pass

def counter(state: State):
    # Increment the count coming from the last checkpoint
    return {"count": state.get("count", 0) + 1}

builder = StateGraph(State)
builder.add_node("counter", counter)
builder.add_edge(START, "counter")
builder.add_edge("counter", END)

graph = builder.compile(checkpointer=InMemorySaver())
config = {"configurable": {"thread_id": "t1"}}

print(graph.invoke({"count": 0}, config))
print(graph.invoke(None, config))

# Expected output:
# {'count': 1}
# {'count': 2}
```

**အဓိကအယူအဆ** — InMemorySaver က process အတွင်းမှာ state ကို ဆက်လက်ထားပေးလို့ invoke နှစ်ချိန် run လို့ရပါတယ်။

## လေ့ကျင့်ခန်း ၂ — thread_id နှစ်ခုက သီးသန့်ဖြစ်ကြောင်း ပြရန်

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

class State(dict):
    pass

def counter(state: State):
    return {"count": state.get("count", 0) + 1}

builder = StateGraph(State)
builder.add_node("counter", counter)
builder.add_edge(START, "counter")
builder.add_edge("counter", END)
graph = builder.compile(checkpointer=InMemorySaver())

cA = {"configurable": {"thread_id": "user-A"}}
cB = {"configurable": {"thread_id": "user-B"}}

print(graph.invoke({"count": 0}, cA))   # user-A first run
print(graph.invoke(None, cA))           # user-A second run
print(graph.invoke({"count": 0}, cB))   # user-B is fully independent

print(graph.get_state(cA).values)
print(graph.get_state(cB).values)

# Expected output:
# {'count': 1}
# {'count': 2}
# {'count': 1}
# {'count': 2}
# {'count': 1}
```

**အဓိကအယူအဆ** — thread_id တူမှ state history တူပြီး ကွဲပြားတဲ့ thread တွေက လုံးဝ သီးသန့်ဖြစ်ပါတယ်။

## လေ့ကျင့်ခန်း ၃ — SqliteSaver နဲ့ ဖိုင်ထဲ သိမ်းရန်

```python
# Run this script twice; the second run continues from the file.
import sqlite3
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.sqlite import SqliteSaver

class State(dict):
    pass

def counter(state: State):
    return {"count": state.get("count", 0) + 1}

builder = StateGraph(State)
builder.add_node("counter", counter)
builder.add_edge(START, "counter")
builder.add_edge("counter", END)

conn = sqlite3.connect("checkpoints.db", check_same_thread=False)
graph = builder.compile(checkpointer=SqliteSaver(conn))

config = {"configurable": {"thread_id": "persist-1"}}
print(graph.invoke({"count": 0}, config))

# Expected output (first run):
# {'count': 1}
# Expected output (second run of the script):
# {'count': 2}
```

**အဓိကအယူအဆ** — SQLite file က checkpoint တွေကို process ပြင်ပမှာ သိမ်းထားလို့ restart ပြီးလည်း ဆက်လက်လို့ရပါတယ်။

## လေ့ကျင့်ခန်း ၄ — Fail ဖြစ်ပြီးနောက် တူညီတဲ့ thread မှာ resume လုပ်ရန်

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

class State(dict):
    pass

def risky_node(state: State):
    # Fail loudly on the first attempt, succeed afterwards
    if state.get("attempt", 0) == 0:
        raise RuntimeError("transient API failure")
    return {"message": "done"}

builder = StateGraph(State)
builder.add_node("risky_node", risky_node)
builder.add_edge(START, "risky_node")
builder.add_edge("risky_node", END)
graph = builder.compile(checkpointer=InMemorySaver())

config = {"configurable": {"thread_id": "recover-1"}}

try:
    graph.invoke({"attempt": 0}, config)
except RuntimeError as e:
    print("Failed:", e)

# Re-invoke on the SAME thread to resume
print(graph.invoke({"attempt": 1}, config))

# Expected output:
# Failed: transient API failure
# {'attempt': 1, 'message': 'done'}
```

**အဓိကအယူအဆ** — Node fail ဖြစ်ရင် အဲဒီ checkpoint မတင်ရှိမီ အဆင့်မှာ ရပ်တန့်နေပြီး တူညီတဲ့ thread မှာ ပြန် invoke လုပ်ရင် resume လုပ်လို့ရပါတယ်။

## လေ့ကျင့်ခန်း ၅ — Checkpoint history ကို ဖတ

အဓိကအယူအဆ — Checkpoint history ကို ဖတ်ခြင်းဖြင့် model က ဘယ်အချိန်မှာ ဘယ်ကုဒ်ကို ပြောင်းခဲ့သည်ကို ခြေရာခံနိုင်ပြီး၊ အမှားဖြစ်ခဲ့ပါက အရင် checkpoint သို့ ပြန်လည်ပြင်ဆင်နိုင်သည်။

## လေ့ကျင့်ခန်း ၆ — Long-term store နဲ့ user profile မျှဝေရန်

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

# First run: thread_1 — save the user profile for user u1
config_1 = {"configurable": {"thread_id": "thread_1", "user_id": "u1"}}
store.put(("profiles", "u1"), "name", {"value": "Aung Aung"})
result_1 = store.get(("profiles", "u1"), "name")
print("Run 1 (thread_1):", result_1.value)

# Second run: a completely new thread_id, same user id
config_2 = {"configurable": {"thread_id": "thread_2", "user_id": "u1"}}
result_2 = store.get(("profiles", "u1"), "name")
print("Run 2 (thread_2):", result_2.value)

# Prove that the store is keyed by user id, not by thread id
assert result_1.value == result_2.value
print("Both threads retrieved the same profile:", result_2.value["value"])
```

**အဓိကအယူအဆ** — `InMemoryStore` ထဲက ဒေတာဟာ thread_id နဲ့ မသက်ဆိုင်ဘဲ user id (namespace) နဲ့ ချိတ်ဆက်ထားတဲ့ကြောင့် ကွဲပြားတဲ့ thread နှစ်ခုကနေ တူညီတဲ့ profile ကို ဖတ်ယူနိုင်ပါတယ်။
