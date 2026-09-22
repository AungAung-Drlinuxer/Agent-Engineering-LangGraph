# LangGraph Foundations — Graphs, Nodes, State

## ၁။ Agent logic အတွက် while-loop ထက် graph က ဘာကြောင့် ပိုကောင်းလဲ

### ဘာကို ဆိုလိုတာလဲ

Agent logic ကို `while` loop နှင့် ရေးနိုင်သည် — LLM ကို ခေါ်၊ tool ခေါ်၊ ပြန် loop။ LangGraph မှာတော့ ဒီ workflow ကို node နှင့် edge များဖြင့် ဖော်ပြသော directed graph အဖြစ် ရေးသည်။ Graph ဆိုတာ logic ကို ပုံသဏ္ဌာန်တကျ ဖော်ပြသော ဖွဲ့စည်းပုံတစ်ခုဖြစ်သည်။

### ဘာကြောင့် လဲ

`while` loop တွင် ဘယ်အဆင့်က ဘယ်အဆင့်ကို ခေါ်သည်ဆိုတာကို code ကြားထဲကမှ ဖတ်ရသည်။ Loop များထပ်နေပါက ခြေရာခံရခက်သည်။ Graph ဖြင့်ရေးလျှင် အဆင့်တိုင်းရဲ့ ဆက်သွယ်မှုကို မြင်သာစွာ ဖော်ပြနိုင်ပြီး၊ state တိုင်းကို checkpoint မှတ်နိုင်ပြီး run တစ်ခုချင်းစီကို ပြန်စစ်နိုင်သည်။

### ဘယ်လို အလုပ်လုပ်လဲ

Node တစ်ခုစီက function တစ်ခုဖြစ်ပြီး state ကို လက်ခံကာ မွမ်းမံပြီး ပြန်ပေးသည်။ Edge က node များအကြား သွားရမည့်လမ်းကြောင်းကို သတ်မှတ်သည်။ `START` မှ `END` ထိ graph တစ်ခုလုံး သတ်မှတ်ပြီး `compile()` ဖြင့် run နိုင်သော object ဖြစ်စေသည်။

### ဥပမာ

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

# Define the shared state schema for the graph
class State(TypedDict):
    question: str
    answer: str

# A node is a plain function that takes state and returns a partial update
def generate(state: State):
    return {"answer": f"You asked: {state['question']}"}

# Build the graph: one node, one edge from START to that node to END
graph = StateGraph(State)
graph.add_node("generate", generate)
graph.add_edge(START, "generate")
graph.add_edge("generate", END)

app = graph.compile()
result = app.invoke({"question": "What is LangGraph?"})
print(result["answer"])
# Expected output: You asked: What is LangGraph?
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Agent တစ်ခုကို production မှာ တင်သောအခါ — debug လုပ်ရခြင်း၊ checkpoint ထည့်ခြင်း၊ run ကို ပြန်စစ်ခြင်းတို့ လိုအပ်သည်။ Graph ဖွဲ့စည်းပုံက ဒီအလုပ်တွေကို ပိုမှုရင်းရှားစေသည်။

## ၂။ StateGraph, Nodes, compile() — အသုံးပြုနည်း

### ဘာကို ဆိုလိုတာလဲ

`StateGraph` ဆိုတာ state schema တစ်ခုကို အခြေခံပြီး workflow ဖော်ပြရန် အသုံးပြုသော builder class ဖြစ်သည်။ State schema က node အားလုံး မျှဝေသုံးသော dictionary ပုံစံကို သတ်မှတ်ပေးသည်။

### ဘာကြောင့် လဲ

Node တိုင်းက state ကို ဖတ်ပြီး အစိတ်အပိုင်းတစ်ခုကို ပြန်မွမ်းမံသည်။ Schema သတ်မှတ်ထားလျှင် state ထဲ ဘာ key များ ရှိရမလဲဆိုတာ ထင်ရှားပြီး type checker များနှင့်လည်း လိုက်ဖက်သည်။

### ဘယ်လို အလုပ်လုပ်လဲ

ပထမ `TypedDict` သို့မဟုတ် Pydantic model ဖြင့် state သတ်မှတ်သည်။ ထို့နောက် `add_node()` ဖြင့် function များထည့်၊ `add_edge()` ဖြင့် ဆက်သွယ်၊ `add_conditional_edges()` ဖြင့် state ပေါ် မူတည်ပြီး လမ်းကြောင်း ရွေးနိုင်သည်။ နောက်ဆုံး `compile()` ဖြင့် runnable object ရသည်။

### ဥပမာ

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

# State includes a list of steps so both nodes can read and update it
class State(TypedDict):
    text: str
    steps: list[str]

def step_one(state: State):
    # Return only the keys you want to update; keys merge into state
    return {"text": state["text"].strip(), "steps": state["steps"] + ["strip"]}

def step_two(state: State):
    return {"text": state["text"].upper(), "steps": state["steps"] + ["upper"]}

builder = StateGraph(State)
builder.add_node("step_one", step_one)
builder.add_node("step_two", step_two)
builder.add_edge(START, "step_one")
builder.add_edge("step_one", "step_two")
builder.add_edge("step_two", END)

app = builder.compile()
print(app.invoke({"text": "  hello langgraph  ", "steps": []}))
# Expected output: {'text': 'HELLO LANGGRAPH', 'steps': ['strip', 'upper']}
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Agent တစ်ခုရဲ့ အဆင့်တိုင်း — plan, tool call, review, final answer — ကို node အဖြစ် သီးခြားချထားလျှင် အဆင့်တစ်ခုချင်းစီကို စမ်းသပ်ရလွယ်ပြီး၊ တစ်နေရာမှ ပြင်ရုံနှင့် တစ်လုံးလုံး မဆိုင်းမတွ မထိခိုက်တော့ပါ။

