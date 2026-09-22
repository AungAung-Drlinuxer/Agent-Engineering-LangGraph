# Tool-Calling Agents (LangGraph)

## bind_tools နဲ့ Pydantic / JSON Schema

### ဘာကို ဆိုလိုတာလဲ
`bind_tools` ဆိုတာ LangChain chat model တစ်ခုပေါ်မှာ tools တွေကို ချိတ်ဆက်ပေးတဲ့ method တစ်ခုဖြစ်ပါတယ်။ Tool တစ်ခုရဲ့ input structure ကို သတ်မှတ်ဖို့ Pydantic model (သို့) JSON Schema ကို သုံးလို့ရပါတယ်။

### ဘာကြောင့် လဲ
LLM က tool ကို ခေါ်တဲ့အခါ argument တွေကို ဘယ်လို format နဲ့ပေးရမလဲ ဆိုတာကို မသိရင် မှားယွင်းတဲ့ input တွေ ထုတ်ပေးတတ်ပါတယ်။ Pydantic schema တစ်ခုက tool ရဲ့ "စာချုပ်" (contract) ဖြစ်ပြီး LLM က အဲဒီ schema အတိုင်း arguments တွေ ထုတ်ပေးစေမှာ ဖြစ်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
Pydantic class တစ်ခုကို `description` နဲ့ field တိုင်းရဲ့ `Field(description=...)` တွေ ရေးပြီး `@tool` decorator နဲ့ tool အဖြစ် ပြောင်းပါတယ်။ အဲဒီ tool ကို `model.bind_tools([...])` ထဲ ထည့်ပေးလိုက်ရင် schema က function-calling format အဖြစ် model ဆီ ပို့လိုက်ပါတယ်။

### ဥပမာ
```python
from pydantic import BaseModel, Field
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

class WeatherArgs(BaseModel):
    city: str = Field(description="City name in English, e.g. 'Yangon'")
    unit: str = Field(default="celsius", description="'celsius' or 'fahrenheit'")

@tool(args_schema=WeatherArgs)
def get_weather(city: str, unit: str = "celsius") -> str:
    # Look up weather for the given city (stubbed here)
    return f"Weather in {city}: 31 {unit}, partly cloudy"

model = ChatOpenAI(model="gpt-4o-mini")
bound_model = model.bind_tools([get_weather])
print(get_weather.name)
print(get_weather.args_schema.model_json_schema()["properties"]["city"]["type"])
# Expected output:
# get_weather
# string
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Schema ရှင်းလင်းလေ LLM က arguments မှန်ကန်စွာ ထုတ်လေ ဖြစ်ပါတယ်။ အထူးသဖြင့် enum values၊ default values နဲ့ field descriptions တွေက agent ရဲ့ tool အသုံးပြုမှု တိကျမှုကို တိုက်ရိုက် သက်ရောက်ပါတယ်။

## ToolNode နဲ့ Graph ထဲ ထည့်တာ

### ဘာကို ဆိုလိုတာလဲ
`ToolNode` က tool တွေကို execute လုပ်ပေးတဲ့ LangGraph node တစ်ခုဖြစ်ပါတယ်။ Agent node (LLM) နဲ့ တွဲပြီး request-response loop တစ်ခု ဖြစ်စေပါတယ်။

### ဘာကြောင့် လဲ
LLM က `AIMessage` ထဲမှာ `tool_calls` တွေ ထည့်ပေးပြီး အဲဒီ calls တွေကို တကယ် run ပေးမယ့်သူ လိုအပ်ပါတယ်။ `ToolNode` က အဲဒီ အလုပ်ကို တာဝန်ယူပြီး results တွေကို `ToolMessage` တွေအဖြစ် ပြန်ထည့်ပေးပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
Agent node က model (bind_tools လုပ်ပြီးသား) ကို ခေါ်ပါတယ်။ Response မှာ tool_calls ပါရင် `tools` (ToolNode) ဆီ conditional edge နဲ့ ပို့ပြီး၊ ToolNode က result ထုတ်ပြီး agent ဆီ ပြန် loop လုပ်ပါတယ်။ tool_calls မပါရင် END ဆီ ဆက်ပါတယ်။

### ဥပမာ
```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.prebuilt import ToolNode

tools = [get_weather]
tool_node = ToolNode(tools)
agent_model = model.bind_tools(tools)

def agent(state: MessagesState):
    # Simply call the bound model with the message history
    return {"messages": [agent_model.invoke(state["messages"])]}

def should_continue(state: MessagesState):
    last = state["messages"][-1]
    return "tools" if last.tool_calls else END

builder = StateGraph(MessagesState)
builder.add_node("agent", agent)
builder.add_node("tools", tool_node)
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", should_continue, ["tools", END])
builder.add_edge("tools", "agent")
graph = builder.compile()
result = graph.invoke({"messages": [("user", "What is the weather in Yangon?")]})
print(result["messages"][-2].content)
# Expected output:
# Weather in Yangon: 31 celsius, partly cloudy
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Tool execution၊ message history ထဲ `ToolMessage` ထည့်တာ၊ error handling တို့ကို `ToolNode` က စံတစ်ခုအတိုင်း လုပ်ပေးလို့ code ပိုရှင်းပြီး bug လျှော့စေပါတယ်။

