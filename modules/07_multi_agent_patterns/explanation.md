# Multi-Agent Patterns အသေးစိတ်ရှင်းလင်းချက်

## ၁။ ဘာကြောင့် agent တစ်ခု မလုံလောက်တာလဲ

### ဘာကို ဆိုလိုတာလဲ

Agent တစ်ခုတည်းက system prompt တစ်ခု၊ tool set တစ်ခုနဲ့ အလုပ်လုပ်တယ်။ Multi-agent system က agent များစွာကို ချိတ်ဆက်ပြီး တစ်ခုစီက ကဏ္ဍတစ်ခုကို တာဝန်ယူတယ်။

### ဘာကြောင့် လဲ

Tool များစွာ တစ်ခုတည်းရဲ့ system မှာ ဆွဲထည့်ရင် — prompt ရှည်လာတယ်၊ LLM က tool မှားရွေတတ်တယ်၊ context window ပြည့်တယ်။ တာဝန်ခွဲရင် တစ်ခုချင်းစီ context သေးသွားပြီး တိကျမှု တိုးတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

LangGraph မှာ agent တစ်ခုစီက graph တစ်ခုရဲ့ node (ဒါမှမဟုတ် subgraph) ဖြစ်တယ်။ သူတို့ကြားမှာ routing logic (conditional edge) က ဘယ် agent ကို ဆက်လှမ်းမလဲ ဆုံးဖြတ်တယ်။

### ဥပမာ

```python
from typing import Literal

# A trivial "agent" represented as a node function
def research_agent(state: dict) -> dict:
    # Each agent owns its own prompt and tools in a real system
    return {"result": f"researched: {state['task']}"}

def writer_agent(state: dict) -> dict:
    return {"result": f"written about: {state['task']}"}

def route(state: dict) -> Literal["research_agent", "writer_agent"]:
    # Decide which agent handles the task
    if "research" in state["task"]:
        return "research_agent"
    return "writer_agent"

print(route({"task": "research quantum computing"}))
# Expected output: research_agent
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Production မှာ စနစ်ကြီးလာတာနဲ့ agent တစ်ခုတည်းက ထိန်းလို့ မရတော့ပါ။ ခွဲခြမ်းတန်ခွဲရင် debugging၊ testing၊ cost ထိန်းခြင်း အားလုံး လွယ်လာတယ်။

## ၂။ Supervisor routing pattern

### ဘာကို ဆိုလိုတာလဲ

Supervisor ဆိုတာ agent တွေကို အုပ်ချုပ်တဲ့ central coordinator ဖြစ်တယ်။ သူက အလုပ်ကို ကြည့်ပြီး ဘယ် worker agent ကို အပ်မလဲ ဆုံးဖြတ်ပြီး ရလဒ်တွေကို ပြန်စုတယ်။

### ဘာကြောင့် လဲ

Worker agent တွေက တစ်ခုနဲ့တစ်ခု အချင်းချင်း ပြောစရာ မလိုဘူး — supervisor ကတစ်ကြိုးတည်း ကိုင်တယ်။ ဒါက အထူးသြဖြင့် တာဝန်အမျိုးမျိုး ကွဲနေတဲ့ အဖွဲ့အတွက် ထိန်းရလွယ်တယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

Supervisor node က LLM (ဒါမှမဟုတ် rule) နဲ့ နောက်ထပ် step ကို ရွေးတယ်။ ရွေးချက်က worker node တွေဆီ conditional edge ဖြစ်သွားတယ်။ Worker တွေပြီးရင် supervisor ကို ပြန်ပြေးပြီး ပြီးပြီလား ဆုံးဖြတ်တယ်။

### ဥပမာ

```python
from langgraph.graph import StateGraph, START, END

# Shared state that all agents read and write
def supervisor(state: dict) -> dict:
    if state["question"].startswith("code"):
        return {"next": "coder"}
    return {"next": "answerer"}

