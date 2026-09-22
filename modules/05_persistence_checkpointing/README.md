# Persistence, Checkpointing & Time Travel

LangGraph agent များရဲ့ state ကို checkpoint အနေနဲ့ သိမ်းဆည်းပြီး crash ဖြစ်ပြီးနောက် ပြန်လည် ဆက်လက်လုပ်ဆောင်ခြင်း (resuming)၊ အတိတ် checkpoint ကနေ ပြန်ကစားခြင်း (time travel) နဲ့ thread အချင်းချင်း memory မျှဝေခြင်းတို့ကို လေ့လာမယ့် module ပါ။

## ဒီ module မှာ ဘာသင်မလဲ

- Checkpointer backend သုံးမျိုး (`InMemorySaver`, `SqliteSaver`, `PostgresSaver`) ရဲ့ ကွာခြားချက်နဲ့ ရွေးချယ်ပုံ
- `thread_id` ဆိုတာ ဘာလဲ၊ conversation တစ်ခုကို ဘယ်လို ခွဲခြားသလဲ
- Failure ဖြစ်ပြီးနောက် checkpoint ကနေ graph ကို ဘယ်လို ပြန်ဆက်လုပ်မလဲ
- အတိတ် checkpoint တစ်ခုကို replay လုပ်ပြီး လမ်းကြောင်းအသစ် ဖန်တီးပုံ (time travel)
- `InMemoryStore` / long-term store သုံးပြီး thread မတူဘဲ memory မျှဝေပုံ

## သင်ခန်းစာများ

1. Checkpointer ဆိုတာ ဘာလဲ — အခြေခံ concept
2. Backend သုံးမျိုး — memory, sqlite, postgres
3. `thread_id` semantics နဲ့ state ခွဲခြားပုံ
4. Failure နောက် ပြန်စခြင်း (resuming after failure)
5. Time travel — အတိတ် checkpoint ကနေ replay
6. Long-term store — thread ချင်းချင်း memory

## လိုအပ်ချက်များ (Prerequisites)

- LangGraph အခြေခံ graph, node, edge concept များ (Week: Agent Engineering ၏ ရှေ့ module များ)
- Python အခြေခံ — dictionary, function, class
- `pip install langgraph` နဲ့ optional: `pip install langgraph-checkpoint-sqlite`
- SQLite/PostgreSQL အခြေခံ နားလည်မှု (postgres အတွက်)

## ဘယ်အချိန်မှာ အသုံးဝင်လဲ

- Agent ကို ရက်သတ္တပတ်များတစ်ကြာ လုပ်ငန်းစဉ်ကြီးတစ်ခုအတွက် သုံးတဲ့အခါ state တွေ မပျောက်စေချင်လျှင်
- Server restart ဒါမှမဟုတ် crash ဖြစ်တဲ့အခါ user နဲ့ စကားဝိုင်း ဆက်လက်ရှိစေချင်လျှင်
- Agent ရဲ့ ဆုံးဖြတ်ချက်တစ်ခုကို ပြန်ပြင်ပြီး အတိတ် state ကနေ လမ်းကြောင်းအသစ် စမ်းသပ်ချင်လျှင်
- တစ် user ရဲ့ နှစ်များစွာကြာဆွေးနွေးမှုများကို သူ့ profile တစ်ခုတည်းအောက်မှာ စုစည်းချင်လျှင်

## ကိုးကား

- Persistence concept — https://langchain-ai.github.io/langgraph/concepts/persistence/
- Persistence how-to — https://langchain-ai.github.io/langgraph/how-tos/persistence_sqlite/
- LangGraph API reference — https://langchain-ai.github.io/langgraph/reference/checkpoint/
- Memory & long-term store — https://langchain-ai.github.io/langgraph/concepts/memory/
