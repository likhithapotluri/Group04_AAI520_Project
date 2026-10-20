# 📈 Autonomous Multi-Agent Investment Research & Financial Analysis System

This project is a part of the **AAI-520** course in the Applied Artificial Intelligence Program at the University of San Diego (USD).

### Project Status: Completed

---

## Installation

The project is a single Jupyter notebook designed to run in **Google Colab**. API keys are optional: without them, the system falls back to a local open-source model or a rule-based engine.

### Option 1: Google Colab (recommended)

1. Clone the repository:
   ```bash
   git init
   git clone https://github.com/likhithapotluri/Group04_AAI520_Project.git
   ```
2. Open `AAI_520_Final_Project_Group_04.ipynb` in Google Colab.
3. *(Optional)* Select **Runtime → Change runtime type → T4 GPU** to enable the local Qwen model.
4. *(Optional)* Add any of these keys in **Colab Secrets** (🔑 icon in the left sidebar):

   | Secret | Purpose | Required? |
   |---|---|---|
   | `GEMINI_API_KEY` | Google Gemini reasoning backend | Optional |
   | `NEWS_API_KEY` | NewsAPI headlines (falls back to Yahoo Finance news) | Optional |
   | `FRED_API_KEY` | FRED via `fredapi` (falls back to the public FRED CSV) | Optional |
   | `SEC_USER_AGENT` | Name + email for SEC EDGAR requests (not a key) | Optional |

5. Run **Runtime → Run all**. When prompted, allow Google Drive access so the agent's memory (`agent_memory.db`) persists across sessions. If you decline, memory is stored locally for that session only.
6. To analyze a different stock, change `TICKER = "NVDA"` in the **Run the System** cell to any US-listed ticker.

### Option 2: Local machine

```bash
git clone https://github.com/likhithapotluri/Group04_AAI520_Project.git
cd Group04_AAI520_Project
pip install yfinance google-genai pandas numpy matplotlib requests beautifulsoup4 lxml fredapi tenacity
# Optional, for the local LLM backend:
pip install torch transformers
```

Set keys as environment variables (e.g. `export GEMINI_API_KEY=...`). Before running, change `db_path` in the `Config` class to a local path, and skip the Google Drive mount cell, since it only works in Colab.

### Backend selection (automatic)

| Backend | Used when | Reasoning engine |
|---|---|---|
| `gemini` | A Gemini key is present | Google Gemini |
| `local` | No key, GPU available | Qwen2.5-1.5B-Instruct (Apache-2.0, runs in Colab) |
| `offline` | No key, no GPU | Rule-based engine |

You can force a backend by setting `LLM_BACKEND = "gemini" | "local" | "offline"` in the configuration cell.

---

## Project Intro/Objective

The main purpose of this project is to build an **autonomous multi-agent system that researches any US-listed stock from start to finish and produces a grounded BUY / HOLD / SELL recommendation with a full research note**. Specialized AI agents plan their own research steps, choose and call data tools at runtime, critique and revise their own drafts before showing them, and remember earlier runs to focus on what changed since the last call.

Traditional equity research is time-consuming and depends on combining market data, financial statements, regulatory filings, macroeconomic indicators, and news. This project shows how agentic AI workflows can automate that process while staying **grounded in real data**: the system checks every report for numbers that can't be traced back to the evidence. It also degrades gracefully, so it keeps working without any API keys.

> ⚠️ **Disclaimer:** This is an academic project and is **not financial advice**.

---

## Partner(s)/Contributor(s)

- Harsh Modi
- Jaiganesh Nagabooshanam
- Likhitha Potluri

---

## Methods Used

- Agentic AI / Multi-Agent Systems
- Large Language Models (LLMs) and Prompt Engineering
- Natural Language Processing (sentiment analysis, entity and event extraction)
- Agentic Workflow Patterns: Prompt Chaining, Routing, Evaluator–Optimizer
- Self-Reflection and Hallucination Detection (numeric grounding checks)
- Financial Technical Analysis (SMA, RSI, volatility, drawdown, momentum)
- Fundamental Analysis (valuation multiples, earnings, EPS surprises)
- Persistent Memory and Caching
- Data Manipulation
- Data Visualization

---

## Technologies

- Python
- Jupyter Notebook / Google Colab
- Google Gemini API (`google-genai`)
- Hugging Face Transformers + PyTorch (Qwen2.5-1.5B-Instruct)
- SQLite
- pandas, NumPy
- Matplotlib
- yfinance, fredapi, Requests
- BeautifulSoup4, lxml

---

## Project Description

### Overview

Instead of a fixed dataset, the system pulls **live data from free public sources** each time it runs:

| Source | Data | Key needed? |
|---|---|---|
| Yahoo Finance (`yfinance`) | 2 years of daily OHLCV prices, 20 fundamentals (P/E, EPS, beta, market cap, debt-to-equity, dividend yield, analyst target, etc.), quarterly income statements, EPS surprise history, news | No |
| FRED | 10-Year Treasury yield, CPI and YoY inflation, Fed Funds rate, unemployment rate | No (public CSV) |
| NewsAPI | Recent financial headlines and descriptions | Optional |
| SEC EDGAR | Latest 10-K / 10-Q filing; the Management's Discussion & Analysis (MD&A) section is extracted automatically | No key, just a name + email as User-Agent |

### System Architecture