## Parallel Tool Calls

### ဘာကို ဆိုလိုတာလဲ
Model တစ်ခုက response တစ်ခုတည်းထဲမှာ tool calls တွေကို တစ်ခုထက် ပို ထုတ်ပေးနိုင်ပါတယ်။ `ToolNode` က default အားဖြင့် အဲဒီ calls တွေကို တစ်ချိန်တည်း (concurrently) run ပေးပါတယ်။

### ဘာကြောင့် လဲ
"ရန်ကုန်နဲ့ မန္တလေး ရာသီဥတု ဘာလဲ" လို့မေးရင် tool call နှစ်ခု ဆိုင်းပြီး တစ်ချက်ချင်း run ရင် နှောင့်နှေးပါတယ်။ Parallel execution က စုစုပေါင်းအချိန်ကို သိသိသာသာ လျှော့ပေးပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
`AIMessage.tool_calls` က list ဖြစ်ပါတယ်။ `ToolNode` က list ထဲက call တိုင်းကို async/sync နည်းနဲ့ ညီတူ run ပြီး၊ ရလာဒတွေကို `tool_call_id` အလိုက် အစဉ်လိုက် ပြန်တွဲပေးပါတယ်။ Response order က request order အတိုင်း ဖြစ်နေအောင် စီစဉ်ပေးပါတယ်။

### ဥပမာ
```python
result = graph.invoke({
    "messages": [("user", "Compare the weather in Yangon and Mandalay.")]
})
calls = result["messages"][-2].tool_calls if result["messages"][-2].tool_calls else []
print([(c["name"], c["args"]) for c in calls])
# Expected output (illustrative, exact wording may vary):
# [('get_weather', {'city': 'Yangon'}), ('get_weather', {'city': 'Mandalay'})]
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Parallel-safe မဟုတ်တဲ့ tool (ဥပမာ — shared state ရေးတဲ့ tool) တွေအတွက် ToolNode ရဲ့ concurrency က အန္တရာယ်ရှိပါတယ်။ ဒါကြောင့် tool တစ်ခုစီရဲ့ thread-safety ကို စဉ်းစားရပါမယ်။

## Tool Error Contracts နဲ့ Retries

### ဘာကို ဆိုလိုတာလဲ
Tool တစ်ခု fail ရင် exception ကို crash မဖြစ်စေဘဲ LLM ဆီ error message အဖြစ် ပြန်ပို့တဲ့ နည်းလမ်းကို ဆိုလိုပါတယ်။

### ဘာကြောင့် လဲ
Graph တစ်ခုလုံး crash သွားရင် agent က ပြန်လည်ဆက်လက် အလုပ်လုပ်လို့ မရပါဘူး။ Error ကို `ToolMessage` အဖြစ် ပြန်ပေးရင် LLM က error ကို ဖတ်ပြီး arguments ပြင်ပြီး ထပ်စမ်းနိုင်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
`ToolNode(tools, handle_tool_errors=True)` (default ဖြစ်ပါတယ်) ဆိုရင် tool ထဲက raise လုပ်လိုက်တဲ့ exception ကို string အဖြစ် ပြောင်းပြီး `ToolMessage` ထဲ ထည့်ပေးပါတယ်။ Custom handler function တစ်ခုကိုလည်း ပေးလို့ရပါတယ်။ Agent loop က error မျိုး feedback ရပြီး retry လုပ်ပါတယ် — ဒါက "self-correcting" agent ရဲ့ အခြေခံ ဖြစ်ပါတယ်။

### ဥပမာ
```python
from langgraph.prebuilt import ToolNode

@tool
def divide(a: float, b: float) -> str:
    # Raise a clear error the agent can understand and fix
    if b == 0:
        raise ValueError("Cannot divide by zero. Choose a non-zero divisor.")
    return str(a / b)

safe_node = ToolNode([divide], handle_tool_errors=True)
state = {"messages": []}
from langchain_core.messages import AIMessage
state["messages"].append(
    AIMessage(content="", tool_calls=[
        {"name": "divide", "args": {"a": 10, "b": 0}, "id": "call_1", "type": "tool_call"}
    ])
)
out = safe_node.invoke(state)
print(out["messages"][0].content)
# Expected output:
# Cannot divide by zero. Choose a non-zero divisor.
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Production မှာ tool တွေက external APIs တွေနဲ့ ချိတ်တာမို့ network errors၊ rate limits တွေ ဖြစ်နိုင်ပါတယ်။ Error message တွေကို ရှင်းလင်းစွာ ရေးပေးရင် LLM ရဲ့ retry အောင်မြင်နိုင်မှု တိုးပါတယ်။ Infinite retry loop တွေ မဖြစ်စေဖို့ `recursion_limit` ကိုလည်း သတ်မှတ်ပါ။

## Tool Arguments တွေကို Validate လုပ်တာ

