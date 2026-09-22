# State, Messages & Reducers

## ၁။ State Schema (TypedDict / Pydantic)

### ဘာကို ဆိုလိုတာလဲ
LangGraph မှာ "state" ဆိုတာ graph တစ်ခုလုံး လည်ပတ်နေစဉ် အတွင်း node တွေအချင်းချင်း မျှဝေသုံးစွဲနေတဲ့ ဒေတာထုပ်ပိုးမှု (data container) ဖြစ်ပါတယ်။ ဒီ state ရဲ့ ပုံစံကို `TypedDict` (Python built-in typing) ဒါမှမဟုတ် `Pydantic` `BaseModel` နဲ့ သတ်မှတ်ပေးရပါတယ်။ Schema ဆိုတာ "ဘယ် field တွေပါမလဲ၊ ဘယ် type ဖြစ်မလဲ" ဆိုတာကို ကြိုတင်သတ်မှတ်တဲ့ စာချုပ်လိုမျိုး ဖြစ်ပါတယ်။

### ဘာကြောင့် လဲ
Schema မသတ်မှတ်ဘူးဆိုရင် state ထဲမှာ မှားယွင်းတဲ့ key ထည့်မိတာ၊ မှားတဲ့ type ထည့်မိတာတွေကို runtime မှာမှ သိရပြီး debug လုပ်ရခက်ပါတယ်။ Schema ရှိရင် (၁) ကုဒ်ဖတ်သူက state ထဲမှာ ဘာရှိလဲဆိုတာ ချက်ချင်းမြင်ရပြီး (၂) Pydantic သုံးရင် runtime validation ရပါတယ်။ LangGraph ကလည်း node တွေရဲ့ return value ကို schema နဲ့ ကိုက်ညီမှုရှိမရှိ စစ်ဆေးပေးပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
`TypedDict` နဲ့ ရေးရင် type checker (mypy, IDE) လုပ်ဆောင်ပေးပြီး runtime မှာ dict သာဖြစ်ပါတယ်။ Pydantic `BaseModel` သုံးရင် object instantiation အချိန်မှာ validation ဖြစ်ပြီး မှားရင် error တက်ပါတယ်။ နှစ်မျိုးစလုံးကို LangGraph state အဖြစ် အသုံးပြုနိုင်ပါတယ်။ State schema ထဲမှာ field တစ်ခုချင်းစီအတွက် type annotation ရေးပြီး၊ reducer လိုအပ်တဲ့ field တွေမှာ `Annotated` နဲ့ တွဲသတ်မှတ်ပါတယ်။

### ဥပမာ

