# AI Agents - Hello World

**See it work first. Understand it after.**

> **Python note:** The code below uses `import`, `def`, `while` loops, `json`, and `print`. If any of that is unfamiliar, the [Python for AI notebook](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_Python_for_AI.ipynb) covers everything you need. But try reading the code first -- you may understand more than you think.

---

## What You Are About to See

In the next 2 minutes of reading, you will see an agent that:

- Receives a question it cannot answer from memory alone
- Reasons about which tool to use
- Calls the tool and observes the result
- Decides if it needs another tool
- Gives a final answer that combines information from multiple tool calls

The agent runs entirely on your laptop using **Ollama** with **Mistral 7B** -- no API keys, no cloud, no cost.

The proof that this is an agent and not a script: **the agent DECIDES which tool to use based on the question.** Ask about math, it calls the calculator. Ask about weather, it calls the weather lookup. Nobody hardcoded that logic.

---

## Prerequisites

Install Ollama and pull the Mistral model:

```bash
# Install Ollama (macOS)
brew install ollama

# Start the Ollama server
ollama serve

# Pull the Mistral model (~4 GB download, one time)
ollama pull mistral
```

**You should see:** After `ollama pull mistral` completes, running `ollama list` should show `mistral` in the list.

---

## The Code -- All of It

```python
import json
import requests

# --- Define the tools the agent can use ---

def calculator(expression: str) -> str:
    """Evaluate a math expression and return the result."""
    try:
        result = eval(expression)  # Safe enough for demo; production uses a parser
        return str(result)
    except Exception as e:
        return f"Error: {e}"

def weather_lookup(city: str) -> str:
    """Look up current weather for a city (simulated)."""
    # Simulated responses -- in production, this calls a real weather API
    weather_data = {
        "seattle": "52°F, partly cloudy, wind 8 mph",
        "new york": "68°F, sunny, wind 12 mph",
        "london": "45°F, rainy, wind 15 mph",
    }
    return weather_data.get(city.lower(), f"No weather data available for {city}")

# --- Tool registry: tells the agent what tools exist ---
TOOLS = {
    "calculator": {
        "function": calculator,
        "description": "Evaluates a math expression. Input: a math expression as a string."
    },
    "weather_lookup": {
        "function": weather_lookup,
        "description": "Gets current weather for a city. Input: a city name."
    },
}

# --- The system prompt that teaches the LLM to be an agent ---
SYSTEM_PROMPT = """You are a helpful assistant with access to tools.

Available tools:
{tool_descriptions}

To use a tool, respond in EXACTLY this format:
Thought: <your reasoning about what to do>
Action: <tool name>
Action Input: <input to the tool>

When you have the final answer, respond in this format:
Thought: <your final reasoning>
Final Answer: <your answer to the user>

Always start with a Thought. Use tools when needed. Do not make up information."""

def build_system_prompt():
    """Build the system prompt with tool descriptions."""
    descriptions = ""
    for name, info in TOOLS.items():
        descriptions += f"- {name}: {info['description']}\n"
    return SYSTEM_PROMPT.format(tool_descriptions=descriptions)

# --- The agent loop ---
def run_agent(question: str, max_steps: int = 5) -> str:
    """Run a ReAct agent loop until it produces a Final Answer or hits max steps."""
    messages = [
        {"role": "system", "content": build_system_prompt()},
        {"role": "user", "content": question},
    ]

    for step in range(max_steps):
        # Call the LLM
        response = requests.post(
            "http://localhost:11434/api/chat",
            json={"model": "mistral", "messages": messages, "stream": False},
        )
        assistant_message = response.json()["message"]["content"]
        print(f"\n--- Step {step + 1} ---")
        print(assistant_message)

        # Check if the agent is done
        if "Final Answer:" in assistant_message:
            return assistant_message.split("Final Answer:")[-1].strip()

        # Parse and execute the tool call
        if "Action:" in assistant_message and "Action Input:" in assistant_message:
            action_line = assistant_message.split("Action:")[-1].split("\n")[0].strip()
            input_line = assistant_message.split("Action Input:")[-1].split("\n")[0].strip()

            if action_line in TOOLS:
                result = TOOLS[action_line]["function"](input_line)
                print(f"  -> Tool '{action_line}' returned: {result}")

                # Append the assistant's message and the observation
                messages.append({"role": "assistant", "content": assistant_message})
                messages.append({"role": "user", "content": f"Observation: {result}"})
            else:
                messages.append({"role": "assistant", "content": assistant_message})
                messages.append({"role": "user", "content": f"Observation: Tool '{action_line}' not found. Available tools: {list(TOOLS.keys())}"})
        else:
            # LLM did not follow the format -- nudge it
            messages.append({"role": "assistant", "content": assistant_message})
            messages.append({"role": "user", "content": "Please use the Thought/Action/Action Input format or provide a Final Answer."})

    return "Agent reached maximum steps without a final answer."


# --- Run it ---
if __name__ == "__main__":
    print("=" * 60)
    print("Question 1: A math question")
    print("=" * 60)
    answer = run_agent("What is 15% of 340, then add 42 to that result?")
    print(f"\nFINAL ANSWER: {answer}")

    print("\n" + "=" * 60)
    print("Question 2: A weather question")
    print("=" * 60)
    answer = run_agent("What is the current weather in Seattle?")
    print(f"\nFINAL ANSWER: {answer}")

    print("\n" + "=" * 60)
    print("Question 3: A multi-tool question")
    print("=" * 60)
    answer = run_agent("What is the temperature in London in Celsius? Convert from Fahrenheit.")
    print(f"\nFINAL ANSWER: {answer}")
```

