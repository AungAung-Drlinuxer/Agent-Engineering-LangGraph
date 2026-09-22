# လေ့ကျင့်ခန်းများ — Conditional Routing, Loops & Termination

## လေ့ကျင့်ခန်း ၁ — အခြေခံ Conditional Edge

`State` ထဲမှာ `score` (int) ရှိပါစေ။ `grade` node တစ်ခုရေးပြီး — score 50 နဲ့အထက်ဆို `pass_node` ကို၊ အောက်ဆို `fail_node` ကို သွားစေပါ။ Router function တစ်ခုနဲ့ `add_conditional_edges` သုံးပါ။

**Hints:** Router function က `state["score"] >= 50` စစ်ပြီး node နာမည် string return ပါ။ `add_conditional_edges("grade", router)` နဲ့ ချိတ်ပါ။
**Expected behavior:** score 80 ထည့်ရင် pass message ရပြီး၊ score 30 ထည့်ရင် fail message ရပါမယ်။

## လေ့ကျင့်ခန်း ၂ — Retry Cycle ထည့်ခြင်း

`generate` node က `attempts` တစ်ဆင့်တိုးပြီး၊ `attempts >= 2` ဖြစ်မှ `is_valid: True` ပြန်ပါစေ။ Router က invalid ဖြစ်နေရင် `generate` ကိုပြန် loop ပါစေ၊ valid ဖြစ်ရင် `END` ကို သွားပါစေပါ။

**Hints:** Router ထဲမှာ `if state["is_valid"]: return END` နဲ့ `return "generate"` သုံးပါ။ `add_conditional_edges` ရဲ့ parameter တတိယမှာ `[END, "generate"]` ပေးနိုင်ပါတယ်။
**Expected behavior:** Graph invoke တစ်ခုကြီးမှာ `generate` နှစ်ကြိမ် run ပြီးမှ ရပ်ပါမယ် — attempts 2 ဖြစ်နေပါမယ်။

## လေ့ကျင့်ခန်း ၃ — recursion_limit ကို စမ်းကြည့်ခြင်း

Loop မရပ်တဲ့ router တစ်ခု (အမြဲ node ကိုယ်တိုင်ကို ပြန်သွားတဲ့) ရေးပြီး `recursion_limit: 5` ထားကာ invoke လုပ်ပါ။ `GraphRecursionError` ကို catch လုပ်ပြီး message တစ်ခု print ပါ။

**Hints:** `from langgraph.errors import GraphRecursionError` import လုပ်ပါ။ `graph.invoke(state, config={"recursion_limit": 5})` သုံးပါ။
**Expected behavior:** Error ပေါ်ပြီး "recursion limit reached" နဲ့ တူတဲ့ message ရပါမယ် — program crash မဖြစ်ပါဘူး။

## လေ့ကျင့်ခန်း ၄ — Counter-based Guardrail

`refine` node တစ်ခုက state ထဲ `attempts` တိုးစေပါ။ Router က `attempts >= 4` ဖြစ်ရင် loop ထွက်ပြီး `END` သွားစေပါ — ဒါက `recursion_limit` မသုံးဘဲ loop ကို ကိုယ်တိုင် ကန့်သတ်တာပဲ။

**Hints:** Router ထဲမှာ attempts စစ်ပြီး `END` သို့မဟုတ် `"refine"` return ပါ။ `attempts` က invoke state မှာ 0 နဲ့ စပါ။
**Expected behavior:** Attempts 4 ရောက်မှ graph က ရပ်ပြီး၊ state ထဲ attempts 4 ဖြစ်နေပါမယ်။

## လေ့ကျင့်ခန်း ၅ — Deterministic Fallback

လေ့ကျင့်ခန်း ၄ အပေါ် အခြေခံပြီး — max attempts ကျော်ရင် `fallback` node ဆီ သွားစေပါ။ `fallback` node က state ထဲ `final` key ထဲမှာ "default" ဆိုတဲ့ သေချာတဲ့ တန်ဖိုးတစ်ခု ထည့်ပါ။

**Hints:** Router က `attempts >= 3` ဖြစ်ရင် `"fallback"` return ပါ။ `fallback` node ကနေ `END` ကို edge ထည့်ပါ။
**Expected behavior:** LLM မသုံးဘဲ attempts ကုန်ရင် သေချာတဲ့ default ရလဒ် ရပါမယ် — `final` = "default" ဖြစ်နေပါမယ်။

## လေ့ကျင့်ခန်း ၆ — ပေါင်းစပ်ခြင်း (မိုင်းလုံးဝင်ပုံစံ)

Workflow တစ်ခုရေးပါ — `draft` node က စာသားတစ်ခု draft လုပ်၊ `review` router က စာသားမှာ "good" စာလုံးပါရင် `END`၊ မပါရင် `draft` ကိုပြန်၊ ဒါပေမယ့် `attempts >= 3` ဖြစ်နေရင် `fallback` node ဆီ သွားပါ။ `fallback` က "partial result" ပြန်ပါစေ။

**Hints:** ဒီလေ့ကျင့်ခန်းမှာ routing function ထဲမှာ termination နှစ်မျိုး (quality + max attempts) တွဲစစ်ရမယ် — quality check ကို အရင်စစ်ပါ။ Draft node က တစ်ကြိမ်မှာ "good" ပါတဲ့ စာသား ထုတ်ပါစေ simulate လုပ်နိုင်ပါတယ်။
**Expected behavior:** Draft က quality မမှီတဲ့အခါ သုံးကြိမ်ပြီးရင် fallback ကို ရောက်ပြီး "partial result" ထွက်ပါမယ် — အချိန်အနန္တ loop မဖြစ်ပါဘူး။
