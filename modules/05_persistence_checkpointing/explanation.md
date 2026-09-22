# Persistence, Checkpointing & Time Travel — အသေးစိတ်ရှင်းလင်းချက်

## ၁။ Checkpointer ဆိုတာ ဘာလဲ

### ဘာကို ဆိုလိုတာလဲ

LangGraph မှာ graph တစ်ခု လုပ်ဆောင်နေတုန်းက တစ် super-step တိုင်း ပြီးတိုင်း ၎င်းရဲ့ state ကို `checkpoint` ဆိုတဲ့ အချက်အလက်တစ်ခုအနေနဲ့ သိမ်းဆည်းပါတယ်။ ဒီ checkpoint တွေကို ဖန်တီးပေးပြီး စီမံခန့်ခွဲပေးတဲ့ component ကို `checkpointer` လို့ ခေါ်ပါတယ်။

### ဘာကြောင့် လဲ

Graph တစ်ခုက LLM call, tool call, human input စတဲ့ အချိန်ကြာတဲ့ အဆင့်တွေပါလေ့ရှိပါတယ်။ အဲဒီအချိန်တွေမှာ process ရပ်သွားရင် (crash, restart) state က လုံးဝ ပျောက်သွားပြီး user စကားဝိုင်း အစကနေ ပြန်စရမယ်။ Checkpointer ရှိရင် နောက်ဆုံး checkpoint ကနေ ဆက်လုပ်လို့ရပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

Graph compile လုပ်တဲ့အချိန်မှာ `compile(checkpointer=...)` နဲ့ checkpointer ထည့်ပေးရပါတယ်။ Run တိုင်းမှာ config ထဲမှာ `thread_id` ထည့်ပေးရပြီး၊ LangGraph က အဲဒီ thread အတွက် checkpoint တွေကို တစ်ဆင့်ချင်း သိမ်းသွားပါတယ်။ ပြန်ခေါ်တဲ့အခါ နောက်ဆုံး checkpoint ကနေ state ကို ပြန်တင်ပေးပါတယ်။

### ဥပမာ

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver

# Define a minimal state with one counter
class State(dict):
    pass

def step_a(state: State):
    return {"count": state.get("count", 0) + 1}

builder = StateGraph(State)
builder.add_node("step_a", step_a)
builder.add_edge(START, "step_a")
builder.add_edge("step_a", END)

# Compile with an in-memory checkpointer
graph = builder.compile(checkpointer=InMemorySaver())

# Run twice on the same thread; state persists across invocations
config = {"configurable": {"thread_id": "demo-1"}}
print(graph.invoke({"count": 0}, config))
print(graph.invoke(None, config))

# Expected output:
# {'count': 1}
# {'count': 2}
```

ဒုတိယ invoke မှာ input `None` ပေးထားပေမယ့် state က နောက်ဆုံး checkpoint ကနေ ပြန်တင်လို့ count ဆက်တိုးသွားတာ မြင်ရပါတယ်။

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Production agent တိုင်းမှာ crash, deploy restart, long-running tool call တွေ ဖြစ်နိုင်ပါတယ်။ Checkpoint မရှိရင် တစ်ခုမှ မအောင်မြင်ရင် တစ်လုံးတစ်စ ကုန်သွားတဲ့ agent ဖြစ်သွားပါတယ်။ Checkpoint ရှိရင်တော့ လုံးဝ မအောင်မြင်တဲ့ run တွေကိုပဲ ပြန်လုပ်ရပါတယ်။

## ၂။ Backend သုံးမျိုး — memory, sqlite, postgres

### ဘာကို ဆိုလိုတာလဲ

`InMemorySaver` က checkpoint တွေကို process memory ထဲမှာပဲ သိမ်းပါတယ်။ `SqliteSaver` က SQLite file ထဲမှာ သိမ်းပြီး၊ `PostgresSaver` က PostgreSQL database ထဲမှာ သိမ်းပါတယ်။

### ဘာကြောင့် လဲ

Memory checkpointer က process သေသွားရင် data အကုန်ပျောက်ပါတယ်။ SQLite က file-based ဖြစ်လို့ ပြန်ဖွင့်လို့ရပေမယ့် တစ် process အတွက် သင့်တင့်ပါတယ်။ Postgres ကတော့ multi-process / multi-server deployment မှာ checkpoint တွေကို တစ်နေရာတည်းကနေ မျှဝေလို့ရပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

SqliteSaver သုံးရန် `pip install langgraph-checkpoint-sqlite` လိုအပ်ပါတယ်။ PostgresSaver အတွက်ကတော့ `langgraph-checkpoint-postgres` ပါ။ အောက်မှာ SQLite နဲ့ ဥပမာ ကြည့်ပါမယ်။

### ဥပမာ

```python
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver

