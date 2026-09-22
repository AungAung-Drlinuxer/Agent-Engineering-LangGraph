# Agent Evaluation & Observability — ဖြေရှင်းနည်းများ

## လေ့ကျင့်ခန်း ၁ — Outcome နှင့် Trajectory အကဲဖြတ်ခြင်းကို ခွဲခြားနားလည်ခြင်း

Outcome evaluation ဆိုသည်မှာ agent ၏ နောက်ဆုံးအဖြေ (final answer) ကိုသာ မှန်/မှား စစ်ဆေးခြင်းဖြစ်ပြီး၊ trajectory evaluation က agent သည် ရောက်ရှိလာစဉ် ဖြတ်သန်းခဲ့သော အဆင့်များ (node calls, tool calls) ကို စစ်ဆေးခြင်းဖြစ်သည်။ အောက်တွင် နှစ်မျိုးစလုံးကို ရိုးရှင်းသော checker ဖြင့် ဖော်ပြထားသည်။

```python
# Compare outcome eval vs trajectory eval for a simple agent run

EXPECTED_ANSWER = "Paris"

def outcome_eval(final_answer: str) -> dict:
    # Outcome eval: only the final answer matters
    passed = final_answer.strip().lower() == EXPECTED_ANSWER.lower()
    return {"type": "outcome", "passed": passed}

def trajectory_eval(steps: list[str]) -> dict:
    # Trajectory eval: check the sequence of tool calls was reasonable
    used_search = any("search" in s for s in steps)
    no_loops = len(steps) == len(set(steps))
    passed = used_search and no_loops
    return {"type": "trajectory", "passed": passed}

# Simulate one agent run
final_answer = "Paris"
steps = ["plan", "search(capital_of_france)", "compose_answer"]

print(outcome_eval(final_answer))   # passes: answer is correct
print(trajectory_eval(steps))       # passes: used search, no repeated steps

# A run with the right answer but a poor trajectory
bad_steps = ["search(a)", "search(a)", "guess"]
print(outcome_eval("Paris"))        # still passes
print(trajectory_eval(bad_steps))  # fails: repeated the same step
```

**အဓိကအယူအဆ** — Outcome eval သည် အဖြေမှန်သည့်တိုင် ရှုံးထွေလမ်းကြောင်းဖြင့် ရောက်နိုင်သဖြင့်၊ ယုံကြည်ရသော agent တည်ဆောက်ရန် trajectory eval နှစ်မျိုးစလုံး လိုအပ်သည်။

## လေ့ကျင့်ခန်း ၂ — Agent evaluation dataset ဒီဇိုင်းချမှု

Evaluation dataset တစ်ခုတွင် task အရင်းအမြစ် (initial state)၊ ခွင့်ပြုထားသော tools၊ မျှော်မှန်းထားသော outcome နှင့် မျှော်မှန်းထားသော လမ်းကြောင်းတို့ ပါဝင်သင့်သည်။ အနည်းဆုံး input/output pair မျှသာ မဟုတ်ဘဲ agent ၏ အလုပ်လုပ်ပုံကို စစ်နိုင်ရန် ဒေတာဖွဲ့စည်းမှု ကောင်းရမည်။

```python
# Design a small agent evaluation dataset with expected trajectories
import json

DATASET = [
    {
        "example_id": "ex_001",
        "task": "What is the capital of France?",
        "initial_state": {"tools": ["search"]},
        "expected_outcome": "Paris",
        "expected_trajectory": ["search", "compose_answer"],
        "tags": ["geography", "single_hop"],
    },
    {
        "example_id": "ex_002",
        "task": "Compare the population of Tokyo and Delhi.",
        "initial_state": {"tools": ["search", "calculator"]},
        "expected_outcome": "Tokyo has more people",
        "expected_trajectory": ["search", "search", "calculator", "compose_answer"],
        "tags": ["comparison", "multi_hop"],
    },
    {
        "example_id": "ex_003",
        "task": "Summarize the latest LangGraph release notes.",
        "initial_state": {"tools": ["fetch_url", "summarize"]},
        "expected_outcome": "a short summary string",
        "expected_trajectory": ["fetch_url", "summarize"],
        "tags": ["retrieval", "tool_chain"],
    },
]

def validate_dataset(dataset: list[dict]) -> list[str]:
    # Check every example has the required fields before running evals
    required = ["example_id", "task", "initial_state", "expected_outcome"]
    errors = []
    for ex in dataset:
        for field in required:
            if field not in ex or not ex[field]:
                errors.append(f"{ex.get('example_id', '?')} missing {field}")
    return errors

print(validate_dataset(DATASET))  # [] means the dataset is well-formed
print(json.dumps(DATASET[0], indent=2))
```

