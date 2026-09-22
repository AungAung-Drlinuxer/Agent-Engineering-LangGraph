# Agent Evaluation & Observability (LangGraph + Langfuse)

## Outcome vs Trajectory Evals

### ဘာကို ဆိုလိုတာလဲ
Agent evaluation ဆိုတာ ကျွန်ုပ်တို့ ဆောက်ထားတဲ့ agent ရဲ့ အလုပ်လုပ်ပုံကို တိုင်းတာစစ်ဆေးတာ ဖြစ်ပါတယ်။ ဒီမှာ နည်းလမ်းနှစ်မျိုးရှိပါတယ် — **outcome eval** က နောက်ဆုံးရလဒ် (final answer) ကိုပဲ ကြည့်တာပါ။ **trajectory eval** က agent က ရလဒ်အထိ သွားခဲ့တဲ့ လမ်းကြောင်းအဆင့်ဆင့် (ဘယ် tool ခေါ်ခဲ့လဲ၊ ဘယ် node ကနေ ဖြတ်ခဲ့လဲ) ကို ကြည့်တာပါ။

### ဘာကြောင့် လဲ
ရလဒ်တစ်ခုတည်းကိုပဲ ကြည့်ရင် မေးခွန်းရိုးရိုးမှာ အလုပ်ဖြစ်ပေမယ့် agent မှာ လမ်းလွဲသွားပြီး ကံကောင်းလို့ အဖြေမှန်ထွက်တာတွေ ရှိနိုင်ပါတယ်။ ဥပမာ — agent က မှားတဲ့ tool ကို အရင်ခေါ်ပြီးမှ အဖြေမှန်ရအောင် ပြန်ကြိုးစားတာ ဒါမှမဟုတ် လုံးဝ evidence မရှိဘဲ စာတွေး၍ အဖြေထုတ်တာမျိုး ဖြစ်နိုင်ပါတယ်။ Trajectory eval က ဒီလို ပြဿနာတွေကို ဖမ်းဖို့ လိုအပ်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
Outcome eval က dataset ထဲက မေးခွန်းတိုင်းအတွက် မျှော်မှန်းအဖြေ (ground truth) နဲ့ agent ရဲ့ အဖြေကို နှိုင်းယှဉ်ပါတယ်။ LLM-as-judge နဲ့ ဖြစ်စေ၊ exact match နဲ့ ဖြစ်စေ အမှတ်ပေးနိုင်ပါတယ်။ Trajectory eval ကတော့ LangGraph run တစ်ခုရဲ့ trace (node/tool call စာရင်း) ကို စည်းမျဉ်းတွေနဲ့ စစ်ပါတယ် — ဥပမာ "tool အနည်းဆုံး ၃ ခု အတွင်း ပြီးရမယ်"၊ "မလိုအပ်တဲ့ tool မခေါ်ရ" စသဖြင့်။

### ဥပမာ

```python
# Simple outcome and trajectory evaluation for a LangGraph agent run

def evaluate_outcome(expected: str, actual: str) -> bool:
    # Compare the final answer with the ground truth
    return expected.strip().lower() == actual.strip().lower()

def evaluate_trajectory(steps: list) -> dict:
    # steps: ordered list of node/tool names from the trace
    tool_calls = [s for s in steps if s.startswith("tool:")]
    checks = {
        "used_search_tool": any("search" in s for s in tool_calls),
        "under_max_steps": len(steps) <= 6,
    }
    return {"passed": all(checks.values()), "checks": checks}

expected_answer = "paris"
actual_answer = "Paris"
trace_steps = ["agent", "tool:web_search", "agent", "__end__"]

print("outcome:", evaluate_outcome(expected_answer, actual_answer))
print("trajectory:", evaluate_trajectory(trace_steps))
# Expected output:
# outcome: True
# trajectory: {'passed': True, 'checks': {'used_search_tool': True, 'under_max_steps': True}}
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Prompt ဒါမှမဟုတ် model ပြောင်းတဲ့အခါ outcome eval တစ်ခုတည်းနဲ့ တခါတရံ မကောင်းတဲ့ အပြောင်းအလဲတွေကို မမြင်ရပါ။ Trajectory ကိုပါ တိုင်းတာမှ agent က တကယ့် မှန်ကန်တဲ့ လုပ်ငန်းစဉ်နဲ့ အဖြေရတာလား၊ ကံကောင်းလို့ ရတာလားဆိုတာ ခွဲခြားနိုင်ပါတယ်။ ထုတ်လုပ်ရေး system မှာ ဒီနှစ်မျိုးလုံး ပေါင်းသုံးဖို့ အလွန်အရေးကြီးပါတယ်။

## Dataset Design for Agents

### ဘာကို ဆိုလိုတာလဲ
Agent evaluation dataset ဆိုတာ agent ကို စမ်းသပ်ဖို့ မေးခွန်း၊ မျှော်မှန်းရလဒ်၊ တခါတရံ မျှော်မှန်း trajectory တွေ ပါတဲ့ စုစည်းမှု ဖြစ်ပါတယ်။ ရိုးရှင်းတဲ့ QA dataset ထက် ပိုငွာ — ပတ်ဝန်းကျင် state၊ ရရှိနိုင်တဲ့ tool တွေ၊ လူသူမသိမန်းရှိမယ့် edge case တွေပါ ထည့်သွင်းစဉ်းစားရပါတယ်။

### ဘာကြောင့် လဲ
Dataset ညံ့ရင် eval score က ဘာမှ အဓိပ္ပာယ်မရှိပါ။ Agent တွေမှာ တစ်ကြိမ်တည်းနဲ့ အဖြေရတာတွေ၊ tool အများကြီးလိုအပ်တာတွေ၊ မရနိုင်တဲ့ tool ကို တောင်းတာတွေ ရှိပါတယ်။ အဲဒီ scenario အမျိုးမျိုးကို dataset ထဲ မထည့်ပေးရင် agent ရဲ့ တကယ့် ခံနိုင်ရည်ကို မသိနိုင်ပါ။

### ဘယ်လို အလုပ်လုပ်လဲ
အလေ့အကျ နည်းလမ်းက — (၁) real user queries တွေကနေ နမူနာယူ၊ (၂) လက်ရှိ trace တွေထဲက failure case တွေကို dataset ထဲ ထည့်၊ (၃) synthetic edge case တွေ ဖန်တီး၊ (၄) ground truth ကို လူနဲ့ စစ်ပါ။ ဥပမာအနေနဲ့ JSON ဖိုင်နဲ့ သိမ်းတာ အလွယ်ဆုံးပါ။

### ဥပမာ

```python
import json