## ၃။ invoke() နှင့် stream()

### ဘာကို ဆိုလိုတာလဲ

`invoke()` က graph တစ်ခုလုံးကို run ပြီး နောက်ဆုံး state ကို တစ်ပါတည်း ပြန်ပေးသည်။ `stream()` ကတော့ node တစ်ခုပြီးတိုင်း အသစ်ဖြစ်နေသော state ကို အဆင့်ဆင့် ထုတ်ပေးသည်။

### ဘာကြောင့် လဲ

Agent run တစ်ခုက ခဏကြာတတ်သည်။ `stream()` ဖြင့် အဆင့်တိုင်းရဲ့ တိုးတက်မှုကို user မြင်ရမြင်သာ အချိန်နှင့်တပြိုင်နက် ပြသနိုင်ပြီး၊ debug လုပ်တဲ့အခါ ဘယ် node မှာ ရပ်နေလဲဆိုတာ ချက်ချင်းသိနိုင်သည်။

### ဘယ်လို အလုပ်လုပ်လဲ

`invoke({"key": value})` ကို ခေါ်လျှင် state ကို အစပြုပြီး START မှ END ထိ သွားပြီး နောက်ဆုံး state ပြန်သည်။ `stream()` က Python generator ကဲ့သို့ loop လုပ်ရပြီး chunk တိုင်းတွင် node နာမည်နှင့် ထွက်ပေါ်လာသော update ပါဝင်သည်။

### ဥပမာ

```python
# Continuing from the two-step graph above
for chunk in app.stream({"text": "  hello  ", "steps": []}):
    print(chunk)
# Expected output:
# {'step_one': {'text': 'hello', 'steps': ['strip']}}
# {'step_two': {'text': 'HELLO', 'steps': ['strip', 'upper']}}
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Streaming က chatbot user experience အတွက် မရှိမဖြစ်ဖြစ်သလို၊ log စုဆောင်ခြင်းနှင့် monitoring အတွက်လည်း graph တစ်ခုချင်းစီရဲ့ အသေးစိတ် သတင်းအချက်အလက်ကို ရရှိစေသည်။

## ၄။ Chain နှင့် State machine ကွာခြားချက်

### ဘာကို ဆိုလိုတာလဲ

Chain က အဆင့်များကို တစ်နေရာမှ တစ်နေရာသို့ ညာဘက်ရှိရာသို့သာ တန်းစီ သွားသော ဖွဲ့စည်းပုံဖြစ်သည် (A → B → C)။ State machine ကတော့ state တစ်ခုမှ နောက်တစ်ခုသို့ စည်းမျဉ်းအရ ကွဲလွဲသွားနိုင်သော ဖွဲ့စည်းပုံဖြစ်သည် (cycle နှင့် branch ပါဝင်နိုင်သည်)။

### ဘာကြောင့် လဲ

Agent တစ်ခုက tool ခေါ်ပြီး ရလဒ်ကို ကြည့်ပြီးတော့ နောက်ထပ် tool တစ်ခု ထပ်ခေါ်နိုင်သည် — ဒါက chain ရေုံဖြင့် မဖော်ပြနိုင်ဘူး၊ loop လိုလို့။ LangGraph မှာ ဒီကိစ္စကို conditional edge တစ်ခုနှင့် cycle ဖော်ပြပြီး ရေးနိုင်သည်။

### ဘယ်လို အလုပ်လုပ်လဲ

Conditional edge က function တစ်ခုဖြစ်ပြီး state ကို ကြည့်ကာ သွားရမည့် node နာမည်ကို ပြန်ပေးသည်။ နာမည်တူ node ကို ပြန်ခေါ်စေခြင်းဖြင့် loop ဖြစ်လာသည်။

### ဥပမာ

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    count: int

def increment(state: State):
    return {"count": state["count"] + 1}

# Decide whether to loop back or finish, based on the current state
def should_continue(state: State):
    if state["count"] < 3:
        return "increment"
    return END

builder = StateGraph(State)
builder.add_node("increment", increment)
builder.add_edge(START, "increment")
builder.add_conditional_edges("increment", should_continue)

app = builder.compile()
print(app.invoke({"count": 0}))
# Expected output: {'count': 3}
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ

Tool-calling agent အများစုက — model ခေါ် → tool ခေါ် → ရလဒ်ပြန်ကြည့် → ထပ် model ခေါ် — ဆိုတဲ့ cycle ဖြစ်နေတတ်သည်။ ဒီ pattern ကို chain နှင့် မရေးနိုင်သောကြောင့် LangGraph ကို အသုံးပြုခြင်းဖြစ်သည်။

## အနှစ်ချုပ်

LangGraph က agent workflow ကို node (function), edge (ဆက်သွယ်မှု), state (မျှဝေ data) သုံးခုဖြင့် ဖော်ပြသည်။ `StateGraph` ဖြင့် တည်ဆောက်၊ `compile()` ဖြင့် runnable ဖြစ်စေ၊ `invoke()`/`stream()` ဖြင့် run သည်။ Chain ထက် ကွာခြားသည်မှာ — cycle, branch, state-based decision တို့ကို ပထမတန်းစား feature အဖြစ် ထောက်ပံ့ထားခြင်းဖြစ်သည်။
