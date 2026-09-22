# အဖြေမှတ်တမ်း — Tool-Calling Agents (LangGraph)

## လေ့ကျင့်ခန်း ၁ — Pydantic schema နှင့် bind_tools သုံးခြင်း

```python
from pydantic import BaseModel, Field
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

# Define the input schema with Pydantic so the model sees structured args
class WeatherInput(BaseModel):
    city: str = Field(description="City name in English, e.g. 'Yangon'")
    unit: str = Field(default="celsius", description="'celsius' or 'fahrenheit'")

@tool(args_schema=WeatherInput)
def get_weather(city: str, unit: str = "celsius") -> str:
    """Get the current weather for a city."""
    # Pretend we call a real weather API here
    return f"Weather in {city}: 30 degrees {unit}"

llm = ChatOpenAI(model="gpt-4o-mini")
# bind_tools publishes the Pydantic schema to the model as a JSON tool spec
llm_with_tools = llm.bind_tools([get_weather])

msg = llm_with_tools.invoke("What is the weather in Yangon?")
print(msg.tool_calls)  # tool name + structured arguments
```

**အဓိကအယူအဆ** — Pydantic schema ကို `Field(description=...)` ဖြင့်တိကျစွာဖော်ပြပြီး `bind_tools` နှင့် ချိတ်လိုက်သောအခါ မော်ဒယ်က tool argument များကို ကျွန်ုပ်တို့ သတ်မှတ်ထားသည့်ပုံစံအတိုင်း ထုတ်ပေးသည်။

## လေ့ကျင့်ခန်း ၂ — ToolNode ဖြင့် tool execution ချဲ့ခြင်း

```python
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import ToolNode, create_react_agent

@tool
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b

tools = [add, multiply]
llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)

# ToolNode routes every tool_call in the last AIMessage to the matching tool
tool_node = ToolNode(tools)

# create_react_agent wires LLM node + ToolNode into a loop automatically
agent = create_react_agent(llm, tools)
result = agent.invoke({"messages": [("user", "What is 3 + 4 times 2?")]})
for m in result["messages"]:
    print(m.__class__.__name__, "->", m.content)
```

**အဓိကအယူအဆ** — `ToolNode` ဆိုသည်မှာ AIMessage ထဲရှိ `tool_calls` အားလုံးကို သက်ဆိုင်ရာ Python function များသို့ ချိတ်ဆက်ပေးသည့် နှစ်သက်ဖွယ်ရာ node တစ်ခုဖြစ်ပြီး `create_react_agent` က LLM နှင့် ToolNode ကြား loop ကို အလိုအလျောက် တည်ဆောက်ပေးသည်။

## လေ့ကျင့်ခန်း ၃ — Parallel tool calls

```python
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent

@tool
def get_word_length(word: str) -> int:
    """Return the length of a word."""
    return len(word)

@tool
def count_vowels(word: str) -> int:
    """Count vowels in a word."""
    return sum(1 for c in word.lower() if c in "aeiou")

tools = [get_word_length, count_vowels]
llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)  # parallel calls enabled by default

agent = create_react_agent(llm, tools)
result = agent.invoke({"messages": [("user", "How long is 'hello' and how many vowels?")]})

last_ai = result["messages"][-2] if result["messages"][-1].tool_calls else result["messages"][-1]
print("Number of parallel tool calls:", len(last_ai.tool_calls))
for tc in last_ai.tool_calls:
    print(tc["name"], tc["args"])
```

**အဓိကအယူအဆ** — မော်ဒယ်အများစုက မေးခွန်းတစ်ခါတည်းဖြင့် လိုအပ်သမျှ tool များကို AIMessage တစ်ခုထဲတွင် တစ်ပြိုင်တည်း ခေါ်ဆိုနိုင်ပြီး `ToolNode` က ၎င်းတို့အားလုံးကို တစ်ဆက်တည်း အလုပ်လုပ်ပေးသည်။

## လေ့ကျင့်ခန်း ၄ — Tool error contracts နှင့် retries