---

## What You Should See

### Question 1: Math

```
--- Step 1 ---
Thought: The user wants me to calculate 15% of 340 and then add 42.
I should use the calculator tool.
Action: calculator
Action Input: 340 * 0.15 + 42
  -> Tool 'calculator' returned: 93.0

--- Step 2 ---
Thought: The calculator returned 93.0. That is the final answer.
Final Answer: 15% of 340 is 51, plus 42 equals 93.0.

FINAL ANSWER: 15% of 340 is 51, plus 42 equals 93.0.
```

### Question 2: Weather

```
--- Step 1 ---
Thought: The user is asking about weather. I should use the weather_lookup tool.
Action: weather_lookup
Action Input: Seattle
  -> Tool 'weather_lookup' returned: 52°F, partly cloudy, wind 8 mph

--- Step 2 ---
Thought: I now have the weather information for Seattle.
Final Answer: The current weather in Seattle is 52°F, partly cloudy, with wind at 8 mph.

FINAL ANSWER: The current weather in Seattle is 52°F, partly cloudy, with wind at 8 mph.
```

### Question 3: Multi-tool (weather + calculator)

```
--- Step 1 ---
Thought: The user wants the temperature in London in Celsius. First I need
to get the temperature in Fahrenheit, then convert it.
Action: weather_lookup
Action Input: London
  -> Tool 'weather_lookup' returned: 45°F, rainy, wind 15 mph

--- Step 2 ---
Thought: London is 45°F. I need to convert to Celsius using (F - 32) * 5/9.
Action: calculator
Action Input: (45 - 32) * 5 / 9
  -> Tool 'calculator' returned: 7.222222222222222

--- Step 3 ---
Thought: I now have the temperature in Celsius.
Final Answer: The temperature in London is approximately 7.2°C (45°F). It is currently rainy with wind at 15 mph.

FINAL ANSWER: The temperature in London is approximately 7.2°C (45°F)...
```

**The proof:** For Question 3, the agent decided on its own to call TWO tools in sequence -- weather first, then calculator. Nobody told it to do this. The LLM reasoned that it needed both pieces of information and orchestrated the calls itself.

---

## What Just Happened -- Line by Line

### The Tools

```python
def calculator(expression: str) -> str:
    result = eval(expression)
    return str(result)
```

Each tool is a plain Python function. It takes a string input and returns a string output. The agent framework calls these functions -- the LLM never executes code directly.

### The Tool Registry

```python
TOOLS = {
    "calculator": {
        "function": calculator,
        "description": "Evaluates a math expression. Input: a math expression as a string."
    },
}
```

This is how the agent knows what tools exist. The descriptions are included in the system prompt. If you add a new tool here, the agent can immediately use it -- no code changes to the agent loop.

### The System Prompt

The system prompt teaches the LLM the ReAct format: Thought, Action, Action Input, and Final Answer. Without this prompt, the LLM would just try to answer directly from its training data. The prompt transforms a chatbot into an agent.

### The Agent Loop

