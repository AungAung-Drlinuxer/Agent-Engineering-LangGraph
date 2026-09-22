# Human-in-the-Loop Approval & Interrupts

LangGraph ရဲ့ `interrupt()` နဲ့ checkpointing ကိုသုံးပြီး AI agent တွေရဲ့ ဆုံးဖြတ်ချက်တွေကို လူသူက စစ်ဆေးပြီး အတည်ပြုပေးရတဲ့ (Human-in-the-Loop) pattern တွေကို တည်ဆောက်ဖို့ သင်ကြားမယ့် module ဖြစ်ပါတယ်။

## ဒီ module မှာ ဘာသင်မလဲ

- `interrupt()` နဲ့ agent run တစ်ခုကို ရပ်တန့်ပြီး လူရဲ့ approval စောင့်တဲ့ gate ထည့်နည်း
- Interrupt ဖြစ်နေချိန်မှာ state ကို ကြည့်ရှု၊ တည်းဖြတ်၊ စစ်ဆေးနည်း
- `Command(resume=...)` နဲ့ workflow ကို ဆက်လက် လည်ပတ်စေနည်း
- Timeout နဲ့ cancellation (သုံးသပ်ပယ်ဖျက်မှု) တွေကို ကိုင်တွယ်နည်း
- Approval တွေရဲ့ audit trail (မှတ်တမ်း) ကို state ထဲ သိမ်းဆည်းနည်း
- Ticket / escalation workflow တွေနဲ့ ချိတ်ဆက်ပုံ pattern

## သင်ခန်းစာများ

1. Approval gate တည်ဆောက်ဖို့ `interrupt()` ရေးနည်း
2. Interrupt ခဏရပ်ချိန်မှာ state ကို inspect နဲ့ edit လုပ်နည်း
3. `Command` နဲ့ resume / reject လုပ်နည်း
4. Timeout နဲ့ cancellation handling
5. Approval မှတ်တမ်း (audit trail) ထည့်သွင်းနည်း
6. Ticket / escalation workflow နဲ့ မေပြောင်းနည်း

## လိုအပ်ချက်များ (Prerequisites)

- Python basics (function, dict, dataclass)
- LangGraph basics — `StateGraph`, nodes, edges (Week 1-2 content)
- LangGraph checkpointing အခြေခံ (`MemorySaver` / `SqliteSaver`)
- pip install လုပ်နိုင်တဲ့ local Python environment

## ဘယ်အချိန်မှာ အသုံးဝင်လဲ

- Agent ကို ငွေပေးချေမှု၊ email ပို့ခြင်း၊ file ဖျက်ခြင်း စတဲ့ အန္တရာယ်ရှိတဲ့ action လုပ်ခွင့်ပေးမယ့်အခါ
- LLM output ကို production မှာ လူက စစ်ပြီးမှ လုပ်ဆောင်ချင်တဲ့အခါ
- Compliance / policy အရ approval မှတ်တမ်းလိုအပ်တဲ့ စနစ်တွေမှာ

## ကိုးကား

- LangGraph Human-in-the-loop concepts: https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/
- LangGraph `interrupt` API reference: https://langchain-ai.github.io/langgraph/reference/graphs/
- LangGraph persistence (checkpoints): https://langchain-ai.github.io/langgraph/concepts/persistence/
