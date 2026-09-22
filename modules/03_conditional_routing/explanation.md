# Conditional Routing, Loops & Termination

## ၁။ Conditional Edges နဲ့ Routing Functions

### ဘာကို ဆိုလိုတာလဲ

Conditional edge ဆိုတာ graph ရဲ့ နောက်ထပ် node ဘယ်ဟာဖြစ်မလဲကို runtime မှာ ဆုံးဖြတ်ပေးတဲ့ edge အမျိုးအစားပဲ ဖြစ်ပါတယ်။ Static edge (`add_edge`) က node အမြဲတမ်း တစ်ခုတည်းကိုပဲ သွားပေမယ့် conditional edge က state ပေါ်မူတည်ပြီး လမ်းကြောင်း ပြောင်းလဲပေးပါတယ်။

### ဘာကြောင့် လဲ

Agent workflow တိုင်းမှာ အခြေအနေအမျိုးမျိုးရှိပါတယ် — အောင်မြင်တာ၊ ပျက်တာ၊ ထပ်စစ်ရမယ့်အခါတွေ။ ဒီအခြေအနေတွေကို ကြိုတင်မသိနိုင်လို့ code ထဲ မှာ hardcoded လုပ်လို့ မရပါဘူး။ State အခြေအခံနဲ့ ဆုံးဖြတ်ရမှာ ဖြစ်လို့ conditional edge လိုအပ်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

`add_conditional_edges` မှာ routing function (router) တစ်ခုပေးရပါတယ်။ Router က state လက်ခံပြီး နောက် node ရဲ့ နာမည် (string) တစ်ခု return ပါတယ်။ LangGraph က return တဲ့ နာမည်အတိုင်း edge ကို လိုက်သွားပါတယ်။ `path_map` နဲ့ return value တွေကို node နာမည်တွေနဲ့ ချိတ်ဆက်နိုင်ပါတယ်။

### ဥပမာ

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    query: str
    category: str

def classify_node(state: State) -> dict:
    # Simple keyword-based classification for demonstration
    query = state["query"].lower()
    if "price" in query or "buy" in query:
        return {"category": "sales"}
    return {"category": "support"}

def route_by_category(state: State) -> str:
    # Router: return the next node name based on state
    return state["category"]

def sales_node(state: State) -> dict:
    return {"query": state["query"] + " [handled by sales]"}

def support_node(state: State) -> dict:
    return {"query": state["query"] + " [handled by support]"}

builder = StateGraph(State)
builder.add_node("classify", classify_node)
builder.add_node("sales", sales_node)
builder.add_node("support", support_node)
builder.add_edge(START, "classify")
builder.add_conditional_edges("classify", route_by_category)
builder.add_edge("sales", END)
builder.add_edge("support", END)

graph = builder.compile()
result = graph.invoke({"query": "what is the price?"})
print(result["query"])
# Expected output: what is the price? [handled by sales]
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Production agent တစ်ခုမှာ tool ဘယ်နားကို ခေါ်မလဲ၊ လုပ်ငန်းပြီးရင် END ကို ရောက်မလဲ ဆိုတာကို အခြေအခံ ဆုံးဖြတ်ရတဲ့အတွက် conditional edges က agent engineering ရဲ့ နှလုံးသားပဲ ဖြစ်ပါတယ်။ ဒါမှတော့ system က မတူညီတဲ့ input တွေကို တူညီတဲ့ hardcoded လမ်းကြောင်းနဲ့ မသွားတော့ပါဘူး။

## ၂။ Cycles — Retry နဲ့ Refine Loops

### ဘာကို ဆိုလိုတာလဲ

Cycle ဆိုတာ graph ထဲမှာ node တစ်ခု (သို့) edge တစ်ခုက နောက်ကျရင် ကိုယ်တိုင်ကိုယ့်ကို သော်လည်းကောင်း၊ အရင် node တစ်ခုဆီ ပြန်loop လုပ်တာပဲ ဖြစ်ပါတယ်။ LangGraph က directed graph ဖြစ်လို့ DAG မဟုတ်ဘူး — cycle တွေ ခွင့်ပြုပါတယ်။

### ဘာကြောင့် လဲ

LLM output တွေက တစ်ခါတစ်ရံ မှားတတ်ပါတယ် — JSON format ပျက်တာ၊ အဖြေမှားတာ၊ validation မပြတာ။ ဒီအခါ အလိုအလျောက် ထပ်စမ်း (retry) စေချင်ရင် သို့မဟုတ် ရလဒ်ကို အဆင့်ဆင်း refine ချင်ရင် cycle က သဘာဝကျတဲ့ နည်းလမ်းပဲ ဖြစ်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

