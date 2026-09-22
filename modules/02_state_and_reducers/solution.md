## လေ့ကျင့်ခန်း ၁ — TypedDict ဖြင့် State Schema ဖန်တီးခြင်း

State ဆိုသည်မှာ graph တစ်ခုလုံး၏ မှတ်တမ်း data structure ဖြစ်သည်။ Python ၏ `TypedDict` ဖြင့် ရိုးရှင်းစွာ သတ်မှတ်နိုင်သည်။

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class AgentState(TypedDict):
    # user_input has no reducer, so each write overwrites the old value
    user_input: str
    # messages uses add_messages reducer to append instead of overwrite
    messages: Annotated[list, add_messages]


def step_a(state: AgentState) -> dict:
    # returning a partial update: only the keys we want to change
    return {"messages": [("user", state["user_input"])]}


def step_b(state: AgentState) -> dict:
    # the messages list already contains step_a's message
    print("Current messages:", state["messages"])
    return {"user_input": "processed"}


builder = StateGraph(AgentState)
builder.add_node("a", step_a)
builder.add_node("b", step_b)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("b", END)

graph = builder.compile()
graph.invoke({"user_input": "hello", "messages": []})
```

**အဓိကအယူအဆ** — TypedDict သည် state ၏ပုံစံကို အကောင်းဆုံး ရှင်းလင်းစွာဖော်ပြပြီး reducer မပါသော key များသည် တိုက်ရိုက် overwrite ဖြစ်သည်။

## လေ့ကျင့်ခန်း ၂ — Pydantic Model ဖြင့် Validation ပြုလုပ်ခြင်း

Pydantic ကိုသုံးလျှင် state ထဲဝင်လာသော data များကို အမျိုးအစားစစ်ဆေးမှု (validation) ရရှိသည်။

```python
from typing import Annotated
from pydantic import BaseModel, field_validator
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class AgentState(BaseModel):
    messages: Annotated[list, add_messages]
    score: int = 0

    @field_validator("score")
    @classmethod
    def check_score(cls, v: int) -> int:
        # score must never be negative
        if v < 0:
            raise ValueError("score must be >= 0")
        return v


def increment(state: AgentState) -> dict:
    # partial update: only score changes, messages untouched
    return {"score": state.score + 1}


builder = StateGraph(AgentState)
builder.add_node("increment", increment)
builder.add_edge(START, "increment")
builder.add_edge("increment", END)

graph = builder.compile()
result = graph.invoke({"messages": [], "score": 5})
print(result["score"])  # 6
```

**အဓိကအယူအဆ** — Pydantic state သည် မမှန်ကန်သော data များ ဝင်လာခြင်းကို အချိန်မီစစ်ဆေးနိုင်သဖြင့် production တွင် ပိုမှာဝန်ခံနိုင်သည်။

## လေ့ကျင့်ခန်း ၃ — add_messages Reducer လုပ်ဆောင်ချက်

`add_messages` သည် messages list ကို အသစ်ရောက်တိုင်း append လုပ်ပေးသော built-in reducer ဖြစ်သည်။

```python
from langgraph.graph.message import add_messages

# add_messages follows operator.add semantics: old + new
old = [
    {"role": "user", "content": "hi"},
    {"role": "assistant", "content": "hello!"},
]
new = [{"role": "user", "content": "what is 2+2?"}]

merged = add_messages(old, new)
print(len(merged))  # 3 messages, nothing was lost

# if a new message has the same id as an old one, it REPLACES it
old_ids = [("user", "hi", "m1")]
updated = [{"role": "user", "content": "hi there", "id": "m1"}]
result = add_messages(old_ids, updated)
```

**အဓိကအယူအဆ** — add_messages သည် message အသစ်များကို ထည့်သွင်းပြီး တူညီသော id ရှိပါက အဟောင်းကို အစားထိုးသောကြောင့် စကားဝိုင်းမှတ်တမ်း လုံးဝမပျောက်ပေ။

## လေ့ကျင့်ခန်း ၄ — Reducer မရှိပါက Overwrite ဖြစ်ပုံ

Reducer သည် တစ်ခုထက်ပိုသော node (သို့) parallel branch များမှ တစ်ပြိုင်နက် update လာသောအခါ ပေါင်းစည်းပုံကို ဆုံးဖြတ်ပေးသည်။

```python
from operator import add
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    # no reducer -> last write wins (overwrite)
    plain: list
    # with add reducer -> lists are concatenated
    merged: Annotated[list, add]