def coder(state: dict) -> dict:
    return {"messages": ["wrote code"]}

def answerer(state: dict) -> dict:
    return {"messages": ["answered question"]}

def route_after_supervisor(state: dict) -> str:
    return state["next"]

builder = StateGraph(dict)
builder.add_node("supervisor", supervisor)
builder.add_node("coder", coder)
builder.add_node("answerer", answerer)
builder.add_edge(START, "supervisor")
builder.add_conditional_edges("supervisor", route_after_supervisor)
builder.add_edge("coder", "supervisor")
builder.add_edge("answerer", END)

graph = builder.compile()
print(graph.invoke({"question": "code a fizzbuzz"}))
# Expected output: {'question': 'code a fizzbuzz', 'next': 'coder', 'messages': ['wrote code']}
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Customer support၊ coding assistants တို့မှာ supervisor က တာဝန်ကို အခွါအမျိုးမျိုးကို မှန်ကန်စွာ လမ်းပြနိုင်တယ်။ ဒါပေမယ့် hop တိုင်းက latency နဲ့ token တိုးစေတာကို သတိထားရမယ်။

## ၃။ Swarm / handoff pattern

### ဘာကို ဆိုလိုတာလဲ

Swarm မှာ central supervisor မရှိဘူး — agent တစ်ခုက ကိုယ်တိုင်ပဲ ဆက်သင့်တဲ့ agent ကို handoff လုပ်တယ်။

### ဘာကြောင့် လဲ

ကြားခံ မလိုတော့လို့ hop လျှော့တယ်၊ agent တွေက ပိုင်းခြားထားတဲ့ ကဏ္ဍကိုယ်စီနဲ့ လွတ်လွတ်လပ်လပ် အလုပ်လုပ်နိုင်တယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

Agent တစ်ခုစီမှာ handoff options ပါတယ်။ LLM က တာဝန်က ကိုယ့်ကဏ္ဍ မဟုတ်ဘူးဆိုရင် နောက် agent ကို ရွေးပြီး edge က အလိုအလျောက် လိုက်သွားတယ်။

### ဥပမာ

```python
from langgraph.graph import StateGraph, START, END

# Swarm: agents hand off directly, no central supervisor
def support_agent(state: dict) -> dict:
    if "billing" in state["message"]:
        return {"next": "billing_agent", "route": "handoff"}
    return {"next": "__end__", "route": "done", "answer": "general help"}

def billing_agent(state: dict) -> dict:
    return {"answer": "billing resolved", "route": "done"}

def route(state: dict) -> str:
    if state.get("route") == "handoff":
        return state["next"]
    return END

builder = StateGraph(dict)
builder.add_node("support_agent", support_agent)
builder.add_node("billing_agent", billing_agent)
builder.add_edge(START, "support_agent")
builder.add_conditional_edges("support_agent", route, ["billing_agent", END])
builder.add_edge("billing_agent", END)

graph = builder.compile()
print(graph.invoke({"message": "I have a billing problem"}))
# Expected output: {'message': 'I have a billing problem', 'next': 'billing_agent', 'route': 'handoff', 'answer': 'billing resolved'}
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Support bot တွေမှာ handoff က သူများဆီ ချက်ချင်း လွှဲလို့ရတယ်။ ဒါပေမယ့် routing loop တွေ (agent နှစ်ခုက တစ်ခုကိုတစ်ခု အဲဒီလိုပဲ ပြန်ပို့နေတာ) ကို ကာကွယ်ဖို့ recursion limit နဲ့ terminal condition ထားရမယ်။

## ၄။ Parallel fan-out / fan-in

### ဘာကို ဆိုလိုတာလဲ

Fan-out က အလုပ်တစ်ခုကို အပိုင်းတွေခွဲပြီး agent များစွာဆီ တစ်ပြိုင်တည်း ပို့တယ်။ Fan-in က ရလဒ်တွေအားလုံးကို ပြန်စုစည်းတယ်。

### ဘာကြောင့် လဲ

Independent ဖြစ်နေတဲ့ အလုပ်တွေကို တစ်ကြိမ်တည်းနဲ့ ဆွဲလို့ရရင် total latency က အချိန်အကြာဆုံး branch တစ်ခုလောက်ပဲ ကုန်တယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

LangGraph မှာ node တစ်ခုကဆီက edge များစွာ ထွက်ရင် သူတို့က super-step တစ်ခုထဲမှာ parallel အလုပ်လုပ်တယ်။ ရလဒ်တွေက state key တစ်ခုစီမှာ စုပြီး fan-in node က ဖတ်တယ်။ State reducer (e.g. `Annotated[list, operator.add]`) က တစ်ပြိုင်တည်း ရေးသည့် အချက်တွေကို ပေါင်းပေးတယ်။

### ဥပမာ

```python
import operator
from typing import Annotated
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    topic: str
    # Reducer merges results written in the same parallel step
    results: Annotated[list[str], operator.add]