```python
from typing_extensions import TypedDict, Annotated
from pydantic import BaseModel, Field
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

# Option 1: TypedDict state schema
class DictState(TypedDict):
    topic: str                       # plain field: last write wins
    messages: Annotated[list, add_messages]  # reducer field: merged

# Option 2: Pydantic state schema (runtime-validated)
class PydanticState(BaseModel):
    topic: str = Field(description="The topic to process")
    messages: Annotated[list, add_messages] = Field(default_factory=list)

def step_a(state: DictState) -> dict:
    # Return only the keys we want to update (partial update)
    return {"topic": state["topic"] + " (processed)"}

builder = StateGraph(DictState)
builder.add_node("step_a", step_a)
builder.add_edge(START, "step_a")
builder.add_edge("step_a", END)
graph = builder.compile()

result = graph.invoke({"topic": "agents"})
print(result["topic"])
# Expected output: agents (processed)
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Agent project တစ်ခုကြီးလာတဲ့အခါ state ထဲမှာ ဘာ field တွေရှိလဲ၊ ဘယ် node က ဘယ် field ကို ပြင်လဲဆိုတာ ရှင်းရှင်းလင်းလင်း သိနေဖို့ အရေးကြီးပါတယ်။ Pydantic schema သုံးတာက production မှာ LLM က မှားတဲ့ data ထည့်လာတာကို စောစောရှင်းနိုင်စေပြီး TypedDict က lightweight ဖြစ်တာကြောင့် prototype အတွက် အသုံးဝင်ပါတယ်။

## ၂။ Annotated Reducers (add_messages)

### ဘာကို ဆိုလိုတာလဲ
Reducer ဆိုတာ state field တစ်ခုမှာ တိုက်ရိုက် overwrite မလုပ်ဘဲ ရှိနေပြီးသား value နဲ့ အသစ်ဝင်လာတဲ့ value နှစ်ခုကို ဘယ်လို ပေါင်းစပ်မလဲဆိုတဲ့ function ဖြစ်ပါတယ်။ LangGraph မှာ ဒါကို `Annotated[Type, reducer_function]` ပုံစံနဲ့ သတ်မှတ်ပါတယ်။ အသုံးအများဆုံး reducer က conversation messages စုစည်းပေးတဲ့ `add_messages` ဖြစ်ပါတယ်။

### ဘာကြောင့် လဲ
Agent တစ်ကောင်မှာ conversation history က ပိုင်းခြားလို့မရပါဘူး — user message၊ AI reply၊ tool call အားလုံးကို အစဉ်အတိုင်း စုထားဖို့လိုပါတယ်။ Reducer မရှိဘူးဆို နောက်ဆုံး node ရဲ့ return value က အရင် messages အားလုံးကို ဖျက်ပြီး overwrite မှာ ဖြစ်ပြီး conversation memory ဆုံးရှုံးသွားပါလိမ့်မယ်။ Reducer က "merge vs overwrite" ကို field တစ်ခုချင်းစီအတွက် ကြိုတင်ဆုံးဖြတ်ပေးတဲ့ စနစ်ကျတဲ့နည်းလမ်း ဖြစ်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
Node တစ်ခုက `{"messages": [new_message]}` ဆိုပြီး return ပြန်တဲ့အခါ LangGraph က `add_messages(old_list, new_list)` ကို ခေါ်ပါတယ်။ ဒီ function က new message တွေကို old list ရဲ့ နောက်မှာ append ပါတယ်။ အဓိကကျတဲ့ အချက် — message အသစ်မှာ ID ရှိပြီး အဲဒီ ID ဟာ old list ထဲမှာ ရှိနေရင် append မလုပ်ဘဲ အဲဒါကို replace လုပ်ပါတယ် (update semantics)။ ID မတူရင် append လုပ်ပါတယ်။ Reducer မသတ်မှတ်ထားတဲ့ field (ဥပမာ `topic: str`) ကတော့ overwrite လုပ်ပါတယ်။

### ဥပမာ

```python
from typing_extensions import TypedDict, Annotated
from langchain_core.messages import HumanMessage, AIMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class ChatState(TypedDict):
    # add_messages reducer: new items are appended to the existing list
    messages: Annotated[list, add_messages]

def human_step(state: ChatState) -> dict:
    return {"messages": [HumanMessage(content="Hello!")]}

def ai_step(state: ChatState) -> dict:
    return {"messages": [AIMessage(content="Hi, how can I help?")]}

builder = StateGraph(ChatState)
builder.add_node("human_step", human_step)
builder.add_node("ai_step", ai_step)
builder.add_edge(START, "human_step")
builder.add_edge("human_step", "ai_step")
builder.add_edge("ai_step", END)
graph = builder.compile()

result = graph.invoke({"messages": []})
for m in result["messages"]:
    print(f"{m.type}: {m.content}")
# Expected output:
# human: Hello!
# ai: Hi, how can I help?
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Chatbot/agent တိုင်းမှာ history ဆက်ထိန်းထားရပါတယ်။ `add_messages` သုံးခြင်းအားဖြင့် multi-turn memory၊ tool message တွဲဖက်ခြင်း၊ streaming update တွေ အလိုအလျောက်ရရှိပါတယ်။ ဒါက LangGraph prebuilt library (`create_react_agent` တို့) ရဲ့ အခြေခံလည်း ဖြစ်ပါတယ်။

## ၃။ Channel Semantics နှင့် Partial State Updates

### ဘာကို ဆိုလိုတာလဲ
LangGraph state ရဲ့ field တစ်ခုစီက "channel" တစ်ခု ဖြစ်ပါတယ်။ Channel တစ်ခုစီမှာ value တစ်ခုပိုင်းသီးသီးသက်သက် သိမ်းဆည်းပြီး၊ update rule က field တစ်ခုနဲ့တစ်ခု သီးသန့်ဖြစ်ပါတယ်။ Node တစ်ခုက state အားလုံး မပြန်ရဘဲ ပြင်ချင်တဲ့ key တွေပဲ ပါတဲ့ dict ကို ပြန်လို့ရပါတယ် — ဒါကို partial state update လို့ ခေါ်ပါတယ်။

