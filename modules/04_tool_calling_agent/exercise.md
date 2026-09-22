## လေ့ကျင့်ခန်း ၁ — Tool အတွက် Pydantic schema ဖန်တီးခြင်း

`pydantic` မှ `BaseModel` ကို အသုံးပြု၍ ရာသီဥတုစစ်ဆေးသည့် tool ၏ argument schema ကို သတ်မှတ်ပါ။ Field နှစ်ခု ပါဝင်ရမည် — `city` (string, စာသားမရှိလျှင် error) နှင့် `unit` (`"celsius"` သို့မဟုတ် `"fahrenheit"` ကိုသာ ခွင့်ပြုသော literal)။ ထို့နောက် `@tool` decorator ဖြင့် tool function ချိတ်ဆက်ပါ။

**Hints:** `from pydantic import BaseModel, Field` နှင့် `from typing import Literal` ကို အသုံးပြုပါ။ Docstring သည် LLM အား tool ၏ ရည်ရွယ်ချက်ကို ပြောပြသဖြင့် ရေးပေးပါ။
**Expected behavior:** `get_weather.invoke({"city": "Yangon"})` ခေါ်ဆိုလျှင် အောင်မြင်စွာ လုပ်ဆောင်ပြီး `unit` မပေးလျှင် default တန်ဖိုး အသုံးပြုသည်။ `unit="kelvin"` ပေးလျှင် validation error ဖြစ်သည်။

## လေ့ကျင့်ခန်း ၂ — bind_tools ဖြင့် model ကို tool ချိတ်ခြင်း

`ChatOpenAI` (သို့မဟုတ် သင့်စွမ်းအင်နှင့်သင့်လျော်သော `ChatModel`) တစ်ခုကို `init_chat_model` ဖြင့် ဖန်တီးပြီး လေ့ကျင့်ခန်း ၁ ၏ tool ကို `bind_tools` ဖြင့် ချိတ်ဆက်ပါ။ `"What is the weather in Mandalay in fahrenheit?"` ဟူသော message တစ်ချက် ပို့ကြည့်ပါ။ Response ထဲမှ `tool_calls` attribute ကို စစ်ဆေးပြီး tool name၊ arguments ကို ပရင့်ထုတ်ပါ။

**Hints:** `response.tool_calls[0]["args"]` တွင် LLM က ဖန်တီးပေးလိုက်သော argument dictionary ပါဝင်သည်။ Model က tool မခေါ်ဘဲ အဖြေပြန်လျှင် temperature နိမ့်စွာ ထားကြည့်ပါ။
**Expected behavior:** Response တွင် `tool_calls` list မှာ အလွတ်မဟုတ်ဘဲ `get_weather` ဟူသော name နှင့် `{"city": "Mandalay", "unit": "fahrenheit"}` ကဲ့သို့သော args ပါဝင်သည်။

## လေ့ကျင့်ခန်း ၃ — ToolNode ပါသော agent graph ဆောက်ခြင်း

`StateGraph` တစ်ခုကို ဆောက်ပါ — `agent` node (bind_tools လုပ်ထားသော model) နှင့် `tools` node (`ToolNode([get_weather])`)။ Conditional edge ကို `tools_condition` ဖြင့် အသုံးပြုပြီး tool call ရှိလျှင် `tools` node သို့၊ မရှိလျှင် `END` သို့ လမ်းကြောင်းခွဲပါ။ Graph ကို compile လုပ်ကာ `"What is the weather in Yangon?"` ကို တစ်ကြိမ် run ကြည့်ပါ။

**Hints:** `from langgraph.graph import StateGraph, MessagesState, START, END` နှင့် `from langgraph.prebuilt import ToolNode, tools_condition` ကို import လုပ်ပါ။ `add_conditional_edges("agent", tools_condition)` သည် default အားဖြင့် `"tools"` နှင့် `END` ကို ဆက်သည်။
**Expected behavior:** Graph run ပြီးဆုံးသည့်အခါ နောက်ဆုံး message တွင် ရာသီဥတုအချက်အလက် (သင်ဖန်တီးထားသော mock အဖြေ) ပါဝင်သည်။

## လေ့ကျင့်ခန်း ၄ — Parallel tool calls စမ်းသပ်ခြင်း