Node တစ်ခုက ရလဒ်ကို state ထဲမှာ မှတ်၊ router က အောင်မြင်/မအောင်မြင် စစ်ပြီး — မအောင်ရင် အရင် node ဆီပြန်၊ အောင်ရင် `END` ဆီ သွားစေပါတယ်။ ဒီပုံစံက graph ကို loop ထဲ ထည့်ပေးပါတယ်။

### ဥပမာ

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    answer: str
    attempts: int
    is_valid: bool

VALID_WORDS = {"correct", "good"}

def generate_node(state: State) -> dict:
    # Simulated generator that fails until the third attempt
    attempts = state["attempts"] + 1
    if attempts >= 3:
        return {"answer": "correct answer", "attempts": attempts, "is_valid": True}
    return {"answer": "bad answer", "attempts": attempts, "is_valid": False}

def check_node(state: State) -> dict:
    # Validation: pass the state through unchanged (check happens in router)
    return {}

def route_check(state: State) -> str:
    # Loop back to "generate" while invalid, otherwise finish
    if state["is_valid"]:
        return END
    return "generate"

builder = StateGraph(State)
builder.add_node("generate", generate_node)
builder.add_node("check", check_node)
builder.add_edge(START, "generate")
builder.add_edge("generate", "check")
builder.add_conditional_edges("check", route_check, ["generate", END])
graph = builder.compile()

result = graph.invoke({"answer": "", "attempts": 0, "is_valid": False})
print(result["answer"], "attempts:", result["attempts"])
# Expected output: correct answer attempts: 3
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Retry/refine loops က agent တွေရဲ့ ယုံကြည်စိတ်ချမှု တိုးစေပါတယ် — validation မပြတဲ့ output တွေကို ကြိုးပမ်းနှစ်ခေါက် လုပ်ပေးလို့ရပါတယ်။ ဒါပေမယ့် cycle တွေက infinite loop ဖြစ်စေနိုင်လို့ နောက် section မှာ သင့်တင့်စွာ ထိန်းချုပ်ရပါမယ်။

## ၃။ recursion_limit နဲ့ Guardrails

### ဘာကို ဆိုလိုတာလဲ

`recursion_limit` က graph run တစ်ခုမှာ super-step (node execution) အရေအတွက် အမြင့်ဆုံး ကန့်သတ်ချက်ပဲ ဖြစ်ပါတယ်။ ကန့်သတ်ချက်ကို ကျော်သွားရင် `GraphRecursionError` ပေါ်ပါတယ်။ Default က 25 ဖြစ်ပါတယ်။

### ဘာကြောင့် လဲ

LLM agent တစ်ခုက ဘယ်တုန်းကမှ အောင်မြင်မှု မရနိုင်တဲ့အခါ router က အမြဲ loop ပတ်နေတာ ရှိနိုင်ပါတယ်။ ဒီလို infinite loop က API cost တွေ၊ time တွေ၊ resource တွေ ပျက်ဆီးစေလို့ production မှာ ကာကွယ်ရပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

`graph.invoke(state, config={"recursion_limit": 10})` ဆိုပြီး config ထဲ ပေးရပါတယ်။ ဒါအပြင် ကိုယ်ပိုင် counter (ဥပမာ — `attempts`) နဲ့ ကိုယ်တိုင်လည်း loop ကို ကန့်သတ်နိုင်ပါတယ်။ နှစ်မျိုးလုံး တွဲသုံးတာက အလုံခြုံဆုံး နည်းလမ်းပဲ ဖြစ်ပါတယ်။

### ဥပမာ