# Create a file-backed SQLite database; data survives restarts
conn = sqlite3.connect("checkpoints.db", check_same_thread=False)
saver = SqliteSaver(conn)

# Compile your graph with the sqlite checkpointer
graph = builder.compile(checkpointer=saver)

# The rest of the API is identical to InMemorySaver
config = {"configurable": {"thread_id": "user-42"}}
print(graph.invoke({"count": 10}, config))

# Expected output:
# {'count': 11}
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Deployment အရွယ်အစားပေါ် မူတည်ပြီး backend ရွေးရပါတယ် — prototype/tests အတွက် memory၊ single-machine အတွက် SQLite၊ container/k8s နဲ့ ချဲ့ထွင်မယ့် production အတွက် Postgres။ ရွေးချယ်မှုမှန်ရင် restart လိုအပ်ချက်တွေက agent ရဲ့ အမှား tolerance ကို သိသိသာသာ တိုးစေပါတယ်။

## ၃။ thread_id semantics

### ဘာကို ဆိုလိုတာလဲ

`thread_id` ဆိုတာ checkpoint history တစ်ခုကို ခေါ်ဝေါ်တဲ့ သော့ (key) ပါ။ Thread တစ်ခု = ဆွေးနွေးမှု (conversation) တစ်ခုလို့ မှတ်ယူနိုင်ပါတယ်။

### ဘာကြောင့် လဲ

User နှစ်ယောက် တစ်ပြိုင်နက် အသုံးပြုနေရင် သူတို့ရဲ့ state တွေကို ခွဲခြားဖို့ လိုပါတယ်။ thread_id မတူရင် LangGraph က လုံးဝ သီးသန့် checkpoint history နှစ်ခုလို့ မှတ်ပြီး တစ်ခုကို တစ်ခု မမြင်ရပါဘူး။

### ဘယ်လို အလုပ်လုပ်လဲ

Run တိုင်းမှာ `config["configurable"]["thread_id"]` ထည့်ရပါတယ်။ thread_id တူရင် state က နောက်ဆုံး checkpoint ကနေ ဆက်တာပါ။ ကွဲပြားရင် အသစ် စတင်တာပါ။

### ဥပမာ

```python
graph = builder.compile(checkpointer=InMemorySaver())

# Two different threads behave like two separate conversations
c1 = {"configurable": {"thread_id": "thread-A"}}
c2 = {"configurable": {"thread_id": "thread-B"}}

print(graph.invoke({"count": 5}, c1))
print(graph.invoke(None, c1))      # continues thread-A
print(graph.invoke({"count": 100}, c2))  # thread-B is independent

# Expected output:
# {'count': 6}
# {'count': 7}
# {'count': 101}
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Chatbot တစ်ခုမှာ "session id" ဒါမှမှာ user id + conversation id ကို thread_id အဖြစ် သုံးတာ အဓိက pattern ပါ။ thread_id ကို မှားယွင်း share လုပ်မိရင် user တစ်ယောက်ရဲ့ အချက်အလက်တွေ အခြားသူဆီ ယိုစိမ့်သွားနိုင်လို့  လုံခြုံရေးအရလည်း အရေးကြီးပါတယ်။

## ၄။ Resuming after failure

### ဘာကို ဆိုလိုတာလဲ

Process ဒါမှမှာ node တစ်ခု fail ဖြစ်သွားရင် နောက်ဆုံး အောင်မြင်တဲ့ checkpoint ကနေ graph ကို ပြန်စလို့ရတဲ့ သတ္တိကို ဆိုလိုပါတယ်။

### ဘာကြောင့် လဲ

LLM provider timeout၊ network ပြတ်တောက်ခြင်း၊ tool error တို့က production မှာ မဖြစ်မနေ ဖြစ်တတ်ပါတယ်။ Checkpoint မရှိရင် လုပ်ငန်းစဉ်တစ်ခုလုံး အစကနေ ပြန်စရမယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

Node တစ်ခု fail ရင် အဲဒီ node ရောက်ပြီးခင်တဲ့ checkpoint အထိပဲ သိမ်းထားပါတယ်။ အချက်အလက် ပြန်စီမံပြီး run အသစ် invoke လုပ်ရင် အဲဒီ checkpoint ကနေ ဆက်လုပ်ပါတယ်။

### ဥပမာ

```python
class State(dict):
    pass

