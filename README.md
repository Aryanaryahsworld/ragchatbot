# RAG Messaging Assistant

A context-aware messaging bot that decides **what to send, on which channel, and when** — without a single hardcoded `if/else` rule. The agent learns the patterns straight from the data using Retrieval-Augmented Generation (RAG) over a local vector store, with a deterministic compliance layer on top to keep things safe.

Built end-to-end in Python. Works with a local LLM (Ollama) out of the box and switches to OpenAI or Gemini with a single environment variable.

---

## The Problem

You get a JSONL file. Each line has a user profile (consent flags, channel preferences, lifecycle stage, timezone, locale) and the expected output for that case. The agent has to read the input, figure out the right action, and produce output that matches.

The interesting constraint: **no hardcoded rules**. You can't write `if sms_opt_in: use_sms`. The agent must infer the policy from examples.

## The Approach

```
   JSONL records                 New input
        |                            |
        v                            v
   [ ChromaDB ]  <--- retrieve --- [ embed ]
        |                            |
        |   top-k similar cases      |
        +--------+-------------------+
                 v
          [  LLM (RAG) ]   <-- few-shot examples
                 |
                 v
        AgentOutput (channel, send_at, body, CTA, next_action)
                 |
                 v
        [ Compliance Guard ]   <-- fair housing + opt-out (deterministic)
                 |
                 v
            Final message
```

1. Every record (input + expected output) gets embedded and stored in ChromaDB.
2. For a new input, the closest past cases are retrieved (leave-one-out so it can't cheat at eval time).
3. Those cases are dropped into the LLM prompt as few-shot examples.
4. The LLM picks the channel, timing, body, CTA, and next action — purely from the patterns it sees in the examples.
5. A deterministic compliance guard scans for fair-housing-protected terms and missing opt-outs. This stage is the only non-probabilistic one, because legal compliance can't depend on model output.

## Why this design

- **RAG over prompting**: the agent gets better automatically as more cases are added. Two records or two thousand, same code.
- **Local vector DB (ChromaDB)**: no API key, no network cost, demo runs anywhere. Swapping in Pinecone is a one-file change.
- **Compliance is deterministic**: fair housing (Title VIII) is a legal liability boundary. A hallucination can never reach a real user.
- **Backend-agnostic LLM**: Ollama for local/private work, OpenAI or Gemini for production-grade quality. Switching is one env variable.

## Results

Tested across 12 scenarios covering SMS, email, voice fallback, consent blocking, Spanish locale, renewal flows, no-show re-engagement, intent branching, and explicit no-send cases.

| Metric              | Score          |
|---------------------|----------------|
| Channel accuracy    | 12 / 12 (100%) |
| Compliance pass     | 12 / 12        |
| Avg semantic score  | 0.93           |
| Avg latency         | ~700 ms / case |

## Tech Stack

| Layer            | Choice                              |
|------------------|-------------------------------------|
| LLM              | Ollama (llama3.2) / OpenAI / Gemini |
| Vector DB        | ChromaDB (local, persistent)        |
| Embeddings       | all-MiniLM-L6-v2 (Chroma default)   |
| Validation       | Pydantic v2                         |
| Terminal UI      | Rich                                |
| Language         | Python 3.10+                        |

## Project Structure

```
rag-message-assistant/
├── agent/
│   ├── models.py           Pydantic schemas (TestCase, AgentOutput, EvalResult)
│   ├── llm.py              Backend wrapper (Ollama / OpenAI / Gemini auto-detect)
│   ├── knowledge_store.py  ChromaDB ingest + retrieve_similar (leave-one-out)
│   ├── stages.py           infer_and_compose + compliance_guard
│   ├── evaluator.py        Composite scoring across 6 dimensions
│   ├── pipeline.py         Orchestration
│   └── cli.py              Rich-based CLI runner
├── sample.jsonl            12 test cases across prospect + resident lifecycles
├── pyproject.toml
└── README.md
```

## Quick Start

### Mac / Linux

```bash
cd rag-message-assistant
python3 -m venv venv
source venv/bin/activate
pip install pydantic rich requests chromadb

# Local LLM (no API key needed)
ollama pull llama3.2
python3 -m agent.cli sample.jsonl

# OpenAI
export OPENAI_API_KEY=sk-...
python3 -m agent.cli sample.jsonl

# Gemini
export GEMINI_API_KEY=...
python3 -m agent.cli sample.jsonl
```

### Windows

```cmd
cd rag-message-assistant
python -m venv venv
venv\Scripts\activate
pip install pydantic rich requests chromadb

:: Local LLM
ollama pull llama3.2
python -m agent.cli sample.jsonl

:: OpenAI
set OPENAI_API_KEY=sk-...
python -m agent.cli sample.jsonl

:: Gemini
set GEMINI_API_KEY=...
python -m agent.cli sample.jsonl
```

The CLI prints each decision in a colored panel, an evaluation table per case, and a final summary across all cases. Results are written to `sample_results.json`.

## Backend Priority

The LLM wrapper auto-selects a backend in this order:

1. `GEMINI_API_KEY` set → Gemini
2. `OPENAI_API_KEY` set → OpenAI
3. Otherwise → Ollama (local)

No code change needed to switch.

## Scenarios Covered

- New prospect welcome (SMS, day 0)
- Long-horizon prospect follow-up (email, day 3)
- No-show re-engagement (SMS)
- Cancelled tour with manager cross-sell (email with budget cue)
- Consent block fallback (SMS blocked → email)
- Resident 90-day renewal notice (email)
- Undecided renewal with intent branching (SMS)
- Resident move-in welcome (email)
- Loyalty enrollment nudge (email)
- All consent false → do-not-send with reasoning
- Spanish-locale prospect (SMS in Spanish)
- Renewal details → e-sign flow (email)

## Evaluation Metrics

Each case is scored on six dimensions and a weighted composite:

| Dimension        | Weight | What it checks                            |
|------------------|:------:|-------------------------------------------|
| Channel match    | 25%    | Did the agent pick the right channel?     |
| Timing           | 15%    | Business hours, same day, right offset    |
| CTA              | 15%    | Is a call-to-action present?              |
| Opt-out          | 15%    | Stop/unsubscribe instructions included    |
| Personalization  | 15%    | Recipient name used in the body           |
| Semantic         | 15%    | LLM-judged similarity to expected body    |

## Compliance Guard

This is the only deterministic stage, on purpose. It:

- Blocks output containing fair-housing-protected terms (race, religion, disability, national origin, familial status).
- Verifies opt-out language on SMS/email; auto-appends a safe default if missing.
- Fails the case if compliance can't be guaranteed.

A model hallucination can never reach a user with a protected-class reference.

## Extending

- **Add more scenarios** by appending to `sample.jsonl`. No code change required — the agent picks them up as new few-shot examples.
- **Swap the vector DB** by replacing the ChromaDB calls in `agent/knowledge_store.py`. Pinecone, Weaviate, or pgvector all fit the same interface.
- **Change the LLM** by editing `agent/llm.py` — the backend adapter is small and easy to extend with Claude, Mistral, or any other provider.

## Author

Built by **Aryan Shaikh**.

For a function-by-function walkthrough of the design, see [`WALKTHROUGH.txt`](./WALKTHROUGH.txt).