```
User ticker ─► 🧠 Recall Memory ─► 📋 Planner ─► 🧭 Supervisor Router
                                                     ├─► 📊 Market Analyzer Agent
                                                     ├─► 📑 Earnings/Filings Agent
                                                     └─► 📰 News Agent (Prompt Chain)
          ─► ✍️ Report Writer ⇄ 🔍 Evaluator + 🪞 Self-Reflection ─► ✅ Recommendation
          ─► 💾 Save to Memory
```

**Core components:**

- **Research Planner:** breaks the goal into 4–7 concrete research steps, using memory of earlier runs to focus on what has changed.
- **Tool Registry:** agents look up and call tools by name at runtime. Results are cached (12-hour TTL), failures are isolated, and every call is logged.
- **Specialist agents:**
  - 📊 **Market Analyzer:** price trend, technical indicators, valuation, and macro context
  - 📑 **Earnings/Filings:** quarterly results, EPS surprises, and SEC MD&A tone
  - 📰 **News:** sentiment, entities, events, catalysts, and risks
- **Persistent memory (SQLite):** stores run history, learned insights, a per-run scratchpad (audit trail), and the tool-result cache.

### The Three Agentic Workflow Patterns

| # | Pattern | Implementation |
|---|---|---|
| 1 | **Prompt Chaining** | The News agent runs a 4-step chain: Preprocess → Sentiment → Entity & Event Extraction → Summary. Each step has validation checks (empty-input handling, guaranteed labels for every article, score normalization). Aggregate sentiment is relevance-weighted and calculated in Python. |
| 2 | **Routing** | A supervisor router uses the LLM to send each research task to one specialist and choose its tools. The choice is validated (unknown agents rejected, out-of-scope tools removed), with keyword-based routing as a fallback. A coverage check makes sure every specialist runs at least once. |
| 3 | **Evaluator–Optimizer** | A writer drafts a note with 8 required sections. An evaluator scores it on factual grounding, absence of hallucinations, coherence, completeness, and risk balance, while a self-reflection module checks numeric grounding and consistency with a composite quant signal. The writer revises for up to 3 rounds. |

**Pass criteria:**
```
quality = 0.60 × evaluator score + 0.25 × numeric grounding + 0.15 × signal consistency
PASS if quality ≥ 0.80 AND confidence ≥ 0.65 AND no critical issues AND no missing sections
```
If no LLM draft passes, a rule-based report is also scored and the better of the two is kept. Anything still below the bar is flagged `LOW_CONFIDENCE_REVIEW` for human review.

### Graceful Degradation

Every LLM call carries a **rule-based fallback**. If the model errors, times out, returns invalid JSON, or leaves out required keys, only that step falls back to rules and the run continues. The system tracks how many steps were produced by the LLM and how many by rules, so every output can be traced to its source.

### Questions Explored

- Can a team of specialized agents produce investment research that is **grounded in real data** rather than hallucinated?
- Do agentic workflow patterns (chaining, routing, evaluator–optimizer) measurably improve output quality and reliability?
- Can persistent memory help an agent focus on **what changed** between runs instead of starting over each time?

### Results (Sample Runs)

| Ticker | Recommendation | Confidence | Status | Quality | Numeric Grounding | Rule-based Fallbacks |
|---|---|---|---|---|---|---|
| NVDA | **BUY** | 0.85 | APPROVED | 0.993 | 0.972 | 0 |
| AAPL | **HOLD** | 0.65 | APPROVED | 1.000 | 1.000 | 0 |

- **Routing:** every routing decision in both runs was made by the LLM (`method: llm`), not the keyword fallback.
- **Evaluator–Optimizer:** both reports passed on the first iteration with no unsupported claims flagged.
- **Prompt Chaining:** the AAPL run fetched and analyzed 25 live news articles end to end.
- **Memory:** recalled history and insights grew across runs, and the planner's tasks shifted to comparing results against the previous call.

The notebook's dashboard plots the price trend with SMA-50/200, quality and confidence per evaluator round, and per-article news sentiment.

### Challenges & Roadblocks

- **Model availability:** some Gemini model names still appear in the model catalog but fail on generation. `resolve_model()` was changed to test each candidate with a live call before using it.
- **Stale cache:** an empty news result cached before the NewsAPI key was added kept being replayed, so `NewsTool` was never actually called. Clearing the cache fixed it. A permanent fix would be to skip caching any result that contains an internal error field.
- **Unpredictable LLM output:** models sometimes return lists instead of objects, numbers as strings, or JSON wrapped in markdown. Defensive parsing helpers (`extract_json`, `safe_float`, `ensure_dict`) handle these cases.

### Limitations

- The numeric grounding check is a heuristic, and evaluators can miss subtle errors.
- Offline mode is a deterministic baseline, not true reasoning.
- SEC EDGAR covers US-listed companies only.
- Tracking whether past calls were right needs many runs over time before it becomes meaningful.

### Future Work

- Replace the offline word-list sentiment with a financial-domain model such as FinBERT
- Backtest stored recommendations over longer periods
- Add vector memory for semantic search over filings and news
- Add more specialists: ESG, insider trading, peer comparison
- Use a different model as the evaluator to reduce self-evaluation bias

---

## Acknowledgments

We thank our AAI-520 instructor, **Professor Rajesh Soni**, and the University of San Diego Applied Artificial Intelligence program for their guidance throughout this project. We also acknowledge the providers of the free data sources used here (Yahoo Finance, FRED, SEC EDGAR, NewsAPI), the Qwen team for the open-source Qwen2.5 model, and the [Multi_Agent_Financial_Analysis_System](https://github.com/marwahfaraj/Multi_Agent_Financial_Analysis_System) project, which inspired parts of the design.