**အဓိကအယူအဆ** — ကောင်းမွန်သော agent dataset ဆိုသည်မှာ မေးခွန်း–အဖြေသာမကဘဲ initial state၊ ခွင့်ပြု tools နှင့် မျှော်မှန်း trajectory တို့ပါဝင်သော စနစ်တကျ ဖွဲ့စည်းထားသော ဒေတာဖြစ်သည်။

## လေ့ကျင့်ခန်း ၃ — Langfuse ဖြင့် node/tool call အားလုံးကို trace လုပ်ခြင်း

Langfuse သည် OpenTelemetry standard အရ LLM application များကို tracing လုပ်ပေးသော open-source platform ဖြစ်သည်။ LangChain/LangGraph integration ဖြင့် graph တစ်ခုလုံး၏ node တိုင်း၊ tool call တိုင်းကို အလိုအလျောက် trace ရရှိသည်။

```python
# Trace every LangGraph node and tool call with Langfuse
import os

# pip install langfuse langchain langgraph
os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-lf-..."
os.environ["LANGFUSE_SECRET_KEY"] = "sk-lf-..."
os.environ["LANGFUSE_HOST"] = "https://cloud.langfuse.com"

from langfuse.langchain import CallbackHandler
from langgraph.graph import StateGraph, END
from typing_extensions import TypedDict

class State(TypedDict):
    question: str
    answer: str

def research_node(state: State) -> State:
    # In a real agent this node would call a search tool
    state["answer"] = "researched: " + state["question"]
    return state

def compose_node(state: State) -> State:
    # Final node that composes the answer
    state["answer"] = state["answer"] + " -> final"
    return state

builder = StateGraph(State)
builder.add_node("research", research_node)
builder.add_node("compose", compose_node)
builder.set_entry_point("research")
builder.add_edge("research", "compose")
builder.add_edge("compose", END)
graph = builder.compile()

# The handler records every node transition and LLM/tool call to Langfuse
handler = CallbackHandler()
result = graph.invoke(
    {"question": "What is LangGraph?"},
    config={"callbacks": [handler]},
)
print(result["answer"])
```

**အဓိကအယူအဆ** — Langfuse `CallbackHandler` ကို graph invoke တွင် ထည့်ပေးရုံဖြင့် node တိုင်းနှင့် tool call တိုင်း၏ latency၊ input/output နှင့် token ကုန်ကျမှုကို တစ်နေရာတည်းတွင် မြင်နိုင်သည်။

## လေ့ကျင့်ခန်း ၄ — CI တွင် regression gate ထည့်သွင်းခြင်း

Regression gate ဆိုသည်မှာ ကုဒ်အသစ်တင်သောအခါ agent ၏ အလုပ်လုပ်မှုအရည်အသွေး ယခင်အဆင့်ထက် မကျဆင်းစေရန် CI pipeline တွင် ထည့်သွင်းသော အလိုအလျောက် စစ်ဆေးမှုဖြစ်သည်။ Pass rate သတ်မှတ်ထားသော threshold အောက် ကျလျှင် pipeline ကို ရပ်တန့်စေသည်။

```python
# A regression gate that blocks CI when the eval pass rate drops
import sys

def run_eval_suite(dataset: list[dict], agent_fn) -> dict:
    # Run the agent on each example and collect pass/fail results
    results = []
    for ex in dataset:
        got = agent_fn(ex["task"])
        results.append(got.strip() == ex["expected_outcome"].strip())
    passed = sum(results)
    return {"total": len(results), "passed": passed,
            "pass_rate": passed / len(results)}

def regression_gate(metrics: dict, min_pass_rate: float = 0.8) -> bool:
    # Fail the gate if the pass rate is below the threshold
    ok = metrics["pass_rate"] >= min_pass_rate
    print(f"pass_rate={metrics['pass_rate']:.2f}, gate={'PASS' if ok else 'FAIL'}")
    return ok

# Tiny fake dataset and agent for demonstration
dataset = [{"task": "capital of France?", "expected_outcome": "Paris"}]

def fake_agent(task: str) -> str:
    # Stand-in for the real graph; returns a canned answer
    return "Paris"

metrics = run_eval_suite(dataset, fake_agent)
if not regression_gate(metrics, min_pass_rate=0.8):
    sys.exit(1)  # non-zero exit code makes the CI step fail
```

**အဓိကအယူအဆ** — Evaluation pass rate အတွက် သတ်မှတ်ထားသော threshold တစ်ခုကို CI တွင် gate အဖြစ် ထည့်သွင်းခြင်းဖြင့် အရည်အသွေးကျဆင်းမှုကို production မရောက်မီ ဖမ်းဆုပ်နိုင်သည်။

## လေ့ကျင့်ခန်း ၅ — Trajectory တစ်ခုချင်းစီအတွက် cost/latency တိုင်းတာခြင်း