def researcher_a(state: State) -> dict:
    return {"results": ["A: facts about " + state["topic"]]}

def researcher_b(state: State) -> dict:
    return {"results": ["B: history of " + state["topic"]]}

def summarize(state: State) -> dict:
    return {"summary": " | ".join(state["results"])}

builder = StateGraph(State)
builder.add_node("researcher_a", researcher_a)
builder.add_node("researcher_b", researcher_b)
builder.add_node("summarize", summarize)
builder.add_edge(START, "researcher_a")
builder.add_edge(START, "researcher_b")
builder.add_edge("researcher_a", "summarize")
builder.add_edge("researcher_b", "summarize")
builder.add_edge("summarize", END)

graph = builder.compile()
print(graph.invoke({"topic": "solar power", "results": []}))
# Expected output: {'topic': 'solar power', 'results': ['A: facts about solar power', 'B: history of solar power'], 'summary': 'A: facts about solar power | B: history of solar power'}
```

### လက်တွေ့မှာ ဘာကြောင်း အရေးကြီးလဲ

Research အပိုင်းများစွာ၊ document အများအပြား ခွဲစိတ်တာ၊ RAG ရဲ multi-query search တို့မှာ parallelism က အချိန်အများကြီး ချွေတာပေးတယ်။ Reducer မထားရင် parallel writes က conflict ဖြစ်တာကို သတိပြုရမယ်။

## ၅။ Subgraph များ၊ state isolation နဲ့ shared context

### ဘာကို ဆိုလိုတာလဲ

Subgraph က graph တစ်ခုအဖြစ် compile လုပ်ထားတဲ့ agent/component ကို node တစ်ခုလို အခြား graph ကြီးထဲ ထည့်သွင်းတာ။

### ဘာကြောင့် လဲ

Agent တစ်ခုစီက ကိုယ့် state schema ကိုယ်က သီးသန့်ထားချင်တယ် — parent က internal key တွေ မမြင်စေချင်ဘူး။ Subgraph က ဒီ isolation ကို သဘာဝကျစွာ ပေးတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

Subgraph ရဲ့ state နဲ့ parent state က တူရင် အလိုအလျောက် ချိတ်မိတယ်။ မတူရင် wrapper node က state ကို translate လုပ်ပေးရတယ် — parent key တွေကနေ subgraph input ဆောက်ပြီး၊ ရလဒ်ကို parent state ပြန်ရေးတယ်။

### ဥပမာ

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

# Child graph with its own private state schema
class ChildState(TypedDict):
    question: str
    answer: str

def child_node(state: ChildState) -> dict:
    return {"answer": "answer to: " + state["question"]}

child_builder = StateGraph(ChildState)
child_builder.add_node("child_node", child_node)
child_builder.add_edge(START, "child_node")
child_builder.add_edge("child_node", END)
child_graph = child_builder.compile()

# Parent state has a different shape; we translate explicitly
class ParentState(TypedDict):
    messages: list[str]
    final: str

def call_child(state: ParentState) -> dict:
    # Translate parent state into child state and back
    last_message = state["messages"][-1]
    result = child_graph.invoke({"question": last_message})
    return {"final": result["answer"]}

parent_builder = StateGraph(ParentState)
parent_builder.add_node("call_child", call_child)
parent_builder.add_edge(START, "call_child")
parent_builder.add_edge("call_child", END)
parent = parent_builder.compile()

print(parent.invoke({"messages": ["what is LangGraph?"], "final": ""}))
# Expected output: {'messages': ['what is LangGraph?'], 'final': 'answer to: what is LangGraph?'}
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Complex agent တွေကို Lego တုံးတွေလို ပြန်သုံးနိုင်တယ် — coding agent တစ်ခုကို project များစွာမှာ ထည့်လို့ရတယ်။ Shared context ကတော့ လိုအပ်ရင် (ဥပမာ အားလုံးက messages တွေ မြင်ရမယ်ဆိုရင်) parent state ကနေ တမင် ထည့်ပေးရတယ်။

## ၆။ Extra hop ကုန်ကျစရိတ်

### ဘာကို ဆိုလိုတာလဲ

Agent တွေကြား တစ်ဆင့်ပြီးတစ်ဆင့် ကူးရတိုင်း (hop) က LLM call၊ serialization နဲ့ context copying ထပ်ပေါင်းစေတယ်။

### ဘာကြောင့် လဲ

Hop တိုင်းမှာ context တစ်ဝက် ပြန်ရေးရတာ၊ coordinator က ရွေးရတာ — token cost နဲ့ latency နှစ်ခုစလုံး တက်တယ်။ Agent အရေအတွက် များလေ ဒီကုန်ကျ ကြီးလေပါပဲ။

### ဘယ်လို အလုပ်လုပ်လဲ

ကုန်ကျကို ထိန်းဖို့ — agent အရေအတွက် အနည်းဆုံးနဲ့ စမယ်၊ direct handoff သုံးပြီး supervisor ကြားခံလျှော့မယ်၊ မလိုအပ်တဲ့ context ကို မမျှေမျွေ ပို့မယ်၊ တိုက်ရိုက် ဖြေလို့ရတဲ့ အလုပ်ကို agent နဲ့ မခွဲမယ်။

### ဥပမာ

```python
def cost_of_hops(num_hops: int, tokens_per_hop: int = 2000) -> int:
    # Rough token cost grows linearly with number of hops
    return num_hops * tokens_per_hop

