# Agent Evaluation & Observability

LangGraph agent များ၏ ရလဒ် (outcome) နှင့် လမ်းကြောင်း (trajectory) ကို တိုင်းတာတဲ့ evaluation နည်းစနစ်၊ Langfuse ဖြင့် tracing၊ regression gates နှင့် failure taxonomy အကြောင်း သင်ကြားပါမယ်။

## ဒီ module မှာ ဘာသင်မလဲ

- Outcome evaluation (နောက်ဆုံးရလဒ်အားဖြင့် အမှတ်ပေးခြင်း) နှင့် trajectory evaluation (agent လိုက်သွားတဲ့ အဆင့်ဆင့် လမ်းကြောင်းကို စစ်ဆေးခြင်း) ကြားထဲက ကွာခြားချက်
- Agent evaluation အတွက် ထိထိရောက်ရောက် dataset စတင်တည်ဆောက်နည်း — input နမူနာများ၊ မျှော်မှန်းရလဒ်များနှင့် လိုအပ်တဲ့ tool call များ ထည့်သွင်းနည်း
- Langfuse ကိုသုံးပြီး graph ရဲ့ တစ်ခုချင်းစီကို node နှင့် tool call တိုင်းကို tracing လုပ်နည်း
- CI pipeline ထဲမှာ regression gate တွေ ထည့်သွင်းပြီး ရလဒ် ဆိုးရွားသွားမှုကို အလိုအလျောက် ဖမ်းဆီးနည်း
- Trajectory တစ်ခုချင်းစီအတွက် token cost နှင့် latency တိုင်းတာနည်း
- Agent failure အမျိုးအစားများကို အမျိုးအစားခွဲခြား (taxonomy) ပြီး စနစ်တကျ ခွဲခြမ်းစိတ်ဖြာနည်း

## သင်ခန်းစာများ

1. **Outcome vs Trajectory Evaluations** — နောက်ဆုံးအဖြေမှန်/မှားကိုသာ ကြည့်တာနဲ့ agent ရွေးချယ်တဲ့ အဆင့်တွေကိုပါ စစ်ဆေးတာ ကွာခြားပုံ
2. **Agent Evaluation Dataset Design** — input နမူနာ၊ expected output နဲ့ expected tool sequence ပါဝင်တဲ့ dataset ဖိုင်ဖွဲ့စည်းနည်း
3. **Tracing Nodes & Tool Calls with Langfuse** — Langfuse callback handler နဲ့ LangGraph graph တိုင်းရဲ့ အတွင်းပိုင်း run များကို မြင်သာစေနည်း
4. **Regression Gates in CI** — evaluation ရလဒ်တွေကို GitHub Actions စတဲ့ CI ထဲမှာ threshold သတ်မှတ်ပြီး စစ်ဆေးနည်း
5. **Cost & Latency per Trajectory** — trajectory တစ်ခုချင်းစီရဲ့ token သုံးစွဲမှု နဲ့ response အချိန်ကို tracing data ကနေ ထုတ်ယူနည်း
6. **Failure Taxonomy** — planning error, tool-selection error, hallucinated tool argument, context loss စတဲ့ failure အမျိုးအစားတွေကို ခွဲခြားသတ်မှတ်နည်း

## လိုအပ်ချက်များ (Prerequisites)

- Python အခြေခံ နှင့် LangGraph graph တည်ဆောက်နည်း (ယခု track ရဲ့ ရှေ့ module များ)
- LangGraph project တစ်ခုကို လက်တွေ့ ဖန်တီးပြီးဖြစ်မှု
- `langfuse` Python SDK ထည့်သွင်းရန် — `pip install langfuse`
- Langfuse cloud account (လိုအပ်ပါက project key နှစ်ခု ရယူရန်)
- Git နှင့် CI (GitHub Actions) အခြေခံ အသုံးပြုနည်း

## ဘယ်အချိန်မှာ အသုံးဝင်လဲ

Agent တစ်ခုကို prototype ထက်ပိုပြီး production ထိ ရောက်အောင် တိုးတက်စေချင်တဲ့အခါ၊ prompt ဒါမှမဟုတ် model ပြောင်းတဲ့အခါ ဘယ်အပြောင်းအလဲက ဘယ်အပိုင်းကို ထိခိုက်လဲဆိုတာ သိရဖို့၊ deployment မတင်ခင် regression တွေ ဖမ်းဖို့၊ နဲ့ တစ် trajectory ချင်းစီရဲ့ ကုန်ကျစရိတ် အချိန်ကို မြင်သာစွာ စီမံခန့်ခွဲဖို့ ဒီ module က တိုက်ရိုက် အသုံးဝင်ပါတယ်။

## ကိုးကား

- Langfuse documentation: https://langfuse.com/docs
- Langfuse OpenTelemetry & tracing guide: https://langfuse.com/docs/opentelemetry/get-started
- LangGraph documentation: https://langchain-ai.github.io/langgraph/
- LangSmith evaluation concepts (outcome vs trajectory): https://docs.smith.langchain.com/evaluation
