## လေ့ကျင့်ခန်း ၁ — Outcome vs Trajectory ခွဲခြားခြင်း

အောက်ပါ agent ဥပမာကို ကြည့်ပါ — user က "မောလ်ဒိုက်ဖိုင်တွေကို folder စနစ်တကျ ထားပါ" လို့ တောင်းဆိုပြီး agent က `list_files`, `classify_file`, `move_file` tools သုံးခုကို အသုံးပြုသည်။

(က) Outcome evaluation အတွက် တိုင်းတာရမည့်အချက် ၃ ခု ရေးပါ။
(ခ) Trajectory evaluation အတွက် တိုင်းတာရမည့်အချက် ၃ ခု ရေးပါ (ဥပမာ — tool call အစဉ်အလင်း၊ မလိုအပ်ဘဲထပ်မံခေါ်မှု၊ လမ်းကြောင်းအရေအတွက်)။
(ဂ) ရလဒ်မှန်ပေမယ့် trajectory ဆိုးရွားနိုင်သည့် အခြေအနေတစ်ခုကို ဖော်ပြပါ။

```python
# Minimal agent sketch used for this exercise
tools = ["list_files", "classify_file", "move_file"]
task = "Organize the documents in /downloads by file type."
```

**Hints:** Outcome က "ရလဒ်က အလုပ်ဖြစ်ခဲ့လား" ကို မေးသည်။ Trajectory က "ဘယ်လမ်းနဲ့ ရောက်ခဲ့လား" ကို မေးသည်။
**Expected behavior:** ခွဲခြားမှုတစ်ခုစီအတွက် တိုင်းတာချက်များ ထင်ရှားစွာ ကွဲပြားပြီး trajectory ဆိုးရွားသည့် ဥပမာတစ်ခု ပါဝင်သည်။

## လေ့ကျင့်ခန်း ၂ — စမ်းသပ် dataset ဒီဇိုင်းဆွဲခြင်း

Customer-support agent တစ်ခုအတွက် evaluation dataset ကို JSONL ဖိုင်ဖြင့် တည်ဆောက်ပါ။ Record အနည်းဆုံး ၈ ခု ပါစေပြီး အောက်ပါ field များ ပါဝင်ရမည် — `input`, `expected_outcome`, `allowed_tools`, `notes`။ ရှုပ်ထွေးမှုအဆင့် ၃ မျိုး (အလွယ် / အလယ် / အခက်ခဲ) စီမံထားပါ၊ ဥပမာ — အလွယ်: FAQ မေးခွန်း၊ အလယ်: order ပြင်ဆင်ရန် လိုအပ်သည်၊ အခက်ခဲ: မရေရာသော တောင်းဆိုမှု သို့မဟုတ် လုပ်ဆောင်ရန် မဖြစ်နိုင်သော တောင်းဆိုမှု။

```json
{"input": "Where is my order #12345?", "expected_outcome": "Order status returned", "allowed_tools": ["get_order"], "notes": "simple"}
{"input": "Cancel order #12345 even though it shipped.", "expected_outcome": "Politely explain cancellation is not possible", "allowed_tools": ["get_order"], "notes": "hard"}
```

**Hints:** ခက်ခဲသည့် case များတွင် မှားယွင်းသွားလွယ်သော အခြေအနေများ (edge cases) ထည့်ပါ — မရှိသော order၊ ပြင်ပ topic၊ ဆန့်ကျင်ဘက်ဖြစ်နေသော တောင်းဆိုမှုများ။
**Expected behavior:** JSONL ဖိုင်ကို Python `json` library ဖြင့် ဖတ်၍ record ၈ ခုစလုံးကို အောင်မြင်စွာ parse လုပ်နိုင်သည်။

## လေ့ကျင့်ခန်း ၃ — သွားရာလမ်းကြောင်းတိုင်း tracing တပ်ဆင်ခြင်း (Langfuse)

အောက်ပါ LangGraph mini-graph ကို Langfuse ဖြင့် instrument လုပ်ပါ — `pip install langfuse` လုပ်ပြီး environment variables (`LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`) သတ်မှတ်ပါ။ ထို့နောက် Langfuse project တွင် node တိုင်းနှင့် tool call တိုင်း observation အဖြစ် ပေါ်လာစေရန် `CallbackHandler` ကို config ထဲ ထည့်သွင်းပါ။

```python
from langgraph.graph import StateGraph

def classify_node(state):
    # Simple classification step
    state["category"] = "billing" if "refund" in state["query"] else "general"
    return state

builder = StateGraph(dict)
builder.add_node("classify", classify_node)
builder.set_entry_point("classify")
graph = builder.compile()
# TODO: attach Langfuse callback handler and run graph.invoke({"query": "I need a refund"})
```

**Hints:** `langfuse.langchain.CallbackHandler()` ကို `graph.invoke(..., config={"callbacks": [handler]})` ထဲ ထည့်ပါ၊ သို့မဟုတ် `LANGSMITH_TRACING` မဟုတ်ဘဲ Langfuse ၏ OpenTelemetry-based integration အတွက် docs ကို ကြည့်ပါ။
**Expected behavior:** Langfuse UI တွင် run တစ်ခုလျှင် trace တစ်ခု ပေါ်ပြီး `classify` node ၏ input/output metadata များ မြင်နိုင်သည်။