### ဘာကြောင့် လဲ
Node တိုင်းက state အားလုံးကို ပြန်ရေးရမယ်ဆိုရင် (၁) ကုဒ်ရှည်ပြီး (၂) မသင့်တော်တဲ့ field ကို မှားယွင်းစဉ်းစားမိတာ (ဥပမာ အခြား channel ထဲ့က ဒေတာကို မတော်တဆ overwrite) ဖြစ်စေပါတယ်။ Partial update က node တစ်ခုက ဘယ် channel တွေကို တာဝန်ယူလဲဆိုတာကို ထင်ရှားစေပြီး parallel execution မှာလည်း အရေးကြီးပါတယ် — parallel branch နှစ်ခုက မတူတဲ့ key တွေကို ပြင်ရင် စနစ်တကျ ပေါင်းနိုင်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
Node ရဲ့ return value က dict ဖြစ်ပြီး ထဲမှာ ပါတဲ့ key တွေက update ခံမဲ့ channel တွေပါ။ Reducer ရှိတဲ့ channel ဖြစ်ရင် reducer နဲ့ merge လုပ်၊ reducer မရှိရင် value အသစ်နဲ့ overwrite လုပ်ပါတယ်။ Return မလုပ်တဲ့ key တွေက မထိခိုက်ပါဘူး။ တစ်ကြောင်းရှိမှန်း သတိပြုရတာက parallel super-step မှာ channel တစ်ခုတည်းကို node တို့ကိုယ်စားမတူတဲ့ value နဲ့ ရေးချင်တာနဲ့ တိုက်ဆိုင်နေရင် `InvalidUpdateError` ရနိုင်ပါတယ် (last-value channel အတွက်)။

### ဥပမာ

```python
from typing_extensions import TypedDict, Annotated
import operator
from langgraph.graph import StateGraph, START, END

class PartialState(TypedDict):
    query: str                                  # no reducer: overwrite on update
    findings: Annotated[list, operator.add]     # reducer: list concatenation
    count: Annotated[int, operator.add]         # reducer: numeric addition

def node_a(state: PartialState) -> dict:
    # Partial update: only "findings" and "count" are touched, "query" untouched
    return {"findings": ["relevant doc 1"], "count": 1}

def node_b(state: PartialState) -> dict:
    return {"query": state["query"] + "?", "findings": ["relevant doc 2"]}

builder = StateGraph(PartialState)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", node_b)
builder.add_edge("node_b", END)
graph = builder.compile()

result = graph.invoke({"query": "langgraph", "findings": [], "count": 0})
print(result)
# Expected output:
# {'query': 'langgraph?', 'findings': ['relevant doc 1', 'relevant doc 2'], 'count': 1}
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Reducer မသတ်မှတ်တာနဲ့ `Annotated[..., operator.add]` သတ်မှတ်တာ နှစ်ခုကြားထဲမှာ behavior လုံးဝမတူပါဘူး။ Parallel branches (map-reduce pattern တို့) မှာ results တွေကို overwrite မခံဘဲ စုစည်းချင်ရင် reducer သတ်မှတ်ဖို့ မဖြစ်မနေလိုအပ်ပါတယ်။ ဒါမသိရင် branch တစ်ခုရဲ့ output က အခြား branch output ကို အတိတ်ကွက် ဖျက်တာမျိုး ဖြစ်ပါတယ်။

## ၄။ State Size နှင့် Cost

### ဘာကို ဆိုလိုတာလဲ
State size ဆိုတာ graph တစ်ခု super-step တစ်ခုမှာ သိမ်းဆည်းရတဲ့ state ရဲ့ အရွယ်အစားကို ဆိုလိုပါတယ်။ Messages list ရှည်လာရင်၊ ကြီးမားတဲ့ object တွေ state ထဲ ထည့်လာရင် state size ကြီးလာပါတယ်။ Cost က (၁) checkpoint storage အတွက် ဒေတာစရိတ်၊ (၂) LLM context window ထဲ ဒေတာတွေပါဝင်တဲ့ token ကုန်ကျစရိတ် နှစ်မျိုးလုံး ဆိုလိုပါတယ်။

### ဘာကြောင့် လဲ
LangGraph မှာ state တစ်ခုလုံးကို super-step တိုင်းမှာ checkpoint သိမ်းတာ (memory ဒါမှမဟုတ် external checkpointer) ရှိနိုင်ပါတယ်။ Messages တွေ မရပ်နိုင်ဘဲ တက်လာရင် checkpoint ကြီးလာပြီး latency နဲ့ storage ကုန်ကျစရိတ် တက်လာပါတယ်။ ပြီးရင် state ထဲက messages တွေကို LLM ဆီ တိုက်ရိုက်ပို့တဲ့ node မှာ token အရေအတွက် အတက်နဲ့တက် ကြီးလာပြီး API ကုန်ကျစရိတ် တက်လာပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
State ထဲ အသုံးမဝင်တော့တဲ့ ဒေတာ မထည့်တာ၊ ကြီးမားတဲ် raw data (ဥပမာ base64-encoded file) ကို state မှာ မသိမ်းဘဲ reference (file path/URL) တစ်ခုတည်း သိမ်းတာ၊ conversation ရှည်လာရင် `RemoveMessage` (ID ဖြင့် message ဖျက်ခြင်း) ဒါမှမဟုတ် `trim_messages` utility နဲ့ history ကို trim လုပ်တာတွေက အသုံးဝင်တဲ့ နည်းလမ်းတွေ ဖြစ်ပါတယ်။ Node တစ်ခုချင်းစီက state အားလုံး မလိုအပ်ဘူးဆိုရင် private input schema (`input` parameter) သတ်မှတ်ပြီး လိုအပ်တဲ့ field တွေပဲ လက်ခံအလုပ်လုပ်စေနိုင်ပါတယ်။

### ဥပမာ

```python
from langchain_core.messages import AIMessage, HumanMessage, RemoveMessage
from langgraph.graph import StateGraph, MessagesState, START, END


