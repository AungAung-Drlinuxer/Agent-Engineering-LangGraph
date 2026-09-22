# State, Messages & Reducers

LangGraph agent တစ်ခုရဲ့ အခြေခံအကျဆုံးဖြစ်တဲ့ state schema တည်ဆောက်ပုံ၊ message များ ပေါင်းစည်းပုံ (reducer) နှင့် partial state update စနစ်ကို လက်တွေ့သင်ကြားပါမယ်။

## ဒီ module မှာ ဘာသင်မလဲ

- `TypedDict` နှင့် Pydantic model ဖြင့် state schema သတ်မှတ်ပုံ နှစ်မျိုးကွာခြားချက်
- `Annotated` reducer (ဥပမာ `add_messages`) က message list ကို overwrite မလုပ်ဘဲ append လုပ်တဲ့ အကြောင်းရင်း
- Channel semantics — node တစ်ခုက state ရဲ့ ဘယ်အပိုင်းကို ပြောင်းလွှဲပိုင်ချင်တာလဲဆိုတာ
- Partial state update — state အားလုံးကို ပြန်မပေးရဘဲ ပြောင်းချင်တဲ့ field တွေပဲ ပြန်ပေးခြင်း
- Reducer မရှိရင် overwrite၊ reducer ရှိရင် merge လုပ်တဲ့ စည်းမျဉ်းနှင့် အကြောင်းအရင်း
- State ထဲမှာ မလိုအပ်တဲ့ data များ ထည့်သွင်းခြင်းကြောင့် ဖြစ်လာနိုင်တဲ့ memory နှင့် cost ဆိုင်ရာ သတိပြုရမည့်အချက်များ

## သင်ခန်းစာများ

1. `TypedDict` ဖြင့် ရိုးရှင်းတဲ့ agent state သတ်မှတ်ပုံ နှင့် type checking အားသာချက်
2. Pydantic `BaseModel` ဖြင့် state schema ရေးပုံ နှင့် runtime validation ရရှိပုံ
3. `add_messages` reducer ကို `Annotated[list, add_messages]` အနေနဲ့ သုံး၍ message history စုစည်းပုံ
4. Reducer မပါတဲ့ field များကို LangGraph က default overwrite လုပ်တဲ့ channel စနစ်
5. Node တစ်ခုကနေ state ရဲ့ တစ်စိတ်တစ်ဒေသကိုပဲ ပြန်တင်တဲ့ partial update ပုံစံ
6. State ထဲက data တွေက context window size နှင့် LLM token cost အပေါ် ဘယ်လိုသက်ရောက်တာလဲ

## လိုအပ်ချက်များ (Prerequisites)

- Python typing (`TypedDict`, `Annotated`, `Optional`) အခြေခံ သိရှိထားခြင်း
- Python 3.9 နှင့်အထက်၊ `langgraph` package install ထားခြင်း (`pip install langgraph`)
- Pydantic v2 ရဲ့ `BaseModel` အခြေခံ သိထားပါက ပိုအဆင်ပြေပါမယ်
- LangGraph graph (node/edge) တည်ဆောက်ပုံ အခြေခံ နားလည်ထားခြင်း

## ဘယ်အချိန်မှာ အသုံးဝင်လဲ

- Chatbot သို့မဟုတ် agent တစ်ခုမှာ conversation history ကို ဆက်တိုက် စုထားချင်တဲ့အခါ (`add_messages` ရဲ့ အဓိက အသုံးဝင်ပုံ)
- Graph ထဲက node တွေက တူညီတဲ့ state field ကို တစ်ခုပြီးတစ်ခု ရေးမှတ်တဲ့အခါ reducer သတ်မှတ်ပြီး merge / overwrite ကို ထိန်းချုပ်ချင်တဲ့အခါ
- Long-running agent များမှာ state ကြီးလာတာကို ကာကွယ်ဖို့ ဘယ် data ကို state ထဲထားရမလဲ ဆုံးဖြတ်ချင်တဲ့အခါ
- Checkpointing သုံးတဲ့ system မှာ state structure ကို စနစ်တကျ ဒီဇိုင်းချင်တဲ့အခါ

## ကိုးကား

- LangGraph Low-level Concepts (State, Channels, Reducers): https://langchain-ai.github.io/langgraph/concepts/low_level/
- LangGraph API Reference — `add_messages`: https://langchain-ai.github.io/langgraph/graphs/messages/
- Pydantic Documentation: https://docs.pydantic.dev/latest/
- Python `typing` module docs: https://docs.python.org/3/library/typing.html