def left(state: State) -> dict:
    return {"plain": ["left"], "merged": ["left"]}


def right(state: State) -> dict:
    return {"plain": ["right"], "merged": ["right"]}


builder = StateGraph(State)
builder.add_node("left", left)
builder.add_node("right", right)
# left and right run in parallel in the same super-step
builder.add_edge(START, "left")
builder.add_edge(START, "right")
builder.add_edge("left", END)
builder.add_edge("right", END)

graph = builder.compile()
result = graph.invoke({"plain": [], "merged": []})
print(result["plain"])   # only "left" or "right" survived
print(result["merged"])  # ["left", "right"] both kept
```

**အဓိကအယူအဆ** — Reducer ရှိမှု၊ မရှိမှုသည် value များ ပေါင်းစည်းမလား အစားထိုးမလားဆိုသည်ကို ဆုံးဖြတ်သောကြောင့် parallel node များရှိပါက reducer မဖြစ်မနေလိုအပ်သည်။

## လေ့ကျင့်ခန်း ၅ — Partial State Updates

Node တစ်ခုသည် state အားလုံးကို ပြန်ပေးရန် မလိုအပ်ဘဲ ပြောင်းလိုသော key များကိုသာ ပြန်ပေးလျှင် ရသည်။

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
    question: str
    answer: str
    attempts: int


def think(state: State) -> dict:
    # partial update: question stays untouched
    return {"attempts": state["attempts"] + 1}


def respond(state: State) -> dict:
    return {"answer": f"Answer to: {state['question']}"}


builder = StateGraph(State)
builder.add_node("think", think)
builder.add_node("respond", respond)
builder.add_edge(START, "think")
builder.add_edge("think", "respond")
builder.add_edge("respond", END)

graph = builder.compile()
result = graph.invoke({"question": "what is LangGraph?",
                       "answer": "", "attempts": 0})
print(result["question"])  # original value preserved
print(result["answer"])    # only what respond wrote
print(result["attempts"])  # 1
```

**အဓိကအယူအဆ** — Node တစ်ခုက return လုပ်သော dict တွင်ပါဝင်သော key များသာ update ဖြစ်ပြီး ကျန် key များမှာ မပြောင်းလဲဘဲ လက်ရှိတန်ဖိုးအတိုင်း ရှိနေသည်။

## လေ့ကျင့်ခန်း ၆ — State အရွယ်နှင့် Cost အကျိုးသက်ရောက်မှု

State ကြီးလာလျှင် super-step တိုင်းတွင် state အားလုံးကို ဖတ်ရ၊ ရေးရ၊ checkpoint တွင် သိမ်းရသောကြောင့် ကုန်ကျစရိတ်တက်သည်။ ကြီးမားသော object များကို state ထဲမထည့်ဘဲ DocStore/external storage တွင် ထားပြီး ID သာ သိမ်းသင့်သည်။

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END


class BigDoc:
    def __init__(self, data: str) -> None:
        self.data = data


# module-level store, outside of graph state
DOC_STORE: dict[str, BigDoc] = {}


class LeanState(TypedDict):
    doc_id: str        # small: just a reference
    summary: str       # small: only what the LLM truly needs


def load_doc(state: LeanState) -> dict:
    # fetch the big object only when needed, keep state small
    doc = DOC_STORE[state["doc_id"]]
    return {"summary": doc.data[:50] + "..."}


DOC_STORE["d1"] = BigDoc("huge document text " * 10000)

builder = StateGraph(LeanState)
builder.add_node("load", load_doc)
builder.add_edge(START, "load")
builder.add_edge("load", END)

graph = builder.compile()
result = graph.invoke({"doc_id": "d1", "summary": ""})
print(result["summary"])
```

**အဓိကအယူအဆ** — State ထဲတွင် လိုအပ်သော သတင်းအချက်အလက်ကလေးများသာ ထားပြီး ကြီးမားသော data များကို အပြင်တွင် ID ဖြင့် ချိတ်ဆက်ထားခြင်းက memory နှင့် checkpoint ကုန်ကျစရိတ်ကို သိသိသာသာ လျှော့ချပေးသည်။