```python
from langchain_core.tools import tool
from langchain_core.messages import ToolMessage
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import ToolNode, create_react_agent

@tool
def divide(a: float, b: float) -> str:
    """Divide a by b."""
    if b == 0:
        # Raise a clear error; ToolNode catches it and returns it as a ToolMessage
        raise ValueError("Cannot divide by zero. Use a non-zero number for b.")
    return str(a / b)

tools = [divide]
llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)

# handle_tool_errors=True (default) converts exceptions into tool results
# so the agent can read the error and retry with corrected arguments
tool_node = ToolNode(tools, handle_tool_errors=True)
agent = create_react_agent(llm, tools)

result = agent.invoke({"messages": [("user", "Compute 10 / 0, fix any issue.")]})
for m in result["messages"]:
    print(m.__class__.__name__, "->", m.content)
```

**အဓိကအယူအဆ** — tool အတွင်း မှားယွင်းမှုဖြစ်ပါက `ToolNode` က exception ကို ToolMessage အဖြစ် မော်ဒယ်ဆီပြန်ပို့သဖြင့် မော်ဒယ်က error စာကိုဖတ်ပြီး argument ပြင်၍ ထပ်မံကြိုးစားနိုင်သည်၊ runtime crash ဖြစ်မသွားပါ။

## လေ့ကျင့်ခန်း ၅ — Tool arguments ကို validation လုပ်ခြင်း

```python
from typing import Literal
from pydantic import BaseModel, Field, field_validator
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent

class UnitInput(BaseModel):
    value: float = Field(description="Numeric temperature value")
    unit: Literal["celsius", "fahrenheit"] = Field(description="Target unit")

    @field_validator("value")
    @classmethod
    def check_range(cls, v: float) -> float:
        # Reject physically impossible temperatures
        if v < -273.15:
            raise ValueError("Value is below absolute zero (-273.15).")
        return v

@tool(args_schema=UnitInput)
def convert_temperature(value: float, unit: str) -> str:
    """Convert the given value to the requested unit (assumed input is celsius)."""
    if unit == "fahrenheit":
        return f"{value} C = {value * 9 / 5 + 32} F"
    return f"{value} F = {(value - 32) * 5 / 9} C"

tools = [convert_temperature]
llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)
agent = create_react_agent(llm, tools)
result = agent.invoke({"messages": [("user", "Convert -500 celsius to fahrenheit.")]})
for m in result["messages"]:
    print(m.__class__.__name__, "->", m.content)
```

**အဓိကအယူအဆ** — Pydantic validator နှင့် `Literal` type တို့ဖြင့် argument များကို tool အလုပ်လုပ်ချင်း မတိုင်ခင် စစ်ဆေးနိုင်ပြီး မှားသည့် input ကို error contract မှတစ်ဆင့် မော်ဒယ်ထံ ပြန်ပေးပါသည်။

## လေ့ကျင့်ခန်း ၆ — Tool permissions ကိုကျဉ်းချဲ့ခြင်း

```python
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent

@tool
def read_report(name: str) -> str:
    """Read a report by name. Only .txt reports in /data are allowed."""
    import os
    base = os.path.abspath("data")
    path = os.path.abspath(os.path.join(base, name))
    # Prevent path traversal: the resolved path must stay inside /data
    if not path.startswith(base + os.sep) or not path.endswith(".txt"):
        raise PermissionError("Access denied: only .txt files inside /data.")
    if not os.path.exists(path):
        raise FileNotFoundError(f"Report '{name}' not found.")
    return open(path, encoding="utf-8").read()

tools = [read_report]
llm = ChatOpenAI(model="gpt-4o-mini").bind_tools(tools)
agent = create_react_agent(llm, tools)
result = agent.invoke({"messages": [("user", "Read ../secrets.txt")]})
for m in result["messages"]:
    print(m.__class__.__name__, "->", m.content)
```

**အဓိကအယူအဆ** — မော်ဒယ်ကို မြင်ရသည့် tool စာရင်းကို အနည်းဆုံးအထိ ကျဉ်းပြီး tool အတွင်းတွင်လည်း path စစ်ဆေးမှုကဲ့သို့သော permission check များထည့်သွင်းခြင်းဖြင့် agent ၏ လုပ်နိုင်စွမ်းကို ဘေးကင်းစွာ ကန့်သတ်ရမည်။
