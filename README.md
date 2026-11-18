# From Prototype to Production: Hardening an LLM Chatbot

> Built on the [Hyperskill starter project](https://github.com/hyperskill-content/llm-evals-2-preparing-for-prod) as part of the **AI Engineer Intensive Bootcamp (2026)**.
> Part 2 of the series, continuing [LLM-evals](https://github.com/DMLearn/LLM-evals). The base chatbot was provided; prompt management, persistent memory, guardrails and cost control are my implementation.

A working RAG chatbot prototype (LangChain + Qdrant + tool calling) taken one step closer to production: **versioned prompts, persistent and token-bounded chat history, input guardrails, and per-user/per-session cost control**, all traceable in Langfuse and LiteLLM.

---

## What I implemented

- **Prompt management & versioning (Langfuse)** – all three system prompts moved out of the code into Langfuse, fetched at runtime and linked to each generation. Prompt changes no longer require a deployment, and every response can be traced back to the exact prompt version that produced it.
- **Persistent chat history (Redis)** – in-memory conversation replaced by `RedisChatMessageHistory` with a TTL of 10 minutes per session, so sessions expire automatically and multiple users don't collide.
- **Token budget per turn** – `trim_messages` keeps the history under 600 tokens, applied before both the tool-calling and the response chain.
- **Input guardrails (NVIDIA NeMo Guardrails)** – a self-check input rail blocks prompt injection, attempts to extract the system prompt, abusive or explicit content, personal data and out-of-scope requests (orders, returns, tracking) *before* they reach the main chain.
- **Cost control via LiteLLM proxy** – virtual keys with budget limits; every LLM *and* embedding call is tagged with user and session ID, so costs can be attributed per user and per conversation.

## Engineering challenges I solved

Bootcamp tasks describe the *what*. These are the problems that came up along the way (details in [learnings/learnings.md](learnings/learnings.md)):

| Problem | Solution |
|---|---|
| `langchain_redis` loses `AIMessage.tool_calls` on serialization, breaking the tool-calling flow after a reload | Serialize `tool_calls` into `additional_kwargs` before saving and restore them on load |
| `RunnableRails` cannot serialize LangChain message objects passed alongside the user input | Convert messages to dicts before the rail, deserialize them inside a `RunnableLambda` wrapper |
| Using rails standalone caused a **second, discarded LLM call** on every allowed input | Pass the real chain as `runnable` to `RunnableRails`: one LLM call per turn |
| Langfuse session metadata got lost when the chain was invoked through the rail | Inject the metadata explicitly inside the wrapper |
| Embedding calls appeared in LiteLLM as unrelated entries, so RAG costs couldn't be attributed | Session-aware embedding instance in the retrieval tool, tagged with user, session and tool name |

## Architecture

```
User (console)
   │
   ▼
Redis history ──► trim (≤ 600 tokens)
   │
   ▼
NeMo input rail ──(blocked)──► refusal message
   │ (allowed)
   ▼
context chain ──► LLM with tools ──► SmartphoneInfo ──► Qdrant
   │
   ▼
final-response chain ──► answer ──► Redis history

Prompts      ◄── Langfuse prompt management (versioned)
All LLM and embedding calls ──► LiteLLM proxy (virtual keys, budgets, user/session tags)
All steps   ──► Langfuse tracing
```

## Results

| Versioned prompts in Langfuse | Generations linked to their prompt version |
|---|---|
| ![Prompts](learnings/task1_prompts_overview.png) | ![Linked generations](learnings/task1_context_prompt_linked_generations.png) |

## Tech stack

Python · LangChain · OpenAI API (gpt-4o-mini) · Qdrant · Redis · NVIDIA NeMo Guardrails · LiteLLM · Langfuse

## Run it

```bash
# Infrastructure
docker run -d --name qdrant -p 6333:6333 qdrant/qdrant
docker run -d --name redis  -p 6380:6379 redis redis-server --save 60 1
# Langfuse (self-hosted): https://langfuse.com/self-hosting
# LiteLLM proxy:          https://docs.litellm.ai/docs/proxy/quick_start

# Prompts: create context_system_prompt, review_system_prompt and
# goodbye_system_prompt in Langfuse (Prompt Management)

cp .env.sample .env      # OPENAI_BASE_URL = LiteLLM proxy, OPENAI_API_KEY = virtual key
pip install -r requirements.txt
python main.py
```

---

<details>
<summary>Original task description (Hyperskill)</summary>

The project is structured into four tasks, see the [tasks](./tasks) folder:

1. **Prompt management and versioning**
2. **Chat history management**
3. **Safety and guardrails** with NeMo Guardrails
4. **API keys and budgets** via a LiteLLM proxy

</details>
