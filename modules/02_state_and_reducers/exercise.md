## လေ့ကျင့်ခန်း ၁ — TypedDict ဖြင့် State Schema တည်ဆောက်ခြင်း

အောက်ပါ `State` schema ကို `TypedDict` ဖြင့်ရေးပြီး LangGraph ထဲတွင် run ကြည့်ပါ။ node နှစ်ခုက state ရှိ field များကို ပြောင်းလဲပါသည်။

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    question: str
    answer: str

def node_a(state: State) -> dict:
    # partial update: only the fields we want to change
    return {"question": "What is LangGraph?"}

def node_b(state: State) -> dict:
    return {"answer": f"A graph framework. Question was: {state['question']}"}

builder = StateGraph(State)
builder.add_node("a", node_a)
builder.add_node("b", node_b)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("b", END)
graph = builder.compile()

result = graph.invoke({"question": "initial", "answer": ""})
print(result)
```

**Hints:** node တစ်ခုစီသည် state အားလုံးမဟုတ်ဘဲ ပြောင်းလဲလိုသော field များသာ return လုပ်သည်ကို သတိပြုပါ။ `node_a` မှ `node_b` သို့ edge ဆက်ထားသဖြင့် `state['question']` ကို `node_b` တွင် ဖတ်နိုင်သည်။
**Expected behavior:** run ပြီးသောအခါ `question` field တွင် "What is LangGraph?" နှင့် `answer` field တွင် ထို question ပါဝင်သော string ပေါ်လာသည်။

## လေ့ကျင့်ခန်း ၂ — Reducer မရှိသည့်အခါ Overwrite ဖြစ်ပုံ လေ့လာခြင်း

`count: int` field ပါသော state တွင် node နှစ်ခုက `count` ကို တစ်ခုစီ update လုပ်ပါ။ reducer မထည့်ဘဲ run ကြည့်ပါ။

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    count: int

def add_one(state: State) -> dict:
    return {"count": state["count"] + 1}

def add_ten(state: State) -> dict:
    # no reducer: this value simply REPLACES the previous one
    return {"count": state["count"] + 10}

builder = StateGraph(State)
builder.add_node("add_one", add_one)
builder.add_node("add_ten", add_ten)
builder.add_edge(START, "add_one")
builder.add_edge("add_one", "add_ten")
builder.add_edge("add_ten", END)
graph = builder.compile()

print(graph.invoke({"count": 0}))
```

ထို့နောက် `add_one` နှင့် `add_ten` နှစ်ခုလုံးကို parallel branch (START မှ node နှစ်ခုလုံးသို့ edge၊ နှစ်ခုလုံးမှ END သို့ edge) ဖြင့် ပြန်တည်ဆောက်ပါ။ ရလဒ်က `10` (သို့ `1`) ဖြစ်သွားသည်ကို တွေ့ရပါမည် — ဘာကြောင့်ကြာင့် စဉ်းစားပါ။

**Hints:** reducer မရှိသော field တွင် update အသစ်သည် တန်ဖိုးအဟောင်းကို အစားထိုးသည်။ parallel node များသည် တစ်ချိန်တည်း same channel ကို update လုပ်လျှင် နောက်ဆုံး write တစ်ခုတည်းသာ ကျန်ရှိသည် (invalid update error ဖြစ်နိုင်သည်)။
**Expected behavior:** sequential version တွင် `count` သည် `11` ဖြစ်သည်။ parallel version တွင် overwrite (သို့ error) ကြောင့် ပေါင်းလဒ် `11` မရနိုင် — ဒါက reducer လိုအပ်ကြောင်း ပြသည်။

## လေ့ကျင့်ခန်း ၃ — Annotated reducer ဖြင့် integer ပေါင်းခြင်း

`operator.add` ကို `Annotated` ဖြင့်သုံးပြီး `count` field ကို **merge (accumulate)** စေပါ။ လေ့ကျင့်ခန်း ၂ ၏ parallel version ကို ဤ schema ဖြင့် ပြန် run ပါ။

```python
import operator
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # operator.add is the reducer: new = old + update
    count: Annotated[int, operator.add]

def add_one(state: State) -> dict:
    return {"count": 1}

def add_ten(state: State) -> dict:
    return {"count": 10}

builder = StateGraph(State)
builder.add_node("add_one", add_one)
builder.add_node("add_ten", add_ten)
builder.add_edge(START, "add_one")
builder.add_edge(START, "add_ten")
builder.add_edge("add_one", END)
builder.add_edge("add_ten", END)
graph = builder.compile()

print(graph.invoke({"count": 0}))
```

**Hints:** reducer ရှိပါက node များက **increment** (တိုးစ တန်ဖိုး) ကိုသာ return လုပ်သင့်သည်၊ state တစ်ခုလုံးမဟုတ်။ `operator.add` သည် integer များအတွက် `+` နှင့် list များအတွက် concatenation ဖြစ်သည်။
**Expected behavior:** parallel run ၌ `count` ရလဒ်မှာ `11` ဖြစ်သည် — update နှစ်ခုလုံး reducer ဖြင့် ပေါင်းစပ်သွားသည်။

## လေ့ကျင့်ခန်း ၄ — add_messages reducer ဖြင့် chat history စုဆောင်းခြင်း

`add_messages` reducer ကိုသုံးပြီး message များကို history အဖြစ် စုပါ။ user message နှစ်ခုကို `invoke` ဖြင့် ဆက်တင်ပြီး history ရှည်လာပုံကို ကြည့်ပါ။

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_core.messages import HumanMessage

class State(TypedDict):
    # add_messages appends messages instead of overwriting the list
    messages: Annotated[list, add_messages]

