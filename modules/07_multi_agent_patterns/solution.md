# ဖြေရှင်းချက်များ

## လေ့ကျင့်ခန်း ၁ — Supervisor router အခြေခံ

```python
from langgraph.graph import StateGraph, START, END

def supervisor(state: dict) -> dict:
    # Choose the worker agent based on keywords in the question
    if "code" in state["question"] or "python" in state["question"]:
        return {"next": "tech_agent"}
    return {"next": "general_agent"}

def tech_agent(state: dict) -> dict:
    return {"answer": "technical answer for: " + state["question"]}

def general_agent(state: dict) -> dict:
    return {"answer": "general answer for: " + state["question"]}

def route(state: dict) -> str:
    return state["next"]

builder = StateGraph(dict)
builder.add_node("supervisor", supervisor)
builder.add_node("tech_agent", tech_agent)
builder.add_node("general_agent", general_agent)
builder.add_edge(START, "supervisor")
builder.add_conditional_edges("supervisor", route, ["tech_agent", "general_agent"])
builder.add_edge("tech_agent", END)
builder.add_edge("general_agent", END)

graph = builder.compile()
print(graph.invoke({"question": "how do I code a loop?"}))
print(graph.invoke({"question": "what is the weather?"}))
# Expected output:
# {'question': 'how do I code a loop?', 'next': 'tech_agent', 'answer': 'technical answer for: how do I code a loop?'}
# {'question': 'what is the weather?', 'next': 'general_agent', 'answer': 'general answer for: what is the weather?'}
```

**အဓိကအယူအဆ** — Supervisor က routing ဆုံးဖြတ်ချက်ကို state ထဲ သိမ်းပြီး conditional edge က အဲဒါကို လိုက်လုပ်တာပါ။

## လေ့ကျင့်ခန်း ၂ — Supervisor loop နဲ့ termination

```python
from langgraph.graph import StateGraph, START, END

def supervisor(state: dict) -> dict:
    # If a worker already produced an answer, finish the loop
    if state.get("answer"):
        return {"next": "__end__"}
    if "code" in state["question"]:
        return {"next": "tech_agent"}
    return {"next": "general_agent"}

def tech_agent(state: dict) -> dict:
    return {"answer": "tech done"}

def general_agent(state: dict) -> dict:
    return {"answer": "general done"}

def route(state: dict) -> str:
    if state["next"] == "__end__":
        return END
    return state["next"]

builder = StateGraph(dict)
builder.add_node("supervisor", supervisor)
builder.add_node("tech_agent", tech_agent)
builder.add_node("general_agent", general_agent)
builder.add_edge(START, "supervisor")
builder.add_conditional_edges("supervisor", route, ["tech_agent", "general_agent", END])
builder.add_edge("tech_agent", "supervisor")
builder.add_edge("general_agent", "supervisor")

graph = builder.compile()
print(graph.invoke({"question": "code a snake game"}))
# Expected output:
# {'question': 'code a snake game', 'next': '__end__', 'answer': 'tech done'}
```

**အဓိကအယူအဆ** — Supervisor loop ကို အလုပ်ပြီးတဲ့ အခြေအနေကို စစ်ပြီး END ကို ပြန်လမ်းပြမယ့် terminal condition နဲ့ ရပ်စေရမယ်။

## လေ့ကျင့်ခန်း ၃ — Swarm handoff

```python
from langgraph.graph import StateGraph, START, END

def front_agent(state: dict) -> dict:
    # Decide directly whether to hand off, without a supervisor
    if "order" in state["message"]:
        return {"next": "order_agent", "answer": ""}
    return {"next": "__end__", "answer": "hello, how can I help?"}

def order_agent(state: dict) -> dict:
    return {"answer": "order placed for " + state["message"]}

def route(state: dict) -> str:
    if state["next"] == "order_agent":
        return "order_agent"
    return END

builder = StateGraph(dict)
builder.add_node("front_agent", front_agent)
builder.add_node("order_agent", order_agent)
builder.add_edge(START, "front_agent")
builder.add_conditional_edges("front_agent", route, ["order_agent", END])
builder.add_edge("order_agent", END)

graph = builder.compile()
print(graph.invoke({"message": "I want to order pizza"}))
# Expected output:
# {'message': 'I want to order pizza', 'next': 'order_agent', 'answer': 'order placed for I want to order pizza'}
```

**အဓိကအယူအဆ** — Swarm မှာ agent ကိုယ်တိုင် handoff ဆုံးဖြတ်တာကြောင့် ကြားခံ hop လျှော့ပြီး latency သက်သာစေတယ်။

## လေ့ကျင့်ခန်း ၄ — Parallel fan-out နဲ့ fan-in

ဤလေ့ကျင့်ခန်းမှာ `pro_agent` နဲ့ `con_agent` ဆီ topic တစ်ခုတည်းကို START ကနေ edge နှစ်ခုနဲ့ တစ်ပြိုင်တည်း ပို့ပြီး၊ သူတို့ရဲ့ ရလဒ်နှစ်ခုကို `judge_agent` က reducer pattern သုံးပြီး ပေါင်းစပ်စိစစ်ပါမယ်။ State ထဲက `results` key ကို `Annotated[list, operator.add]` နဲ့ ကြေညာထားတာကြောင့် parallel nodes နှစ်ခုက တစ်ပြိုင်တည်း ရေးသည့်အခါ conflict မဖြစ်ပဲ list နှစ်ခု အလိုအလျောက် ပေါင်းစပ်သွားပါမယ်။ LangGraph က `pro_agent` တက်ထားတဲ့ super-step တစ်ခုတည်းမှာ node နှစ်ခုလုံးကို အလုပ်လုပ်ခိုင်းပြီး၊ နှစ်ခုလုံးပြီးမှ `judge_agent` ကို ဆက်ဖို့ပေးလိုက်ပါမယ်။

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END