# Define the state schema using MessagesState, which already includes
# the "messages" key with the built-in add_messages reducer.
class ChatState(MessagesState):
    pass


def trim_messages(state: ChatState):
    """Keep only the most recent 2 messages in the conversation history."""
    # If there are more than 2 messages, delete the oldest ones
    if len(state["messages"]) > 2:
        # Build a list of RemoveMessage instances targeting the oldest ids
        removals = [
            RemoveMessage(id=m.id)
            for m in state["messages"][:-2]
        ]
        # Returning these inside "messages" lets the add_messages reducer
        # interpret them as deletions rather than appends.
        return {"messages": removals}
    # Nothing to trim; return an empty update (no change to state)
    return {"messages": []}


def chatbot(state: ChatState):
    # Simple echo-style reply so the example stays runnable without an API key
    last = state["messages"][-1]
    reply = AIMessage(content=f"Echo: {last.content}")
    return {"messages": [reply]}


# Build the graph
builder = StateGraph(ChatState)
builder.add_node("trimmer", trim_messages)
builder.add_node("chatbot", chatbot)
builder.add_edge(START, "trimmer")
builder.add_edge("trimmer", "chatbot")
builder.add_edge("chatbot", END)

graph = builder.compile()

# Send three messages one at a time so trimming actually happens
result1 = graph.invoke({"messages": [HumanMessage(content="Hi")]})
result2 = graph.invoke({"messages": result1["messages"] + [HumanMessage(content="How are you?")]})
result3 = graph.invoke({"messages": result2["messages"] + [HumanMessage(content="Tell me about LangGraph")]})

# Print the ids and contents that survive after the final run
for m in result3["messages"]:
    print(f"{m.id}: {m.__class__.__name__} -> {m.content}")

# Expected output:
# <uuid 2>: HumanMessage -> How are you?
# <uuid 3>: AIMessage -> Echo: How are you?
# <uuid 4>: HumanMessage -> Tell me about LangGraph
# <uuid 5>: AIMessage -> Echo: Tell me about LangGraph
```

## အနှစ်ချုပ်

- State schema ဆိုသည်မှာ graph ၏ data structure ကို သတ်မှတ်ပေးသော `TypedDict` ဖြစ်ပြီး၊ LangGraph က ၎င်းကို မှတ်ယူပြီး nodes အချင်းချင်း တွင် မျှဝေသည်။
- Reducer ဆိုသည်မှာ state key တစ်ခုအတွင်း တန်ဖိုးအသစ်နှင့် တန်ဖိုးဟောင်းကို ဘယ်လို ပေါင်းစည်းမလဲဆိုသည်ကို သတ်မှတ်သော function ဖြစ်ပြီး၊ `add_messages` reducer က message အသစ်များကို ထည့်သွင်းပေးပြီး `RemoveMessage` ကို တွေ့လျှင် ဖျက်ပစ်သည်။
- Message များ အလွန်အကျွံ စုဆောင်းမိလျှင် context window ကျော်လွန်ပြီး API ကုန်ကျစရိတ် တက်သွားပြီး၊ ရှည်လျားသော history ကြောင့် model ၏ တုံ့ပြန်မှု အရည်အသွေး ကျဆင်းနိုင်သည်။
- ထို့ကြောင့် အရေးကြီးသော နောက်ဆုံး message များကိုသာ ထားရှိစေဖို့ argeis trimming node တစ်ခု ထည့်သွင်းခြင်းဖြင့် ကုန်ကျစရိတ် သက်သာပြီး graph ၏ လုပ်ဆောင်မှုကို မြန်ဆန်စေသည်။
