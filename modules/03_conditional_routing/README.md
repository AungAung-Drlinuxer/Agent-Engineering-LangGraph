# Conditional Routing, Loops & Termination

LangGraph ထဲမှာ conditional edges တွေ၊ retry loops တွေနဲ့ graph ရပ်တန့်မှု (termination) ကို ထိန်းချုပ်နည်းသင်ကြားမယ့် module ဖြစ်ပါတယ်။

## ဒီ module မှာ ဘာသင်မလဲ

- Conditional edges နဲ့ routing functions ရေးနည်း
- Graph ထဲမှာ cycles (loops) ဖန်တီးပြီး retry/refine လုပ်နည်း
- `recursion_limit` နဲ့ infinite loop ကာကွယ်နည်း
- Termination conditions သတ်မှတ်နည်း
- Deterministic fallbacks နဲ့ လုံခြုံစွာ ရပ်တန့်စေနည်း

## သင်ခန်းစာများ

1. Conditional edges နဲ့ routing functions
2. Cycles — retry နဲ့ refine loops
3. `recursion_limit` နဲ့ guardrails
4. Termination conditions နဲ့ deterministic fallbacks

## လိုအပ်ချက်များ (Prerequisites)

- Python အခြေခန်း (functions, dictionaries, dataclasses)
- LangGraph basics — `StateGraph`, nodes, edges, `START` / `END`
- `pip install langgraph` နဲ့ install ပြုလုပ်ထားရန်

## ဘယ်အချိန်မှာ အသုံးဝင်လဲ

- LLM output ကို အရည်အသွေးမမှီရင် အလိုအလျောက် retry ချင်တဲ့အခါ
- စာသားကို အဆင့်ဆင့် refine လုပ်ချင်တဲ့ agent workflow တွေမှာ
- Agent တစ်ခုက loop ထဲမှာ မနားတော့ဘူးဆိုတာ သတိပြုရတဲ့အခါ
- Production မှာ graph run တစ်ခုက အလွန်ရှည်ပြီး resource ပျက်ဆီးမှု ကာကွယ်ချင်တဲ့အခါ

## ကိုးကား

- LangGraph Low-level Concepts — <https://langchain-ai.github.io/langgraph/concepts/low_level/>
- LangGraph API Reference — <https://langchain-ai.github.io/langgraph/reference/graphs/>