## လေ့ကျင့်ခန်း ၄ — Trajectory တစ်ခု၏ cost နှင့် latency တွက်ချက်ခြင်း

Langfuse trace (သို့မဟုတ် local trace log) တစ်ခုမှ token usage နှင့် အချိန်ကြာမှုကို ထုတ်ယူ၍ trajectory တစ်ခုစီအတွက် စုစုပေါင်း cost နှင့် latency ကို တွက်ချက်သည့် Python function ရေးပါ။ Input: observations စာရင်း (generation များ၊ tool call များ)။ Output: total tokens၊ estimated cost၊ wall-clock latency။

```python
def summarize_trajectory(observations):
    # observations: list of dicts with keys "type", "input_tokens",
    # "output_tokens", "start_time", "end_time", "model"
    # TODO: sum tokens, compute durations, apply per-model pricing map
    pass
```

**Hints:** latency ကို ISO timestamps မှ `datetime` ဖြင့် ဖြည့်ပါ။ Pricing ကို မိမိကိုယ်တိုင် သတ်မှတ်ထားသော table ဖြင့် အသုံးပြုပြီး စဉ်းစားစရာတန်ခိုးများ မကြေည့်ပါနှင့်။
**Expected behavior:** Function က observation list တစ်ခုကို လက်ခံ၍ token စုစုပေါင်း၊ ကုန်ကျစရိတ် ခန့်မှန်းချက်၊ စုစုပေါင်းအချိန်ကြာမှု သုံးခုကို dict ဖြင့် ပြန်ပေးသည်။

## လေ့ကျင့်ခန်း ၅ — Failure taxonomy ချမှတ်ခြင်း

Agent run များမှ ဖော်ပြထားသော failure log များကို ကြည့်ပြီး failure အမျိုးအစား ခွဲခြားသော taxonomy တစ်ခု သတ်မှတ်ပါ။ အနည်းဆုံး အမျိုးအစား ၅ မျိုး ထည့်ပါ — ဥပမာ: (၁) tool selection မှားယွင်းမှု၊ (၂) tool input ပုံစံမှားယွင်းမှု၊ (၃) reasoning လမ်းကြောင်း မှားယွင်းမှု၊ (၄) context ဆုံးရှုံးမှု / context နယ်နိမိတ်ကျော်လွန်မှု၊ (၅) final answer ပုံစံပျက်မှု။ ထို့နောက် အောက်ပါ log များကို အလိုအလျောက် အမျိုးအစားခွဲသော classifier function ရေးပါ။

```python
logs = [
    {"error": "Tool 'search_orders' raised: invalid date format", "step": 3},
    {"error": "Agent called 'send_email' instead of 'create_ticket'", "step": 1},
    {"error": "Answer given in English, expected Burmese", "step": 6},
]
```

**Hints:** Error string ထဲရှိ သော့ချက်စကားလုံးများ (ဥပမာ "invalid", "instead of") ဖြင့် rule-based ခွဲခြားခြင်းဖြင့် စတင်ပါ။
**Expected behavior:** Function က log တစ်ခုစီအတွက် taxonomy အမျိုးအစားတစ်ခုကို ပြန်ပေးပြီး အမျိုးအစားစဉ်မရှိ ဖြစ်ပါက `unknown` ဟု ပြန်သည်။

## လေ့ကျင့်ခန်း ၆ — CI regression gate တည်ဆောက်ခြင်း

Exercise ၂ ၏ dataset ဖြင့် agent ကို အလုံးစုံ စမ်းသပ်ပြီး GitHub Actions job တစ်ခုအတွင်း regression gate တပ်ဆင်ပါ။ Pass criteria — outcome pass rate ≥ ၉၀%၊ trajectory အတွက် မလိုအပ်သော tool call များ ပျမ်းများ ၂ ခုအောက်နိမ့်ရမည်။ Criteria တစ်ခုခု ကျရောက်ပါက exit code မှားဖြင့် job ကို fail ဖြစ်စေပါ။

```python
import sys

def regression_gate(results):
    # results: list of dicts with keys "outcome_pass", "tool_calls", "expected_tool_calls"
    # TODO: compute pass rate and avg extra tool calls, return True/False
    pass

if __name__ == "__main__":
    results = load_results("eval_results.jsonl")
    sys.exit(0 if regression_gate(results) else 1)
```

**Hints:** ထောင်ပြန်နှုန်းထက် မည်သည့် threshold ကမှ ယုံကြည်စိတ်ချရမည်ကို သတိရပါ — dataset သေးသောအခါ တစ်ခုမှားရုံနှင့် ၉၀% ကျဆင်းသည်။ YAML workflow file တွင် `python eval_gate.py` ကို run စေပါ။
**Expected behavior:** Dataset အားလုံး အောင်မြင်ပါက script က exit code 0 ပြန်ပြီး၊ gate threshold တစ်ခု ကျော်လွန်ပါက exit code 1 ဖြင့် CI pipeline ရပ်တန့်သည်။
