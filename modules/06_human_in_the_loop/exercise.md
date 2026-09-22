# လေ့ကျင့်ခန်းများ — Human-in-the-Loop Approval & Interrupts

အောက်မှာ လေ့ကျင့်ခန်း ၆ ခုပါဝင်ပါတယ်။ အားလုံးက `MemorySaver` checkpointer နဲ့ compile လုပ်ထားတဲ့ StateGraph တစ်ခု အသုံးပြုဖို့ လိုအပ်ပါတယ်။

## လေ့ကျင့်ခန်း ၁ — ပထမ Approval Gate

`interrupt()` ကိုသုံးပြီး အောက်ပါ graph တည်ဆောက်ပါ — `START → plan → approve → execute → END`။ `plan` node က `{"action": "Send email to team@example.com"}` ကို state ထဲ ထည့်ပါ။ `approve` node က `interrupt()` ခေါ်ပြီး ရပ်ပါ။ ပထမ invoke မှာ `__interrupt__` key ပါလာသလား စစ်ပါ။

**Hints:** `MemorySaver` မပါဘဲ `interrupt()` က အလုပ်မလုပ်ပါဘူး — checkpointer နဲ့ compile လုပ်ပါ။ Interrupt payload က dict တစ်ခု ဖြစ်စေပါ။

**Expected behavior:** ပထမ invoke မှာ graph က approve node မှာ ရပ်သွားပြီး result state ထဲ `__interrupt__` ပါလာပါမယ်။

## လေ့ကျင့်ခန်း ၂ — State ကို Inspect လုပ်ခြင်း

လေ့ကျင့်ခန်း ၁ ရဲ့ graph မှာ interrupt ဖြစ်နေတဲ့အခါ `graph.get_state(config)` ကို ခေါ်ပြီး `.values` နဲ့ `.next` ကို print လုပ်ပါ။ ဘယ် node မှာ စောင့်နေလဲ ဖော်ပြပါ။

**Hints:** `.next` က tuple တစ်ခု ပြန်ပါတယ် — ရပ်နေတဲ့ node ရဲ့နာမည် ပါဝင်ပါမယ်။

**Expected behavior:** `values` ထဲမှာ `action` တန်ဖိုးပြီး `next` မှာ `('approve',)` လိုမျိုး ပါဝင်ပါမယ်။

## လေ့ကျင့်ခန်း ၃ — Resume နဲ့ Execute

လေ့ကျင့်ခန်း ၁-၂ အပေါ် အခြေခံပြီး `graph.invoke(Command(resume={"approved": True}), config)` နဲ့ ဆက်လည်ပတ်စေပါ။ `execute` node က approved ဖြစ်ရင် `"action executed"` ဆိုတဲ့ message ထည့်ပါ၊ reject ဖြစ်ရင် `"action rejected"` ထည့်ပါ။ နှစ်မျိုးလုံး စမ်းပါ။

**Hints:** `interrupt()` ရဲ့ return value က resume dict ဖြစ်ပါတယ် — `decision["approved"]` လို့ ဖတ်ပါ။ တူညီတဲ့ `thread_id` သုံးပါ။

**Expected behavior:** Approve လုပ်ရင် final state မှာ `"action executed"` ပြီး reject လုပ်ရင် `"action rejected"` ပါဝင်ပါမယ်။

## လေ့ကျင့်ခန်း ၄ — Edit Before Approve

Interrupt ဖြစ်နေတဲ့အခါ `graph.update_state(config, {"action": ...})` နဲ့ action ကို တည်းဖြတ်ပြီးမှ approve လုပ်ပါ — ဥပမာ email address ကို ပြင်ပါ။ Resume လုပ်တဲ့အခါ execute node က ပြင်ပြီးတဲ့ action ကို အသုံးပြုသလား စစ်ပါ။

**Hints:** `update_state` က checkpoint ကို တိုက်ရိုက် ပြင်ပါတယ် — resume မလုပ်ခင်မှာ ခေါ်ပါ။

**Expected behavior:** Final output မှာ ပြင်ပြီးတဲ့ action တန်ဖိုး ပါဝင်ပါမယ်။

## လေ့ကျင့်ခန်း ၅ — Timeout Auto-Reject

Pending interrupt တစ်ခုအတွက် အချိန်စစ်တဲ့ logic ရေးပါ — state ထဲမှာ `created_at` (timestamp) သိမ်းပြီး၊ helper function တစ်ခုက သက်တမ်းကျရင် `Command(resume={"approved": False, "reason": "timeout"})` နဲ့ auto-reject လုပ်ပါ။

**Hints:** `time.time()` နဲ့ timestamp သိမ်းပြီး helper function မှာ `get_state(config)` က ဒီ pending run လား စစ်ပါ — `.next` ဗလာဖြစ်ရင် run က ပြီးပါပြီ။

**Expected behavior:** Timeout helper ကို ခေါ်ရင် run က reject နဲ့ အဆုံးသတ်ပြီး audit record ထဲ `"reason": "timeout"` ပါဝင်ပါမယ်။

## လေ့ကျင့်ခန်း ၆ — Audit Trail နဲ့ Escalation

`approvals: Annotated[list, operator.add]` ကို state ထဲထည့်ပြီး တစ်ခြောက် approve/reject တိုင်းက record (decision, reviewer, timestamp, reason) တစ်ခု တွင်းထည့်ပါ။ Reject ဖြစ်ရင် `escalate` node တစ်ခုကို သွားပြီး `"escalated to manager"` message ထည့်စေပါ။

**Hints:** Reject path ကိ်ု `add_conditional_edges` နဲ့ `approve` node ကနေ `escalate` (reject) သို့မဟုတ် `execute` (approve) ဆီ ရွေးပါ။ Audit log က append-only ဖြစ်ဖို့ `operator.add` reducer သုံးပါ။

**Expected behavior:** Reject လုပ်တဲ့ run မှ approvals record တစ်ခု ပြီး `"escalated to manager"` message ပါဝင်ပါမယ်။ Approve လုပ်တဲ့ run မှာ escalation မရှိပါ။