```python
for step in range(max_steps):
    # 1. Call the LLM
    # 2. Check if it said "Final Answer" → done
    # 3. Check if it said "Action" → call the tool, append the result
    # 4. Repeat
```

This is the entire agent pattern. Everything else -- LangChain, LangGraph, Autogen -- is a more sophisticated version of this same loop.

---

## You Should See Checkpoints

| After This Step | You Should See |
|---|---|
| `ollama serve` | Ollama running on http://localhost:11434 |
| `ollama pull mistral` | Download completes, model appears in `ollama list` |
| Run the script | Step-by-step output with Thought/Action/Observation |
| Question 1 (math) | Agent calls `calculator`, returns correct math result |
| Question 2 (weather) | Agent calls `weather_lookup`, returns weather data |
| Question 3 (multi-tool) | Agent calls BOTH tools in sequence without being told to |

---

## Common First-Time Issues

| Problem | Cause | Fix |
|---|---|---|
| `Connection refused` on localhost:11434 | Ollama server not running | Run `ollama serve` in a separate terminal |
| `Model not found: mistral` | Model not pulled yet | Run `ollama pull mistral` |
| Agent does not follow Thought/Action format | Smaller models sometimes ignore format instructions | Try the prompt again, or use a larger model (`ollama pull llama3`) |
| Agent loops without reaching Final Answer | Model keeps calling tools instead of concluding | Reduce `max_steps` or add "You MUST give a Final Answer within 3 steps" to the system prompt |
| `eval()` security warning | Python `eval` runs arbitrary code | For demos only. Production agents use a safe math parser like `numexpr` or `sympy` |
| Agent hallucinates a tool name | LLM invents a tool that does not exist | The code handles this -- it returns "Tool not found" and lists available tools |
| Very slow responses | Mistral 7B on CPU is slow | Use a machine with a GPU, or reduce model size with `ollama pull mistral:7b-instruct-q4_0` |

---

## What Comes Next

You have seen it work. Now the question is: how does it REALLY work under the hood, and how do you make it work in production?

| Question | Where You Find the Answer |
|---|---|
| How does the LLM know to output "Action:" instead of just answering? | [04 - How It Works](04_How_It_Works.md) -- prompt engineering and output parsing |
| Should I use Mistral, GPT-4, or Claude for my agent? | [05 - Building It](05_Building_It.md) -- every tradeoff |
| How do I build a multi-agent system where agents collaborate? | The notebook -- Section 3 builds multi-agent with LangGraph |
| How do production agent systems handle failures? | [07 - System Design](07_System_Design.md) -- fault tolerance and human-in-the-loop |
| What if someone tricks my agent into calling dangerous tools? | [08 - Quality, Security, Governance](08_Quality_Security_Governance.md) -- prompt injection and sandboxing |

**Next:** [04 - How It Works](04_How_It_Works.md) -- The mechanics behind what you just saw. How the LLM generates structured tool calls, how parsing works, and why agents fail.

---

## Quick Links

| Chapter | Topic |
|---|---|
| [01 - Why](01_Why.md) | Why agents matter |
| [02 - Concepts](02_Concepts.md) | Agent architecture, ReAct, tools, memory |
| [03 - Hello World](03_Hello_World.md) | This page |
| [04 - How It Works](04_How_It_Works.md) | The ReAct loop in detail |
| [05 - Building It](05_Building_It.md) | LLM, framework, architecture tradeoffs |
| [06 - Production Patterns](06_Production_Patterns.md) | How production agent systems work |
| [07 - System Design](07_System_Design.md) | Scaling, state, fault tolerance |
| [08 - Quality, Security, Governance](08_Quality_Security_Governance.md) | Prompt injection, tool misuse, sandboxing |
| [09 - Observability & Troubleshooting](09_Observability_Troubleshooting.md) | Tracing, cost monitoring, debugging |
| [10 - Decision Guide](10_Decision_Guide.md) | Decision table and production readiness |

**Hands-on notebook:** [Agents on Colab](https://colab.research.google.com/github/sunilmogadati/systems-in-production/blob/main/implementation/notebooks/AI_Engineer_Accelerator_Agents.ipynb) -- the full implementation with from-scratch agents, LangChain agents, and LangGraph multi-agent systems.

**Production architecture:** [Production Diagnostics Architecture](../../systems/production-diagnostics/architecture.md)
