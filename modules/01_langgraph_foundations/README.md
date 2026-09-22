# LangGraph Foundations — Graphs, Nodes, State

LangGraph ကို အသုံးပြုပြီး agent logic ကို explicit graph အဖြစ် တည်ဆောက်နည်းကို သင်ကြားသော module ဖြစ်သည်။

## ဒီ module မှာ ဘာသင်မလဲ

- `while` loop အစား graph ကို ဘာကြောင့် အသုံးပြုရသလဲဆိုတာကို နားလည်ခြင်း
- `StateGraph`, node, edge တို့ရဲ့ အယူအဆများ
- `compile()`, `invoke()`, `stream()` တို့ရဲ့ အသုံးပြုပုံ
- chain နဲ့ state machine ရဲ့ ကွာခြားချက်

## သင်ခန်းစာများ

1. Agent logic အတွက် while-loop ထက် graph က ဘာကြောင့် ပိုကောင်းလဲ
2. `StateGraph` နှင့် State schema သတ်မှတ်နည်း
3. Node ရေးနည်း၊ edge ချိတ်နည်း၊ `compile()` လုပ်နည်း
4. `invoke()` နှင့် `stream()` နှစ်မျိုးလုံး အသုံးပြုနည်း
5. Chain နှင့် State machine ကွာခြားချက်

## လိုအပ်ချက်များ (Prerequisites)

- Python function ရေးတတ်ခြင်း၊ dictionary နှင့s typing (Pydantic) အခြေခံ
- Python 3.10 နှင့်အထက် တစ်ခု တပ်ဆင်ထားခြင်း
- အောက်ပါ package များ တပ်ဆင်ထားရန် — `pip install langgraph`

## ဘယ်အချိန်မှာ အသုံးဝင်လဲ

- Agent workflow ကို loop၊ branch သို့မဟုတ် human-in-the-loop step များနှင့် ထိန်းချုပ်ချင်သောအခါ
- Agent run တစ်ခုချင်းစီရဲ့ state history ကို စစ်ခိုင်းချင်သောအခါ
- Chain တစ်ခုတည်းထက် ပိုရှုပ်ထွေးပြီး အဆင့်များစွာ ပြန်လည်သွားလာရသော logic ရေးသောအခါ

## ကိုးကား

- LangGraph official docs — https://langchain-ai.github.io/langgraph/
- Graph API concepts — https://langchain-ai.github.io/langgraph/concepts/low_level/
- Quickstart — https://langchain-ai.github.io/langgraph/tutorials/introduction/