# Each dataset item includes input, expected answer, and metadata
dataset = [
    {
        "id": "q001",
        "input": "What is the capital of France?",
        "expected": "Paris",
        "max_tool_calls": 1,
        "tags": ["simple", "factual"],
    },
    {
        "id": "q002",
        "input": "Summarize the latest changes in our pricing doc.",
        "expected_contains": ["pricing"],
        "max_tool_calls": 4,
        "tags": ["tool-heavy", "retrieval"],
    },
    {
        "id": "q003",
        "input": "Translate this to Latin.",
        "expected": None,  # agent should say it cannot do this
        "max_tool_calls": 0,
        "tags": ["edge-case", "refusal"],
    },
]

# Save to disk so CI and experiments share the same cases
with open("agent_eval_dataset.json", "w", encoding="utf-8") as f:
    json.dump(dataset, f, indent=2)

print(f"saved {len(dataset)} eval cases")
# Expected output:
# saved 3 eval cases
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Dataset က ခင်းပြချက် (specification) လို ဖြစ်လာပါတယ် — "agent က ဒီ case တွေအပေါ် ဒီလို အလုပ်လုပ်သင့်တယ်" ဆိုတာကို စာရွက်စာတမ်းသဖွယ် ဖြစ်လာစေပါတယ်။ Prompt ပြင်တိုင်း၊ model ပြောင်းတိုင်း ဒီ dataset နဲ့ ပြန်စစ်နိုင်တော့ regression ဖမ်းနိုင်ပါတယ်။

## Tracing Every Node/Tool Call with Langfuse

### ဘာကို ဆိုလိုတာလဲ
Tracing ဆိုတာ agent run တစ်ခုရဲ့ အဆင့်တိုင်း — LLM call, tool call, node transition, token usage, latency — ကို မှတ်တမ်းတင်ထားတာပါ။ Langfuse က open-source observability platform တစ်ခုဖြစ်ပြီး LangChain/LangGraph နဲ့ integration တွေ တောင်းဆိုပါတယ်။

### ဘာကြောင့် လဲ
Agent တွေက black box သဖွယ် ဖြစ်တတ်ပါတယ် — ဘာကြောင့် အဖြေမှားလဲ၊ ဘယ် node မှာ နှောင့်နှေးလဲ၊ ဘယ် tool က token အများကြီး စားလဲ ဆိုတာကို မမြင်ရင် ပြင်ဖို့ မလွယ်ပါ။ Trace ရှိမှ failure ကို diagnose လုပ်နိုင်ပြီး eval dataset အတွက် နမူနာတွေပါ ရပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
`langfuse` နဲ့ `langchain` integration သုံးရင် `CallbackHandler` တစ်ခုကို LangGraph app ရဲ့ `invoke` မှာ pass လုပ်ရုံပါ။ LangGraph က node/tool တိုင်းကို trace ထဲမှာ nested span အဖြစ် တလဲလဲ မှတ်ပါလိမ့်မယ်။ Langfuse dashboard မှာ trace တိုင်းကို ကြည့်နိုင်ပါတယ်။ Environment variables ကနေ `LANGFUSE_PUBLIC_KEY` နဲ့ `LANGFUSE_SECRET_KEY` ထည့်ပေးရပါမယ်။

### ဥပမာ

