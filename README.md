# 🤖 Human-in-the-Loop (HITL) Stock Chatbot with LangGraph

A CLI chatbot built with **LangGraph** and a local **Ollama** LLM (`qwen2.5`) that can look up stock prices and "buy" stocks, but **never places an order without your approval**.

It demonstrates the **Action Approval Pattern** of Human-in-the-Loop (HITL): the graph pauses right before a critical action, asks a human to approve or reject, then resumes.

---

## 📖 What is HITL?

**Human-in-the-Loop (HITL)** is a design approach where a human actively participates at critical points of an AI workflow, to **supervise, approve, correct, or guide** the model's output.

Think of it as a human **checkpoint** inside an AI pipeline, so important decisions are not made autonomously by the model.

**Why it matters**

- Helps agentic systems stay under control
- Adds accountability
- Improves accuracy, safety, ethical alignment, and user experience

**Common HITL patterns**

| Pattern                 | Description                                        |
| ----------------------- | -------------------------------------------------- |
| Action Approval         | Approve / reject before execution ← _this project_ |
| Output Review / Edit    | Human edits the AI's output before it's used       |
| Ambiguity Clarification | AI asks the human when the request is unclear      |
| Escalation              | AI hands off to a human when it can't proceed      |

---

## ✨ Features

- 💬 Conversational chatbot with memory (per `thread_id`)
- 📈 `get_stock_price`: fetches the latest quote from Alpha Vantage
- 🛒 `purchase_stock`: simulated purchase, **gated by human approval**
- ⏸️ Uses LangGraph's `interrupt()` and `Command(resume=...)` to pause and resume the graph
- 🔒 Runs the LLM locally via Ollama (no OpenAI key needed)

---

## 🧠 How it works

```
          ┌───────┐
  START ─▶│ chat  │──── no tool call ───▶ END
          │ node  │
          └───┬───┘
              │ tool call
              ▼
          ┌───────┐
          │ tools │── purchase_stock ──▶ interrupt() ⏸ ── human says yes/no ──▶ resume
          └───┬───┘
              │
              └──────▶ back to chat node
```

1. You send a message. The LLM decides whether to call a tool.
2. For `get_stock_price`, the tool runs immediately.
3. For `purchase_stock`, the tool calls `interrupt(...)`. The graph **pauses** and its state is saved by the checkpointer (`MemorySaver`).
4. The CLI shows the approval prompt. You type `yes` or anything else.
5. The CLI resumes the graph with `Command(resume=decision)`. Only an exact `yes` places the order; anything else cancels it.

> ⚠️ A checkpointer is **required** for `interrupt()` to work, because the graph needs somewhere to save its state while waiting for you.

---

## 🛠️ Prerequisites

- **Python 3.10+**
- **[Ollama](https://ollama.com/download)** installed and running
- A free **[Alpha Vantage API key](https://www.alphavantage.co/support/#api-key)**

---

## 🚀 Setup

### 1. Clone the repo and create a virtual environment

```bash
git clone <your-repo-url>
cd <your-repo-folder>

python -m venv venv
source venv/bin/activate        # macOS / Linux
venv\Scripts\activate           # Windows
```

### 2. Install dependencies

```bash
pip install langgraph langchain-core langchain-ollama requests
```

Or, with a `requirements.txt`:

```text
langgraph
langchain-core
langchain-ollama
requests
```

```bash
pip install -r requirements.txt
```

### 3. Pull the model with Ollama

```bash
ollama pull qwen2.5
```

Make sure Ollama is running (`ollama serve`, or just open the Ollama app).

### 4. Set your Alpha Vantage API key

Never hardcode API keys in source code. Store it as an environment variable:

```bash
export ALPHAVANTAGE_API_KEY="your_key_here"      # macOS / Linux
setx ALPHAVANTAGE_API_KEY "your_key_here"        # Windows (restart terminal after)
```

Then read it in `chat_bot_hitl.py`:

```python
import os

API_KEY = os.getenv("ALPHAVANTAGE_API_KEY")

url = (
    "https://www.alphavantage.co/query"
    f"?function=GLOBAL_QUOTE&symbol={symbol}&apikey={API_KEY}"
)
```

### 5. Run the chatbot

```bash
python chat_bot_hitl.py
```

---

## 💡 Example session

```text
You: What is the price of AAPL?
Bot: The latest price of AAPL is $xxx.xx ...

You: Buy 10 shares of TSLA
HITL: Approve buying 10 shares of TSLA? (yes/no)
Your decision: yes
Bot: Purchase order placed for 10 shares of TSLA.

You: Buy 5 shares of MSFT
HITL: Approve buying 5 shares of MSFT? (yes/no)
Your decision: no
Bot: Purchase of 5 shares of MSFT was declined by human.

You: exit
Goodbye!
```

Type `exit` or `quit` to stop.

---

## 📁 Project structure

```text
.
├── chat_bot_hitl.py    # LLM, tools, graph, and CLI loop
└── README.md
```

---

## 🧩 Key code concepts

| Concept               | Where                               | Purpose                                                    |
| --------------------- | ----------------------------------- | ---------------------------------------------------------- |
| `@tool`               | `get_stock_price`, `purchase_stock` | Exposes Python functions to the LLM                        |
| `interrupt()`         | inside `purchase_stock`             | Pauses the graph and asks the human                        |
| `Command(resume=...)` | CLI loop                            | Resumes the paused graph with the human's answer           |
| `MemorySaver`         | graph compile                       | Saves state so the graph can pause and resume              |
| `thread_id`           | `config`                            | Identifies the conversation; reuse it to resume            |
| `tools_condition`     | conditional edge                    | Routes to the tools node only when the LLM requests a tool |

---

## 🔧 Troubleshooting

| Problem                                           | Fix                                                                             |
| ------------------------------------------------- | ------------------------------------------------------------------------------- |
| `Connection refused` to Ollama                    | Start Ollama (`ollama serve`)                                                   |
| `model 'qwen2.5' not found`                       | Run `ollama pull qwen2.5`                                                       |
| Alpha Vantage returns `{}` or a "rate limit" note | Free tier allows ~25 requests/day; wait or use a new key                        |
| Model doesn't call tools                          | Use a tool-calling-capable model (e.g. `qwen2.5`, `llama3.1`)                   |
| `__interrupt__` never appears                     | Make sure the graph is compiled with a checkpointer and a `thread_id` is passed |

---

## 🔐 Security notes

- Keep API keys in environment variables or a `.env` file, and add `.env` to `.gitignore`.
- `purchase_stock` is a **simulation**. If you connect it to a real brokerage, keep the human approval step.
- Treat any key that was ever committed to a repo as compromised and rotate it.

---

## 🌱 Ideas to extend

- Let the human **edit** the quantity instead of just approving or rejecting (Output Review / Edit pattern)
- Ask for clarification when the symbol is ambiguous (Ambiguity Clarification pattern)
- Escalate large orders to a manager (Escalation pattern)
- Swap `MemorySaver` for a persistent checkpointer (SQLite / Postgres)
- Build a web UI with Streamlit or FastAPI

---

## 📚 Resources

- [LangGraph docs](https://langchain-ai.github.io/langgraph/)
- [LangGraph human-in-the-loop guide](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/)
- [Ollama](https://ollama.com)
- [Alpha Vantage API](https://www.alphavantage.co/documentation/)

---

## 📄 License

MIT. Feel free to use and adapt.