`get_weather` အပြင် `get_time` ဟူသော tool နှစ်ခုမြောက်ကို ထပ်ထည့်ပါ။ `"What's the weather in Yangon and also in Mandalay?"` ကဲ့သို့ တစ်ပြိုင်တည်း မေးချင်သော prompt တစ်ခုကို graph သို့ ပို့ပါ။ `ToolNode` မှ ထုတ်ပေးသော `AIMessage` ရှိ `tool_calls` အရေတွက်ကို ရေတွက်ပြီး node တစ်ခုချင်းစီ လုပ်ဆောင်ချက်ကို trace ပြပါ။

**Hints:** ToolNode သည် message တစ်ခုထဲမှ tool calls များစွာကို အလိုအလျောက် တစ်ပြိုင်တည်း run ပြီး `ToolMessage` များစွာ ထုတ်ပေးသည်။ မရလျှင် `"check the weather in Yangon AND the weather in Mandalay"` ဟု ရှင်းရှင်းလင်းလင်း ရေးပါ။
**Expected behavior:** Agent တစ်ကြိမ် run တွင် tool calls နှစ်ခု ပါဝင်ပြီး နှစ်ခုလုံး၏ရလဒ်များကို နောက်ဆုံးအဖြေတွင် ပေါင်းစပ်ဖော်ပြသည်။

## လေ့ကျင့်ခန်း ၅ — Tool error contract နှင့် retry

`get_weather` ၏ `city` argument မတွေ့လျှင် `ValueError("City not found")` တွင်ထစ်စေပါ။ ထို့နောက် tool ကို ကာရံ (wrap) လုပ်၍ error ကို `ToolMessage` အဖြစ် `status="error"` ဖြင့် state ထဲ ပြန်ပို့ပြီး LLM က ထပ်တွေးစေသော retry pattern တစ်ခုကို ရေးပါ — ဥပမာ agent က မြို့အမည်ကို ပြင်ပြီး နောက်တစ်ကြိမ် ထပ်ခေါ်သည်။

**Hints:** `ToolNode` တွင် `handle_tool_errors=True` ကို သတ်မှတ်နိုင်သည် — error string ကို LLM ဆီ ပြန်ပို့သဖြင့် model က မှားယွင်းချက်ကို ဖတ်ပြီး ပြင်ဆင်နိုင်သည်။ Error သည် exception အဖြစ် graph တစ်ခုလုံးကို မဖျက်ဆီးစေရ။
**Expected behavior:** တည်ရှိသောမြို့အမည် လွဲ၍ မေးလျှင် graph သည် error ဖြစ်ပြီးနောက် agent က မှန်ကန်သောမြို့ဖြင့် သင့်လျော်စွာ ထပ်မံခေါ်ဆိုကာ အမှားအဖြေ မပေးပါ။

## လေ့ကျင့်ခန်း ၆ — Argument validation နှင့် tool permission narrowing

မတူသော permission အဆင့်နှစ်ခုရှိသော multi-tool system တစ်ခု ဆောက်ပါ — အောက်ဆင့် user များအတွက် `get_weather` (သာရနိုင်)၊ အထက်ဆင့်အတွက် `send_report` (email ပို့နိုင်)။ (၁) `send_report` ၏ `recipient` argument ကို Pydantic email validator ဖြင့် စစ်ဆေးပါ၊ (၂) user ၏ role အပေါ်မူတည်၍ runtime မှာ bind လုပ်မယ့် tool list ကို ရွေးချယ်ပါ၊ (၃) အောက်ဆင့် user ၏ graph ကို `"send a report to admin@x.com"` ဖြင့် run လျှင် `send_report` လုံးဝ မရနိုင်ကြောင်း အဖြေပြန်သည်ကို စမ်းသပ်ပါ။

**Hints:** `pydantic` ၏ `EmailStr` (email-validator package လိုအပ်သည်) သို့မဟုတ် ကိုယ်ပိုင် validator ကို သုံးပါ။ Permission narrowing သည် မတူညီသော `bind_tools([...])` list နှစ်ခု ဖန်တီးခြင်းဖြင့်သာ လုပ်ဆောင်နိုင်သည် — LLM မတွင်မချိတ်ထားသော tool ကို ခေါ်ဆိုလို့ မရပါ။
**Expected behavior:** အောက်ဆင့် user graph တွင် `send_report` ၏ tool call လုံးဝ မဖြစ်ပေါ်ဘဲ ရာသီဥတုမေးခွန်းများကိုမူ မှန်ကန်စွာ ဖြေဆိုသည်။ အထက်ဆင့် user တွင် email format မှားလျှင် validation error နှင့် ထပ်စဉ်းစားခိုင်းသော အခြေအနေ ဖြစ်ပေါ်သည်။
