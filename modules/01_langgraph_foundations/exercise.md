# လေ့ကျင့်ခန်းများ — LangGraph Foundations

## လေ့ကျင့်ခန်း ၁ — ရိုးရိုး graph တစ်ခု တည်ဆောက်ခြင်း

`State` ကို `greeting: str` key တစ်ခုတည်းဖြင့် သတ်မှတ်ပါ။ `make_greeting` node တစ်ခုထည့်ပြီး `name: str` ကို input မှ လက်ခံကာ `"Hello, {name}!"` ဆိုသော `greeting` ကို ထုတ်ပေးစေပါ။ `START → make_greeting → END` graph ကို compile လုပ်ပြီး `invoke()` ဖြင့် စမ်းပါ။

**Hints:** State schema ထဲ `name` ပါ ထည့်ရေးရန် — input state ထဲ key နှစ်ခုလုံး ပါဝင်မည်။ `TypedDict` ကို `typing` မှ import လုပ်ပါ။
**Expected behavior:** `invoke({"name": "Aung", "greeting": ""})` ကို ခေါ်လျှင် `greeting` ရလဒ်မှာ `Hello, Aung!` ဖြစ်သည်။

## လေ့ကျင့်ခန်း ၂ — Node နှစ်ခု တန်းဆက်ခြင်း

`steps: list[str]` နှင့် `value: int` ပါဝင်သော state သတ်မှတ်ပါ။ `double` node က `value` ကို ၂ ဆတိုးပြီး `steps` ထဲ `"double"` ထည့်ပါ။ `add_ten` node က `value` ကို ၁၀ တိုးပြီး `steps` ထဲ `"add_ten"` ထည့်ပါ။ Graph ကို `double → add_ten` အစီအစဉ်ဖြင့် ချိတ်ပါ။

**Hints:** List ကို update လုပ်တဲ့အခါ `state["steps"] + ["name"]` ဖြင့် list အသစ် ပြန်ပေးပါ — ရှိပြီး list ကို တိုက်ရိုက် မပြင်ပါနှင့်။
**Expected behavior:** `invoke({"value": 5, "steps": []})` ရလဒ်မှာ `value: 20`, `steps: ['double', 'add_ten']` ဖြစ်သည်။

## လေ့ကျင့်ခန်း ၃ — stream() ဖြင့် အဆင့်ဆင့် ကြည့်ခြင်း

လေ့ကျင့်ခန်း ၂ ရဲ့ graph ကို `invoke()` အစား `stream()` ဖြင့် run ပြီး chunk တိုင်းကို print လုပ်ပါ။ Chunk တိုင်းထဲ ဘယ် node က ဘာ update လုပ်လဲဆိုတာ သတိပြုပါ။

**Hints:** `for chunk in app.stream(inputs):` ပုံစံဖြင့် generator ကဲ့သို့ loop လုပ်ပါ။ Chunk က dictionary ဖြစ်သည်။
**Expected behavior:** Chunk နှစ်ခု ထွက်ပြီး ပထမတစ်ခုတွင် `double` node ရဲ့ update၊ ဒုတိယတွင် `add_ten` ရဲ့ update ပါဝင်သည်။

## လေ့ကျင့်ခန်း ၄ — Conditional edge ဖြင့် branch လုပ်ခြင်း

State ထဲ `count: int` ပါဝင်စေပါ။ `increment` node တစ်ခုနှင့် conditional edge function တစ်ခုရေးပါ — `count` သည် ၅ အောက်ဖြစ်နေလျှင် `increment` ကို ပြန်ခေါ်ပြီး၊ မဟုတ်လျှင် `END` သို့ သွားစေပါ။

**Hints:** `add_conditional_edges("increment", decide)` ကို အသုံးပြုပြီး `decide` function က node နာမည် (သို့) `END` ကို ပြန်ပေးပါ။
**Expected behavior:** `invoke({"count": 0})` ရလဒ်မှာ `{'count': 5}` ဖြစ်သည်။

## လေ့ကျင့်ခန်း ၅ — Branch နှစ်ခု ရွေးခြင်း

State ထဲ `number: int` နှင့် `label: str` ထည့်ပါ။ Conditional edge ဖြင့် — number စုံဆိုလျှင် `even` node, မစုံဆိုလျှင် `odd` node သို့ သွားစေပါ။ Node တိုင်းက `label` ထဲ `"even"` သို့မဟုတ် `"odd"` ထည့်ပါ။

**Hints:** `add_conditional_edges` ရဲ့ router function က `"even"` သို့မဟုတ် `"odd"` ဆိုသော string ပြန်ပေးပါ။ `START` ကို router နှင့် ချိတ်ရန် `add_conditional_edges(START, router)` ကိုလည်း အသုံးပြုနိုင်သည်။
**Expected behavior:** `invoke({"number": 4, "label": ""})` ရလဒ်မှာ `label: 'even'` ဖြစ်ပြီး `number: 5` ဖြင့် ခေါ်လျှင် `label: 'odd'` ဖြစ်သည်။

## လေ့ကျင့်ခန်း ၆ — Chain နှင့် state machine နှိုင်းယှဉ်ခြင်း

Function သုံးခ် — `call_model` (state ထဲ `"model"` ထည့်), `call_tool` (`"tool"` ထည့်), `check` (`"done"` ထည့်) — ဖြင့် state machine တစ်ခု တည်ဆောက်ပါ။ `call_tool` ပြီးလျှင် `call_model` ကို ပြန်သွားစေပြီး `check` node ကနေ `END` သို့ ရောက်စေပါ။ ဆိုလိုရင်း — chain သက်သက်နဲ့ ဒီ cycle ကို မဖော်ပြနိုင်ကြောင်း comment တစ်ကြောင်း ရေးပါ။

**Hints:** `call_model → call_tool → call_model` cycle ကို conditional edge ဖြင့် `call_tool` မှ `call_model` ကို တစ်ဆင့်၊ ထို့နောက် `check` သို့ သွားစေပါ — ဥပမာ `rounds` counter တစ်ခု သုံးပါ။
**Expected behavior:** Graph run ပြီးသောအခါ state ထဲ `"model"`, `"tool"`, `"done"` သုံးမျိုးလုံး ပါဝင်နေပြီး cycle အနည်းဆုံး တစ်ကြိမ် ဖြစ်ခဲ့သည်။
