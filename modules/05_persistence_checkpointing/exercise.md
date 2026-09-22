# လေ့ကျင့်ခန်းများ — Persistence, Checkpointing & Time Travel

## လေ့ကျင့်ခန်း ၁ — InMemorySaver နဲ့ state ဆက်လက်တင်ရန်

`count` field တစ်ခုပါတဲ့ state နဲ့ counter node တစ်ခုပါတဲ့ graph တစ်ခုကို `InMemorySaver` သုံးပြီး compile လုပ်ပါ။ thread_id `"t1"` နဲ့ invoke နှစ်ချိန် run ပြီး ဒုတိယ run မှာ input `None` ပေးပါ။ count က ဆက်တိုးနေတာ သက်သေပြပါ။

**Hints:** `builder.compile(checkpointer=InMemorySaver())` သုံးပါ၊ state class မှာ `count` default `0` ဖြင့် စစ်ဆေးပါ (`state.get("count", 0)`)။

**Expected behavior:** ပထမ invoke → `{'count': 1}`၊ ဒုတိယ invoke (input None) → `{'count': 2}`။

## လေ့ကျင့်ခန်း ၂ — thread_id နှစ်ခုက သီးသန့်ဖြစ်ကြောင်း ပြရန်

လေ့ကျင့်ခန်း ၁ ရဲ့ graph ကိုပဲ သုံးပြီး thread_id `"user-A"` နဲ့ `"user-B"` တို့နဲ့ invoke လုပ်ပါ။ user-A မှာ နှစ်ချိန် run ပြီး၊ user-B မှာ တစ်ချိန် run ပါ။ နှစ် user ရဲ့ count တွေ တစ်ခုကို တစ်ခု မသက်ရောက်ကြောင်း ပြပါ။

**Hints:** config dict နှစ်ခု ခွဲပြီး `{"configurable": {"thread_id": ...}}` သတိပြုပါ။

**Expected behavior:** user-A → 2၊ user-B → 1 ဖြစ်ပြီး user-A run နောက်ဆုံးတစ်ချက် user-B ကို မထိခိုက်ပါ။

## လေ့ကျင့်ခန်း ၃ — SqliteSaver နဲ့ ဖိုင်ထဲ သိမ်းရန်

`SqliteSaver` သုံးပြီး `checkpoints.db` ဖိုင်ထဲ checkpoint သိမ်းပါ။ `count` graph ကို run ပြီး process ပိတ်ရင်လည်း (object အသစ်ဆောက်ရင်လည်း) တူညီတဲ့ file နဲ့ thread_id ကနေ state ပြန်ရကြောင်း ပြပါ။

**Hints:** `pip install langgraph-checkpoint-sqlite`၊ `sqlite3.connect("checkpoints.db", check_same_thread=False)`၊ checkpointer အသစ်နဲ့ graph အသစ် compile လုပ်ပါ။

**Expected behavior:** Program နှစ်ချက်ဆက် run ရင် ဒုတိယအကြိမ်မှာ count က ပထမအကြိမ်ရဲ့ တန်ဖိုးကနေ ဆက်တိုးပါ။

## လေ့ကျင့်ခန်း ၄ — Fail ဖြစ်ပြီးနောက် တူညီတဲ့ thread မှာ resume လုပ်ရန်

Node တစ်ခုက ပထမ ကြိုးစားမှုမှာ `RuntimeError` တက်စေပြီး၊ try/except နဲ့ဖမ်းပြီး နောက်ဆုံးတွင် တူညီတဲ့ thread_id မှာ ပြန် invoke လုပ်ပြီး အောင်မြင်စေပါ။

**Hints:** state ထဲ `attempt` field ထည့်ပြီး 0 ဖြစ်ရင် raise၊ 1 ဖြစ်ရင် အောင်မြင်အောင် ရေးပါ။ Exception ဖမ်းပြီး နောက် invoke မှာ `{"attempt": 1}` ပေးပါ။

**Expected behavior:** Exception တစ်ကြိမ် ပေါ်ပြီး နောက်ဆုံးမှာ `{'attempt': 1, 'message': 'done'}` ကို ရရှိပါ။

## လေ့ကျင့်ခန်း ၅ — Checkpoint history ကို ဖတ်ရန် (time travel view)

Graph တစ်ခုကို တစ် thread မှာ သုံးကြိမ် invoke လုပ်ပြီး `get_state_history` နဲ့ checkpoint အားလုံးရဲ့ `values` တွေကို အစဉ်လိုက် ပရင့်ထုတ်ပါ။

**Hints:** `for snap in graph.get_state_history(config):` သုံးပြီး `snap.values` ကို ကြည့်ပါ။ History က နောက်ဆုံး checkpoint ကနေ စပြီး ပြောင်းပြန်အစီအစဉ် ရောက်တတ်ပါတယ်။

**Expected behavior:** ပရင့်ထုတ်တဲ့ state တွေက အသစ်ဆုံးကနေ အရင်ဆုံးအထိ ပြောင်းပြန်အစီအစဉ်နဲ့ ပေါ်ပါ။

## လေ့ကျင့်ခန်း ၆ — Long-term store နဲ့ user profile မျှဝေရန်

`InMemoryStore` သုံးပြီး user id တစ်ခုအတွက် profile (ဥပမာ — နာမည်) ကို သိမ်းပြီး၊ thread_id မတူတဲ့ run နှစ်ခုကနေ တူညီတဲ့ profile ကို ဖတ်ယူပြပါ။

**Hints:** `store.put(("profiles", "u1"), "name", {"value": "Aung Aung"})` နဲ့ သိမ်းပြီး `store.get(("profiles", "u1"), "name")` နဲ့ ဖတ်ပါ။ Thread_id အသစ်နဲ့ invoke လုပ်လည်း store က thread နဲ့ မသက်ဆိုင်ကြောင်း ပြပါ။

**Expected behavior:** ကွဲပြားတဲ့ thread နှစ်ခုကနေ store ထဲက တူညီတဲ့ profile ကို ရရှိပါ။