# Define the shared state; "results" uses operator.add so parallel
# writes from pro_agent and con_agent are merged instead of overwritten.
class State(TypedDict):
    topic: str
    results: Annotated[list, operator.add]
    verdict: str


# Fan-out node 1: argues in favor of the topic.
def pro_agent(state: State) -> dict:
    result = f"PRO: {state['topic']} is promising because demand keeps growing."
    print(f"[pro_agent] produced: {result}")
    return {"results": [result]}


# Fan-out node 2: argues against the topic.
def con_agent(state: State) -> dict:
    result = f"CON: {state['topic']} is risky because costs are hard to predict."
    print(f"[con_agent] produced: {result}")
    return {"results": [result]}


# Fan-in node: judge reads all collected results and builds a verdict.
def judge_agent(state: State) -> dict:
    collected = state["results"]
    verdict = "VERDICT on '" + state["topic"] + "' -> " + " | ".join(collected)
    print(f"[judge_agent] verdict: {verdict}")
    return {"verdict": verdict}


# Build the graph: START fans out to both agents, both converge on judge.
builder = StateGraph(State)
builder.add_node("pro_agent", pro_agent)
builder.add_node("con_agent", con_agent)
builder.add_node("judge_agent", judge_agent)

# Fan-out: START connects to both agents in the same super-step.
builder.add_edge(START, "pro_agent")
builder.add_edge(START, "con_agent")

# Fan-in: both agents flow into the judge node.
builder.add_edge("pro_agent", "judge_agent")
builder.add_edge("con_agent", "judge_agent")
builder.add_edge("judge_agent", END)

graph = builder.compile()

# Run the graph with a sample topic.
final_state = graph.invoke(
    {"topic": "building an AI startup", "results": []}
)

print("Final verdict:", final_state["verdict"])
```

**အဓိကအယူအဆ** — `Annotated[list, operator.add]` ဆိုတဲ့ reducer ကို state ထဲ သတ်မှတ်ပေးခြင်းအားဖြင့် parallel nodes နှစ်ခုက တစ်ပြိုင်တည်း ရေးသည့်အခါ ရလဒ်တွေ overwrite မဖြစ်ပဲ ပေါင်းလိုက်ပြီး fan-in node ဖြစ်တဲ့ judge ဆီ အပြည့်အဝ ရောက်သွားစေသည်။

## လေ့ကျင့်ခန်း ၅ — Subgraph နဲ့ state translation

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict


# Child graph state uses a different schema than the parent.
class ChildState(TypedDict):
    question: str


# Parent graph state holds a list of messages.
class ParentState(TypedDict):
    messages: list[str]


def answer_node(state: ChildState) -> dict:
    # Simple child node that processes the question string.
    question = state["question"]
    return {"question": f"Answer to: {question}"}


# Build the child graph with its own state schema.
child_builder = StateGraph(ChildState)
child_builder.add_node("answer", answer_node)
child_builder.add_edge(START, "answer")
child_builder.add_edge("answer", END)
child_graph = child_builder.compile()


def child_wrapper(state: ParentState) -> dict:
    # Translate parent state -> child input.
    last_message = state["messages"][-1] if state["messages"] else ""
    child_input = {"question": last_message}

    # Run the child graph invisibly inside this wrapper node.
    child_result = child_graph.invoke(child_input)

    # Translate child result -> parent state update.
    return {"messages": state["messages"] + [child_result["question"]]}


# Build the parent graph.
parent_builder = StateGraph(ParentState)
parent_builder.add_node("child_wrapper", child_wrapper)
parent_builder.add_edge(START, "child_wrapper")
parent_builder.add_edge("child_wrapper", END)
parent_graph = parent_builder.compile()


if __name__ == "__main__":
    result = parent_graph.invoke({"messages": ["What is LangGraph?"]})
    print(result)
```

**အဓိကအယူအဆ** — State schema မတူတဲ့ child graph ကို wrapper node function တစ်ခုနဲ့ ချိတ်ဆက်ပြီး parent state ကနေ child input ဆောက်ကာ ရလဒ်ကို parent state ထဲ ပြန်ဘွဲ့ပေးရုံမျှသာရှိလျှင် subgraph က လှိုင်းလျှို့မြင်မသိ run သွားနိုင်ပါသည်။

## လေ့ကျင့်ခန်း ၆ — Hop cost တွက်ချက်ခြင်း

```python
def estimate_tokens(num_agents, supervisor_hops):
    # Each hop costs 2000 tokens
    TOKENS_PER_HOP = 2000

    # Supervisor pattern: 1 routing hop + 1 agent hop per agent
    supervisor_total_hops = num_agents * 2
    supervisor_tokens = supervisor_total_hops * TOKENS_PER_HOP

    # Direct handoff pattern: only agent hops
    handoff_total_hops = num_agents
    handoff_tokens = handoff_total_hops * TOKENS_PER_HOP

    # Difference between the two patterns
    difference = supervisor_tokens - handoff_tokens

    return {
        "supervisor_tokens": supervisor_tokens,
        "handoff_tokens": handoff_tokens,
        "difference": difference,
    }


result = estimate_tokens(3, None)
print("Supervisor pattern tokens:", result["supervisor_tokens"])
print("Direct handoff pattern tokens:", result["handoff_tokens"])
print("Difference (numeric):", result["difference"])
```

**အဓိကအယူအဆ** — Supervisor pattern တွင် agent တစ်ခုချင်းစီအတွက် လမ်းပြ hop နှင့် agent hop ဟူ၍ hop နှစ်ခုစီကျသဖြင့် direct handoff pattern ထက် token cost ကိုးသောင်းပိုမိုကုန်ကျသည်။
