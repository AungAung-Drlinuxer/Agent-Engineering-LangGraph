# Multi-Agent Patterns: Supervisor, Swarm, Subgraphs

Agent တစ်ခုတည်းနဲ့ မလုံလောက်တဲ့အခါ LangGraph နဲ့ multi-agent system ဆောက်နည်းများကို လေ့လာရန် module ဖြစ်သည်။

## ဒီ module မှာ ဘာသင်မလဲ

- Agent တစ်ခုတည်းရဲ့ ကန့်သတ်ချက်နဲ့ ဘယ်အချိန်မှာ multi-agent လိုအပ်လဲ
- Supervisor pattern — အုပ်ချုပ်သူ agent က အခြား agent များကို လမ်းညွှန်ပေးပုံ
- Swarm / handoff pattern — agent တွေက တစ်ယောက်နဲ့တစ်ယောက် တာဝန်လွှဲပြောင်းပုံ
- Parallel fan-out / fan-in — အလုပ်တွေကို တစ်ပြိုင်တည်း ခွဲထုတ်ပြီး ပြန်ပေါင်းပုံ
- Subgraph — ပြန်သုံးနိုင်တဲ့ agent component များနဲ့ state isolation
- Extra hop တွေရဲ့ ကုန်ကျစရိတ် (latency, token cost)

## သင်ခန်းစာများ

1. ဘာကြောင့် agent တစ်ခု မလုံလောက်တာလဲ
2. Supervisor routing pattern
3. Swarm / handoff pattern
4. Parallel fan-out နှင့် fan-in
5. Subgraph များနှင့် state isolation / shared context
6. Extra hop ကုန်ကျစရိတ် သုံးသပ်ခြင်း

## လိုအပ်ချက်များ (Prerequisites)

- Python basics (function, dict, class)
- LangGraph basics: StateGraph, nodes, edges, conditional edges
- LangChain ရဲ့ message များနဲ့ LLM invocation အခြေခံ
- `pip install langgraph langchain` ပြီးသား environment

## ဘယ်အချိန်မှာ အသုံးဝင်လဲ

- စနစ်ကြီးများမှာ tool များစွာ သုံးရပြီး context window ကျဉ်းလာတဲ့အခါ
- Agent တစ်ခုချင်းစီက ကျွမ်းကျင်မှု ကွဲပြားရမဲ့ အလုပ်မျိုး (research, coding, writing)
- တစ်ပြိုင်တည်း အလုပ်လုပ်ပြီး ရလဒ်ပေါင်းရမဲ့ workflow များ
- Agent component တွေကို နေရာများများမှာ ပြန်သုံးချင်တဲ့အခါ

## ကိုးကား

- LangGraph Multi-Agent Concepts: https://langchain-ai.github.io/langgraph/concepts/multi_agent/
- LangGraph Low-Level Concepts: https://langchain-ai.github.io/langgraph/concepts/low_level/
- LangGraph API Reference: https://langchain-ai.github.io/langgraph/reference/
