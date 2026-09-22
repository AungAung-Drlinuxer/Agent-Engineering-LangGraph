# လေ့ကျင့်ခန်းများ

## လေ့ကျင့်ခန်း ၁ — Supervisor router အခြေခံ

`tech_agent` နဲ့ `general_agent` ဆိုတဲ့ node နှစ်ခုပါတဲ့ graph ဆောက်ပါ။ `supervisor` node က question ထဲမှာ "code" ဒါမှမဟုတ် "python" ပါရင် `tech_agent` ကို မဟုတ်ရင် `general_agent` ကို လမ်းညွှန်ပါ။

**Hints:** `add_conditional_edges` ကို သုံးပါ။ Supervisor ရဲ့ ရွေးချယ်ချက်ကို state key (ဥပမာ `"next"`) ထဲ သိမ်းပါ။

**Expected behavior:** `"how do I code a loop?"` ထဲ့လိုက်ရင် tech path ကို သွားပြီး `"what is the weather?"` ထဲ့လိုက်ရင် general path ကို သွားသည်။

## လေ့ကျင့်ခန်း ၂ — Supervisor loop နဲ့ termination

လေ့ကျင့်ခန်း ၁ က graph ကို ပြင်ပါ — worker တွေ ပြီးရင် supervisor ဆီ ပြန်ပြေးပြီး supervisor က အလုပ်ပြီးပြီလို့ မြင်ရင် `END` ကို ရောက်စေပါ။

**Hints:** state ထဲ `done` flag ထားပါ။ Supervisor က `next` ထဲ `END` ကို ဆက်ပေးနိုင်အောင် routing function ရေးပါ။

**Expected behavior:** worker တစ်ခု run ပြီးသွားရင် နောက်တစ်ခေါက် supervisor ကို ပြန်ရောက်ပြီး graph က ရပ်သည် (infinite loop မဖြစ်)။

## လေ့ကျင့်ခန်း ၃ — Swarm handoff

`front_agent` နဲ့ `order_agent` ဆိုတဲ့ swarm graph ဆောက်ပါ။ `front_agent` က message ထဲ "order" ပါရင် `order_agent` ကို တိုက်ရိုက် handoff လုပ်ပါ — supervisor မထည့်ပါနဲ့။

**Hints:** `front_agent` ရဲ့ return value ထဲ နောက်တစ်ခု ဘယ် agent လဲဆိုတာ ထည့်ပြီး conditional edge နဲ့ ချိတ်ပါ။

**Expected behavior:** `"I want to order pizza"` ထဲ့လိုက်ရင် `order_agent` က အလုပ်လုပ်ပြီး ရလဒ် ထွက်သည်။

## လေ့ကျင့်ခန်း ၄ — Parallel fan-out နဲ့ fan-in

Topic တစ်ခုကို `pro_agent` နဲ့ `con_agent` ဆီ တစ်ပြိုင်တည်း ပို့ပြီး `judge_agent` က ရလဒ်နှစ်ခုကို ပေါင်းပါ။ Reducer သုံးပြီး parallel writes ကို လက်ခံနိုင်အောင် လုပ်ပါ။

**Hints:** `results` key ကို `Annotated[list, operator.add]` နဲ့ ကြေညာပါ။ START ကနေ node နှစ်ခုလုံးဆီ edge ဆွဲပါ။

**Expected behavior:** judge ရဲ့ output ထဲ pro ရလဒ် နဲ့ con ရလဒ် နှစ်ခုလုံး ပါဝင်သည်။

## လေ့ကျင့်ခန်း ၅ — Subgraph နဲ့ state translation

State schema မတူတဲ့ parent graph တစ်ခုထဲ child graph တစ်ခုကို node အဖြစ် ထည့်ပါ။ Parent က `messages: list[str]` သုံးပြီး child က `question: str` သုံးပါ။

**Hints:** Wrapper node function တစ်ခုရေးပြီး parent state ကနေ child input ဆောက်၊ ရလဒ်ကို parent state ပြန်ရေးပါ။

**Expected behavior:** Parent ကို invoke လုပ်လိုက်ရင် child graph က လှိုင်းလျှို့မြင်မသိ run ပြီး ရလဒ် parent state ထဲ ပါလာသည်။

## လေ့ကျင့်ခန်း ၆ — Hop cost တွက်ချက်ခြင်း

`estimate_tokens(num_agents, supervisor_hops)` ဆိုတဲ့ function ရေးပြီး supervisor pattern နဲ့ direct handoff pattern က token cost ကွာခြားချက်ကို နမူနာ agent ရေအရေအတွက် (ဥပမာ ၃) နဲ့ နှိုင်းယှဉ်ပါ။ Hop တစ်ခုက ယူတင်း ၂၀၀၀ ဆိုပြီး သတ်မှတ်ပါ။

**Hints:** Supervisor pattern မှာ agent တစ်ခုချင်း လမ်းပြ hop တစ်ခု၊ agent hop တစ်ခု ဆိုပြီး တွက်ပါ။ Handoff မှာ agent hop တွေပဲ ရှိသည်။

**Expected behavior:** ရလဒ်နှစ်ခုကြား ကွာခြားချက် ဂဏန်းအားဖြင့် ပြနိုင်သည်။