print("1 agent:", cost_of_hops(1))
print("supervisor + 3 workers:", cost_of_hops(4))
# Expected output:
# 1 agent: 2000
# supervisor + 3 workers: 8000
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Production မှာ over-engineering လုပ်ထားတဲ့ multi-agent က ရလဒ် မကောင်းစေဘူး — အငြင်းပွားမှု၊ loop နဲ့ cost ပိုတာတွေ ဖြစ်တတ်တယ်။ အရင်ဆုံး အလွန်ရိုးရိုးရှင်းရှင်း graph နဲ့ စစ်ပြီးမှ လိုအပ်ရင် ခွဲတာ မှန်တဲ့ ချဉ်းကပ်နည်းပါ။

## အနှစ်ချုပ်

Multi-agent pattern တွေက စနစ်ကြီးလာတဲ့အခါ လိုအပ်လာတယ် — supervisor က တစ်ကြိုးတည်း ထိန်းတယ်၊ swarm က ကြားခံလျှော့တယ်၊ parallel က latency ချတယ်၊ subgraph က ပြန်သုံးနိုင်တဲ့ component တွေပေးတယ်။ ဒါပေမယ့် hop တိုင်းက cost တက်စေလို့ လိုအပ်မှသာ layer ထပ်ဖို့ သတိနဲ့ ဆုံးဖြတ်ပါ။