def risky_node(state: State):
    # Simulate a node that fails on the first attempt
    if state.get("attempt", 0) == 0:
        raise RuntimeError("transient API failure")
    return {"message": "done"}

builder2 = StateGraph(State)
builder2.add_node("risky_node", risky_node)
builder2.add_edge(START, "risky_node")
builder2.add_edge("risky_node", END)
graph2 = builder2.compile(checkpointer=InMemorySaver())

c = {"configurable": {"thread_id": "recover-1"}}
try:
    graph2.invoke({"attempt": 0}, c)
except RuntimeError as e:
    print("Failed:", e)

# Fix the condition and re-invoke on the SAME thread
print(graph2.invoke({"attempt": 1}, c))

# Expected output:
# Failed: transient API failure
# {'attempt': 1, 'message': 'done'}
```

### လက္တေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Batch processing ဒါမှမှာ နာရီပေါင်းများစွာ လုပ်ငန်းတွေမှာ တစ် node ချင်း ပြန်စလို့ရတာက ကုန်ကျစရိတ်နဲ့ အချိန် အများကြီး ချွေတာပေးပါတယ်။ `interrupt_before` / `interrupt_after` option တွေနဲ့ တွဲသုံးရင် human-in-the-loop workflow တွေလည်း ဖန်တီးလို့ရပါတယ်။

## ၅။ Time travel နဲ့ long-term store

### ဘာကို ဆိုလိုတာလဲ

Time travel က history ထဲက အတိတ် checkpoint တစ်ခုကို replay လုပ်တာပါ။ Long-term store (`InMemoryStore` စသည်) ကတော့ thread အချင်းချင်း ကျော်လွန်ပြီး သိမ်းဆည်းလို့ရတဲ့ key-value memory ပါ။

### ဘာကြောင့် လဲ

Agent ရဲ့ အတိတ်ဆုံးဖြတ်ချက်ကို ပြင်ပြီး "what if" စမ်းသပ်ချင်တာ ဒါမှမှာ တစ် user ရဲ့ profile ကို စကားဝိုင်းအားလုံးမှာ မှတ်မိစေချင်တာတို့အတွက် ဒီနှစ်ခု လိုအပ်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

`graph.get_state_history(config)` နဲ့ checkpoint စာရင်း ယူလို့ရပြီး အချက်အလက်ပြင်ပြီး replay လုပ်လို့ရပါတယ်။ Long-term store ကတော့ `compile(store=...)` နဲ့ တွဲပေးရပြီး namespace (များသာအားဖြင့် user id) အလိုက် သိမ်းပါတယ်။

### ဥပမာ

```python
from langgraph.checkpoint.memory import InMemorySaver

graph3 = builder.compile(checkpointer=InMemorySaver())
c = {"configurable": {"thread_id": "history-1"}}

graph3.invoke({"count": 0}, c)
graph3.invoke(None, c)

# Walk the checkpoint history (time travel view)
for snap in graph3.get_state_history(c):
    print(snap.config["configurable"]["thread_id"],
          snap.values, snap.next)

# Expected output:
# history-1 {'count': 2} ()
# history-1 {'count': 1} ('step_a',)
# history-1 {} ()
```

### လက္တေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Time travel က agent ရဲ့ ဆုံးဖြတ်ချက်တွေကို debug ပြုလုပ်ရန်နှင့် စမ်းသပ်ရန် အလွန်ကောင်းတဲ့ နည်းလမ်းဖြစ်ပြီး၊ long-term store ကတော့ agent ကို user ရဲ့ ရှေ့ဆက်မှတ်မိစေတဲ့ စွမ်းရည် ပေးပါတယ်။

## အနှစ်ချုပ်

Checkpointing က LangGraph agent တိုင်းရဲ့ production-readiness အတွက် မဖြစ်မနေ လိုအပ်တဲ့ အစိတ်အပိုင်းပါ — backend သုံးမျိုးထဲက deployment အတွက် သင့်တင့်တဲ့တစ်ခု ရွေးချယ်ပြီး၊ thread_id နဲ့ စကားဝိုင်းတွေကို ခွဲခြားပါ။ Failure နောက် checkpoint ကနေ resume လုပ်ခြင်း၊ history ကနေ replay လုပ်ခြင်း၊ နဲ့ store နဲ့ cross-thread memory တို့က agent ကို ပိုမိုတောင့်တင်းစေပြီး user အတွေ့အကြုံ ပိုကောင်းစေပါတယ်။