### ဘာကို ဆိုလိုတာလဲ
LLM ထုတ်ပေးတဲ့ arguments တွေက schema နဲ့ ကိုက်ညီမကိုက်ညီ စစ်ဆေးပြီးမှ tool ကို run လုပ်တာကို ဆိုလိုပါတယ်။

### ဘာကြောင့် လဲ
Model တွေက တစ်ခါတစ်ရံ hallucinated argument (မရှိတဲ့ city name၊ မှားတဲ့ type) တွေ ထုတ်ပေးတတ်ပါတယ်။ Validation မရှိရင် tool ထဲကို ဆိုးရွမ်းတဲ့ input တွေ ရောက်သွားနိုင်ပါတယ်။

### ဘယ်လို အလုပ်လုပ်လဲ
Pydantic `args_schema` က type coercion နဲ့ constraint checks (ဥပမာ `Literal`၊ `Field(ge=...)`) တွေ လုပ်ပေးပါတယ်။ Validation ပြီးရင် validated value တွေကိုသာ tool function ဆီ ပို့ပါတယ်။ Fail ရင် `ValidationError` raise လုပ်ပြီး error contract အတိုင်း LLM ဆီ ပြန်ပို့ပါတယ်။

### ဥပမာ
```python
from typing import Literal
from pydantic import BaseModel, Field

class UnitArgs(BaseModel):
    city: str = Field(min_length=2)
    unit: Literal["celsius", "fahrenheit"]

@tool(args_schema=UnitArgs)
def get_weather_checked(city: str, unit: str = "celsius") -> str:
    return f"Weather in {city}: 31 {unit}"

try:
    get_weather_checked.invoke({"city": "Y", "unit": "kelvin"})
except Exception as e:
    print(type(e).__name__)
# Expected output:
# ValidationError
```

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Permission က ကျပ်တဲ့ tool တွေ (database writes၊ payments) အတွက် argument validation က security ရဲ့ ပထမဆင့် ဖြစ်ပါတယ်။ SQL injection ကဲ့သို့ input attacks တွေကို အစောဆုံး ဖမ်းဆုပ်နိုင်ပါတယ်။

## Tool Permissions တွေကို ကျဉ်းမြောင်းစေတာ

### ဘာကို ဆိုလိုတာလဲ
Agent တစ်ခုကို သူ တကယ်လိုအပ်တဲ့ tools တွေကိုသာ ပေးတာကို ဆိုလိုပါတယ် — ဥပမာ read-only tools တွေပဲ ပေးတာမျိုး။

### ဘာကြောင့် လဲ
Tools အများအပြား ပေးထားရင် LLM က မလိုအပ်တဲ့ tool ကို ရွေးတာ၊ permission ကျယ်တဲ့ tool ကို မှားသုံးတာတွေ ဖြစ်နိုင်ပါတယ်။ အန္တရာယ်ရှိတဲ့ လုပ်ဆောင်ချက် (delete၊ payment) တွေကို agent လက်ထဲ မပေးရင် မလုပ်နိုင်တော့ပါဘူး။

### ဘယ်လို အလုပ်လုပ်လဲ
Tool set တွေကို agent တစ်ခုစီ အလိုက် ခွဲပြီး `bind_tools` မှာ တစ်ခုစီ ကွဲပြားစွာ သတ်မှတ်ပါတယ်။ Runtime မှာတော့ `tool_call_id` ကို စစ်ပြီး ခွင့်မပြုတဲ့ call တွေကို ပယ်ဖျောက်နိုင်ပါတယ်။ ဒါက "allowlist" နည်းလမ်း ဖြစ်ပါတယ်။

### ဥပမာ
```python
ALLOWED_TOOLS = {"get_weather", "get_weather_checked"}

def gated_tool_node(state: MessagesState) -> dict:
    # Reject tool calls that are not in the allowlist
    messages = state["messages"]
    from langchain_core.messages import ToolMessage
    results = []
    for call in messages[-1].tool_calls:
        if call["name"] in ALLOWED_TOOLS:
            results.append(call)
        else:
            results.append(ToolMessage(
                content=f"Tool '{call['name']}' is not permitted.",
                tool_call_id=call["id"],
            ))
    return tool_node.invoke({"messages": messages[:-1] + [messages[-1]]})
```
*(ပိုမှန်ကန်တာက — rejection ပြုလုပ်တဲ့ logic ကို node တစ်ခုအဖြစ် သီးခြားရေးပြီး graph ထဲ ထည့်သွင်းဖို့ ဖြစ်ပါတယ်။)*

### လက်တွေ့မှာ ဘာကြောင့် အရေးကြီးလဲ
Production system တွေမှာ agent က user data ကို မထိခိုက်စေဖို့၊ side-effect ရှိတဲ့ operations တွေကို လူ (human-in-the-loop) က confirm လုပ်စေဖို့ permission design က မဖြစ်မနေ လိုအပ်ပါတယ်။ Least privilege သဘောတရားကို အသုံးချရပါမယ်။

## အနှစ်ချုပ်
- `bind_tools`