```python
from langgraph.graph import StateGraph, START, END
from langgraph.errors import GraphRecursionError
from typing import TypedDict

class State(TypedDict):
    value: int

def loop_node(state: State) -> dict:
    # This node never satisfies the router, so the loop never ends
    return {"value": state["value"] + 1}

def route(state: State) -> str:
    # Never returns END -> infinite loop without a limit
    return "loop_node"

builder = StateGraph(State)
builder.add_node("loop_node", loop_node)
builder.add_edge(START, "loop_node")
builder.add_conditional_edges("loop_node", route, [END])
graph = builder.compile()

try:
    graph.invoke({"value": 0}, config={"recursion_limit": 10})
except GraphRecursionError:
    print("stopped: recursion limit reached")
# Expected output: stopped: recursion limit reached
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Retry loop တိုင်းမှာ `recursion_limit` ထားတာက စျေးကြီးတဲ့ LLM call တွေ အလွန်အကင်း မဖြစ်စေဘဲ၊ error ကိုလည်း သိသိသာသာ catch လုပ်ပြီး fallback လုပ်နိုင်စေပါတယ်။ Counter-based guardrail ကလည်း loop တစ်ခုချင်းကို ပိုတိကျစွာ ထိန်းချုပ်ပေးပါတယ်။

## ၄။ Termination Conditions နဲ့ Deterministic Fallbacks

### ဘာကို ဆိုလိုတာလဲ

Termination condition ဆိုတာ graph က loop ကနေ ထွက်ပြီး `END` မှာ ရပ်တန့်ရမလဲ ဆုံးဖြတ်တဲ့ စည်းမျဉ်းပဲ ဖြစ်ပါတယ်။ Deterministic fallback က termination မအောင်မြင်ရင် ဘာမှမှန်းမသိတဲ့ LLM ဆုံးဖြတ်ချက်အစား သေချာတဲ့ code logic နဲ့ ရလဒ်တစ်ခု ပြန်ပေးတာပဲ ဖြစ်ပါတယ်။

### ဘာကြောင့် လဲ

Retry အကြိမ်ကြောင့် ကုန်သွားရင် user ကို error ချည်း မပြဘဲ သေချာတဲ့ အနည်းငယ် ရလဒ်တစ်ခု ပြန်ပေးတာက UX အတွက် ပိုကောင်းပါတယ်။ ဥပမာ — LLM က summary မရနိုင်ရင် မူရင်းစာသားရဲ့ ပထမစာကြောင်းကို ပြန်ပြခြင်းနဲ့ fallback ပြုလုပ်နိုင်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ

Router ထဲမှာ `attempts >= max_attempts` ဆိုတဲ့ စည်းမျဉ်းနဲ့ `END` (သို့) fallback node ဆီ သွားစေပြီး၊ `GraphRecursionError` ကို `try/except` နဲ့ catch လုပ်ပြီး default ရလဒ် ထုတ်ပေးနိုင်ပါတယ်။

### ဥပမာ

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    text: str
    attempts: int
    final: str

MAX_ATTEMPTS = 3

def refine_node(state: State) -> dict:
    # Always returns a "low quality" result in this simulation
    attempts = state["attempts"] + 1
    return {"attempts": attempts, "text": state["text"] + " (refined)"}

def route(state: State) -> str:
    # Give up after MAX_ATTEMPTS and use the deterministic fallback
    if state["attempts"] >= MAX_ATTEMPTS:
        return "fallback"
    return "refine"

def fallback_node(state: State) -> dict:
    # Deterministic fallback: use the first line of the original text
    first_line = state["text"].split("\n")[0]
    return {"final": first_line}

builder = StateGraph(State)
builder.add_node("refine", refine_node)
builder.add_node("fallback", fallback_node)
builder.add_edge(START, "refine")
builder.add_conditional_edges("refine", route, ["refine", "fallback"])
builder.add_edge("fallback", END)
graph = builder.compile()

result = graph.invoke({"text": "line one\nline two", "attempts": 0, "final": ""})
print("final:", result["final"])
print("attempts:", result["attempts"])
# Expected output:
# final: line one
# attempts: 3
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Production system တိုင်းမှာ "အကုန်အကြောင်း အောင်မြင်ရမယ်" ဆိုတဲ့ ယူဆချက် မလုံခြုံပါဘူး။ Termination condition သေချာ၊ fallback သေချာရင် system က အဆိုးဆုံးအခြေအနေမှာတောင် မှန်ကန်တဲ့ ရလဒ်တစ်ခု ပြန်ပေးနိုင်ပါတယ်။

## အနှစ်ချုပ်

Conditional edges က state အခြေခံ လမ်းကြောင်း ရွေးပေးပြီး၊ cycles က retry/refine လုပ်ပေးပါတယ်။ `recursion_limit` နဲ့ ကိုယ်ပိုင် counter guardrails တွေက infinite loop ကာကွယ်ပြီး၊ termination conditions နဲ့ deterministic fallbacks က system ကို ဘေးအန္တရာယ်ကင်းစွာ ရပ်တန့်စေပါတယ်။ ဒီလေးခု တွဲထားရင် production-ready agent loop တစ်ခု ရပါတယ်။