```python
# pip install langfuse langchain langgraph
import os
from langfuse.langchain import CallbackHandler
from my_agent import graph  # your compiled LangGraph agent

# Keys can be set via environment variables instead of hardcoding
handler = CallbackHandler()

result = graph.invoke(
    {"messages": [("user", "What is LangGraph?")]},
    config={"callbacks": [handler]},
)

print(result["messages"][-1].content[:60])
# Expected output:
# LangGraph is a framework for building stateful, multi-actor...
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Production agent မှာ အဖြေမှားတဲ့အခါ "ဘာဖြစ်လဲ" ဆိုတာကို trace ကြည့်ပြီး မိနစ်ပိုင်းအတွင်း ဖြေရှင်းနိုင်ဖို့ အရေးကြီးပါတယ်။ အပြင်မှ user complaint နဲ့ အတွင်းမှာ trace ကို ချိတ်ဆက်နိုင်ဖို့ trace ID ကို response မှာ ထည့်ပေးတာလည်း နည်းကောင်းပါ။

## Cost/Latency per Trajectory + Regression Gates in CI

### ဘာကို ဆိုလိုတာလဲ
Trajectory တိုင်းအတွက် ကုန်ကျစရိတ် (token/USD) နဲ့ အချိန် (latency) ကို တိုင်းတာတာပါ။ Regression gate က CI pipeline (ဥပမာ GitHub Actions) ထဲမှာ "score ဒါမှမဟုတ် latency က threshold ထက် ပိုဆိုးသွားရင် build fail အဖြစ် ကြေညာ" ဆိုတဲ့ စည်းမျဉ်းပါ။

### ဘာကြောင့် လဲ
Agent တစ်ခုက မေးခွန်းတစ်လုံးအတွက် LLM call ၁၀ ခါထိ ခေါ်နိုင်ပါတယ်။ Prompt အသေးစားတစ်ခု ပြောင်းလိုက်တာနဲ့ cost နဲ့ latency က သိသိသာသာ တက်နိုင်ပါတယ်။ ဒါကို မတိုင်းဘဲ နေရင် user experience နဲ့ bill နှစ်ခုလုံး ထိခိုက်ပြီးမှ သိတတ်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
Langfuse trace တိုင်းမှာ token usage နဲ့ duration ပါပြီးသားမို့ observation တွေကနေ စုပေါင်းနိုင်ပါတယ်။ ဒါမှမဟုတ် ကိုယ်တိုင် timer တပ်ပြီး တိုင်းလည်း ရပါတယ်။ ပြီးရင် eval script တစ်ခုရေးပြီး CI မှာ အမြဲတမ်း run ခိုင်းပြီး baseline နဲ် နှိုင်းပါတယ်။

### ဥပမာ

```python
import subprocess, sys, time

# Simple CI regression gate script: run evals, then compare against baseline
MAX_AVG_LATENCY_SECONDS = 8.0
MAX_FAILURE_RATE = 0.2

def run_gate(avg_latency: float, failure_rate: float) -> None:
    if avg_latency > MAX_AVG_LATENCY_SECONDS:
        sys.exit(f"FAIL: avg latency {avg_latency:.1f}s exceeds gate")
    if failure_rate > MAX_FAILURE_RATE:
        sys.exit(f"FAIL: failure rate {failure_rate:.1%} exceeds gate")
    print("PASS: all regression gates satisfied")

# In real CI these values come from running the eval dataset
avg_latency = 5.4
failure_rate = 0.1
run_gate(avg_latency, failure_rate)
# Expected output:
# PASS: all regression gates satisfied
```

CI ထဲမှာ ဒီ script ကို `python eval_gate.py` ဆိုပြီး run ခိုင်ပြီး exit code မှန်ရမှ merge ခွင့်ပေးဖို့ branch protection rule တပ်ဆင်ပါတယ်။

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
အဖွဲ့အစည်းတစ်ခုလုံး agent ပြင်နေတဲ့အခါ "ငါ့ပြင်ချက်က score ကို တက်စေလား၊ cost/latency ကို တက်စေလား" ဆိုတာကို အလိုအလျောက် စစ်ပေးနိုင်ဖို့ regression gate က စောင့်နေတဲ့ စောင့်ရှောက်သမားသဖွယ် ဖြစ်ပါတယ်။

## Failure Taxonomy

### ဘာကို ဆိုလိုတာလဲ
Failure taxonomy ဆိုတာ agent ရဲ့ အမှားတွေကို အမျိုးအစားခွဲတဲ့ စနစ်တစ်ခုပါ။ ဥပမာ — tool input မှားတာ (wrong tool args)၊ မလိုအပ်တဲ့ tool ခေါ်တာ (unnecessary call)၊ context ကို မှတ်မမိတာ (context loss)၊ မှားတဲ့ state ကို ဖတ်တာ (state misread)၊ refusal failure (ငြင်းသင့်တာကို မငြင်းတာ) စသဖြင့်။

### ဘာကြောင့် လဲ
အမှားအားလုံးကို "agent မှားတယ်" လို့ပဲ စာရင်းတင်ထားရင် ဘယ်နေရာကို အရင်ပြင်ရမလဲ ဆုံးဖြတ်လို့ မရပါ။ အမျိုးအစားခွဲမှ ပြင်ဆင်မှုရဲ့ ဦးစားပေးကို data-driven နဲ့ ဆုံးဖြတ်နိုင်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
Trace