Agent တစ်ခု၏ တကယ့်ကုန်ကျမှုကို ခန့်မှန်းရန် single call မဟုတ်ဘဲ တစ်ခုလုံးသော trajectory (node + tool calls အားလုံး) ၏ latency နှင့် token/cost ကို ပေါင်းစည်းတွက်ချက်ရမည်။

```python
# Measure cost and latency per trajectory by summing over all steps
import time

PRICES = {  # illustrative pricing keys; use your provider's current prices
    "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},  # USD per 1K tokens
}

def measure_trajectory(steps: list[dict], model: str = "gpt-4o-mini") -> dict:
    # steps: [{"name": "research", "input_tokens": 500, "output_tokens": 120}, ...]
    total_latency = 0.0
    total_cost = 0.0
    for step in steps:
        t0 = time.time()
        # (In a real run the node/tool would execute here)
        time.sleep(0.01)  # simulate the step taking some time
        total_latency += time.time() - t0
        rate = PRICES[model]
        total_cost += (
            step["input_tokens"] / 1000 * rate["input"]
            + step["output_tokens"] / 1000 * rate["output"]
        )
    return {
        "num_steps": len(steps),
        "latency_seconds": round(total_latency, 3),
        "cost_usd": round(total_cost, 6),
    }

trajectory = [
    {"name": "search", "input_tokens": 300, "output_tokens": 80},
    {"name": "reason", "input_tokens": 600, "output_tokens": 200},
    {"name": "compose", "input_tokens": 400, "output_tokens": 150},
]
print(measure_trajectory(trajectory))
```

**အဓိကအယူအဆ** — Cost နှင့် latency ကို ခေါ်ဆိုမှုတစ်ခုချင်း မကြည့်ဘဲ trajectory တစ်ခုလုံးရှိ step အားလုံးကို ပေါင်းခြင်းဖြင့် မှတ်ယူမှသာ agent ၏ တကယ့် စရိတ်ကျုံ့မှုနှင့် အလျင်ကို အတိအကျ သိနိုင်သည်။

## လေ့ကျင့်ခန်း ၆ — Failure taxonomy တည်ဆောက်ပြီး ခွဲခြားစဉ်းစားခြင်း

Failure taxonomy ဆိုသည်မှာ agent မေ့မြာပျက်စီးပုံကို အမျိုးအစားခွဲခြားထားသော စနစ်ဖြစ်ပြီး၊ မေးခွန်းမှားယွင်းမှု၊ 工具 ရွေးချယ်မှု မှားယွင်းခြင်း၊ context နားမလည်ခြင်း စသည်ဖြင့် ခွဲခြားခြင်းအားဖြင့် ပြင်ဆင်ရန် ဦးစားပေးနေရာကို ချိန်းဆိုနိုင်သည်။

```python
# A simple failure taxonomy for classifying agent errors
from collections import Counter

FAILURE_CATEGORIES = [
    "task_misinterpretation",   # agent misunderstood the user's goal
    "tool_selection_error",     # picked the wrong tool entirely
    "tool_execution_error",     # right tool, call failed (bad args, timeout)
    "reasoning_error",          # correct tools, wrong intermediate logic
    "final_answer_error",       # everything ran, but the answer is wrong
    "budget_exceeded",          # ran out of steps/tokens before finishing
]

def classify_failure(failure_description: str) -> str:
    # Map a short description to one category via keyword matching
    rules = {
        "tool_selection_error": ["wrong tool", "called search instead"],
        "tool_execution_error": ["timeout", "invalid arguments", "api error"],
        "reasoning_error": ["bad logic", "wrong step order"],
        "final_answer_error": ["answer wrong", "hallucinated"],
        "budget_exceeded": ["out of steps", "token limit"],
        "task_misinterpretation": ["misunderstood", "wrong goal"],
    }
    desc = failure_description.lower()
    for category, keywords in rules.items():
        if any(k in desc for k in keywords):
            return category
    return "task_misinterpretation"  # default bucket

failures = [
    "agent called search instead of calculator",
    "tool returned timeout after 30s",
    "final answer was wrong for ex_002",
    "ran out of steps before finishing",
]
counts = Counter(classify_failure(f) for f in failures)
print(dict(counts))
# Knowing the most frequent category tells you what to fix first
print("top priority:", counts.most_common(1)[0][0])
```

**အဓိကအယူအဆ** — ပျက်စီးမှုတိုင်းကို သတ်မှတ်စနစ်တကျ အမျိုးအစားခွဲခြင်းဖြင့် အဖြစ်များဆုံး failure အမျိုးအစားကို အ优先 ပြင်ဆင်နိုင်ပြီး တိုင်းတာမှုမရသော ပြဿနာကို စီမံခန့်ခွဲနိုင်သော အလုပ်အဖြစ် ပြောင်းလဲပေးသည်။