def echo(state: State) -> dict:
    # only return the NEW message(s); add_messages appends them
    return {"messages": [HumanMessage(content="echo node ran")]}

builder = StateGraph(State)
builder.add_node("echo", echo)
builder.add_edge(START, "echo")
builder.add_edge("echo", END)
graph = builder.compile()

r1 = graph.invoke({"messages": [HumanMessage(content="hello")]})
r2 = graph.invoke({"messages": [HumanMessage(content="second turn")],
                   # carry previous history forward for this exercise
                   })
for m in r1["messages"]:
    print(m.type, ":", m.content)
```

`r2` ကို run ရန် state တွင် ယခင် history ကို ထည့်ပေးပြီး စမ်းပါ၊ မည်သို့ append ဖြစ်သည်ကို လေ့လာပါ။

**Hints:** `add_messages` သည် `HumanMessage`, `AIMessage` စသည့် LangChain message objects များ (သို့ `(id, content)` tuples) ကို လက်ခံသည်။ node က message list အသစ်ကိုသာ return လုပ်ရုံသာရှိစေရမည် — reducer က ပေါင်းပေးမည်။
**Expected behavior:** node တစ်ခြောက် run တိုင်း `messages` list သည် ရှုံးသွားခြင်းမရှိဘဲ ရှေ့မှနောက်သို့ တစ်လျှောက်တည်း တိုးလာသည်။

## လေ့ကျင့်ခန်း ၅ — add_messages ၏ ID-matching ပြုမူချက် စမ်းသပ်ခြင်း

`add_messages` သည် **id တူသော** message ရောက်လာပါက append မလုပ်ဘဲ existing message ကို update လုပ်သည်။ ထိုစနစ်ကို အောက်ပါ script ဖြင့် စစ်ဆေးပါ။

```python
from typing import Annotated, TypedDict
from langchain_core.messages import AIMessage
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

base = AIMessage(content="version 1", id="msg_001")

# same id -> the reducer UPDATES instead of appending
updated = AIMessage(content="version 2", id="msg_001")

current = State(messages=[base])
new_state = add_messages(current["messages"], [updated])
print(len(new_state))
for m in new_state:
    print(m.id, "->", m.content)
```

ထို့နောက် `updated` message မှ `id` ကိုဖျက်ပြီး (သို့ id အသစ်ပေးပြီး) ပြန် run ပါ။ message အရေအတွက် ကွဲပြားပုံကို မှတ်တမ်းတင်ပါ။

**Hints:** id မပါသော message သည် အသစ်တစ်ခုအဖြစ် သတ်မှတ်ခံရပြီး list ထဲ ထပ်တိုးသည်။ id တူပါက reducer က ထို id ရှိ message ၏ `content` ကို အစားထိုးသည်။
**Expected behavior:** id တူသောအခါ list အရှည် `1` ဖြင့် content "version 2" ဖြစ်သည်။ id ကွဲသောအခါ list အရှည် `2` ဖြစ်သည်။

## လေ့ကျင့်ခန်း ၆ — Pydantic Schema နှင့် State Size စီမံခြင်း

`pydantic.BaseModel` ဖြင့် state schema တည်ဆောက်ပြီး (a) default တန်ဖိုးများ၊ (b) `add_messages` ဖြင့် history ရှည်လာမှု၊ (c) node တစ်ခုဖြင့် history ကို အနှစ်ချုံ့ခြင်း — ဤသုံးချက်လုပ်ပါ။

```python
from typing import Annotated
from pydantic import BaseModel, Field
from langchain_core.messages import HumanMessage, SystemMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class State(BaseModel):
    messages: Annotated[list, add_messages] = Field(default_factory=list)
    # full history is NOT stored here; keep state lean
    summary_turns: int = 0

def talk(state: State) -> dict:
    return {"messages": [HumanMessage(content=f"turn {len(state.messages)}")]}

def summarize(state: State) -> dict:
    # replace long history with a compact one; add_messages id-matching
    # won't help here, so we keep only the latest message
    latest = state.messages[-1:]
    return {"messages": latest, "summary_turns": state.summary_turns + 1}

builder = StateGraph(State)
builder.add_node("talk", talk)
builder.add_node("summarize", summarize)
builder.add_edge(START, "talk")
builder.add_edge("talk", "summarize")
builder.add_edge("summarize", END)
graph = builder.compile()

r = graph.invoke(State())
print("messages kept:", len(r["messages"]))
print("summary turns:", r["summary_turns"])
```

စိန်ခေါ်မှု — `summarize` node က history အားလုံးကို ဖျက်ရန် `add_messages` က id-matching ကြောင့် append လုပ်နေလျှင် မည်သို့ဖြေရှင်းမည်ကို စဉ်းစားပါ (ဥပမာ — `RemoveMessage` သုံးခြင်း၊ သို့ state ကို `messages` field မပါဘဲ `BaseModel` ခွဲ တည်ဆောက်ခြင်း)။

**Hints:** state ထဲ ကြီးမားသော history သိမ်းထားခြင်းသည် ရှည်သော conversation များတွင် memory နှင့် processing ကုန်ကျစရိတ် တိုးစေသည်။ LangGraph docs အရ `RemoveMessage(id=...)` သည် message တစ်ခုကို channel မှ ဖျက်ရန် သုံးသည်။ Pydantic schema သည် TypedDict ထက် runtime validation ပေးသည်။
**Expected behavior:** graph run တိုင်း history တိုးလာပြီး `summarize` က နောက်ဆုံး message တစ်ခုသာ ကျန်စေရန် ကြိုးစားသည် — သင့်ဖြေရှင်းနည်းအရ `messages` အရေအတွက် ကန့်သတ်ဖြစ်ပြီး `summary_turns` တိုးနေသည်။
