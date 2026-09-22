# Tool-Calling Agents

LLM ကို external tools (function calling) နဲ့ ချိတ်ဆက်ပြီး LangGraph ဖြင့် practical agent တစ်ခုကို တည်ဆောက်နည်းကို ဒီ module မှာ လေ့လာပါမယ်။

## ဒီ module မှာ ဘာသင်မလဲ

- `bind_tools` ကို Pydantic model ဒါမှမဟုတ် JSON schema နဲ့ သုံးပြီး LLM ဆီက structured tool arguments ယူနည်း
- LangGraph ရဲ့ built-in `ToolNode` ကိုသုံးပြီး tool execution ကို graph state ထဲ ထည့်သွင်းနည်း
- Model က တစ်ကြိမ်ထက်ပိုတဲ့ tool call တွေကို parallel အနေနဲ့ ခေါ်ဆိုတဲ့အခါ ဆက်ဆံရေးနည်း
- Tool တွေက failed ရင် ဘယ်လို format နဲ့ error ပြန်ပြီး model အလုပ်ခိုင်းအောင် လုပ်နည်း
- Tool arguments တွေကို validation စစ်ဆေးပြီး permission တွေကို ကျဉ်းသွားစေတဲ့ pattern တွေ

## သင်ခန်းစာများ

1. **Tool ကို model နဲ့ bind လုပ်ခြင်း** — `bind_tools` သုံးပြီး Pydantic class တစ်ခုက function schema အဖြစ် ပြောင်းပေးပုံ။
2. **ToolNode နဲ့ graph ဆောက်ခြင်း** — model node၊ `ToolNode` နဲ့ conditional edge တွေနဲ့ agent loop တစ်ခု ဖွဲ့စည်းပုံ။
3. **Parallel tool calls** — တစ်ချိန်တည်းမှာ tool အများအပြား ခေါ်တဲ့ pattern နဲ့ state handling။
4. **Tool error contracts နဲ့ retries** — `ToolMessage` နဲ့ error ပြန်တာ၊ invalid JSON၊ API failure စတာတွေကို model က retry ခိုင်းနည်း။
5. **Argument validation** — Pydantic validation က tool call မလုပ်ခင် ကာကွယ်ပေးပုံ၊ `InvalidToolCall` handling။
6. **Permission narrowing** — Graph recursion ကို သတ်မှတ်ခြင်း၊ tool choice ကို ကန့်သတ်ခြင်းနဲ့ least-privilege design အခြေခံများ။

## လိုအပ်ချက်များ (Prerequisites)

- Python basics — function, class, decorator နဲ့ type hints
- Pydantic အခြေခံ (model သတ်မှတ်ခြင်း၊ validation error)
- LangGraph StateGraph၊ nodes၊ edges အခြေခံ (အထက် module တွေကနေ)
- OpenAI ဒါမှမဟုတ် Anthropic API key တစ်ခု (tool calling support ပါတဲ့ model)

## ဘယ်အချိန်မှာ အသုံးဝင်လဲ

- LLM က web search၊ calculator၊ database query စတဲ့ external functions တွေကို ခေါ်သင့်တဲ့အခါ
- Structured output လိုအပ်ပြီး model ရဲ့ raw text ကို ယုံကြည်လို့မရတဲ့ workflow တွေမှာ
- Real-world API error တွေနဲ့လည်း agent က ရပ်တန့်မသွားစေချင်တဲ့ production system တွေမှာ
- User data ဒါမှမဟုတ် internal service တွေနဲ့ ထိတွေ့တဲ့ tool တွေကို permission နဲ့ ထိန်းချုပ်ချင်တဲ့အခါ

## ကိုးကား

- How to create a ReAct agent: https://langchain-ai.github.io/langgraph/how-tos/create-react-agent/
- Passing input to tools: https://langchain-ai.github.io/langgraph/how-tos/passing-input-to-tools/
- Forcing tool calling: https://langchain-ai.github.io/langgraph/how-tos/force-calling-a-tool-first/
- Handling tool calling errors: https://langchain-ai.github.io/langgraph/how-tos/tool-calling-errors/
- ReAct agent from scratch: https://langchain-ai.github.io/langgraph/how-tos/react-agent-from-scratch/
