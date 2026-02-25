# Guardrails
- mostly neuro-symbolic: uses a combination of deterministic (symbolic) & probabilistic (neuro networks) approaches
- deterministic: uses egex, keyword lists
    - too brittle: cannot detect "tone," "implied jailbreaks," or "hallucinations."
- probabilistic: uses small language models (slm) specialized in the required niche

## categories
1. input
    - where: checks user input before it reaches llm
    - goal: stops jailbreak, prompt injections, toxic inputs, etc.
    - example
        - LlamaGuard 4: multimodal; 14 harzrd categories (for user input)
        - Lakera Guard: market leader against prompt injection
        - NVIDIA NeMo Guardrails: checks user input against arbitrary topics
2. RAG
    - where:
        - checks retrieved nodes before it reaches inference for relevancy and context injection (hidden commands in node)
            - example
                - relevancy: re-ranking (llamaindex can use a model to score the retrieved chunk based on user query)
                - context injection: pass the documents through lakera guard before indexing them in the first place
                - context injection: just let a separate llm/slm summarize the document then pass the summary to your inference
        - checks inference answers against the retrieved nodes (reduces hallucination)
            - example
                - just let your llm or a slm check the inference answer b/f showing the user
                - force inference answers to cite the chunk; then deterministically kill answers wihout or with nonexistent citations
    - goal: stops irrelevant documents from reaching inference, context injection (hacker puts prompts in uploaded files), hallucination, etc.
3. agentic
    - where: during llm/agent workflow
    - goal: stops infinite loops, too much api credit usage, unauthorized execution, etc.
    - example
        - langgraph built-ins: e.g. recursion_limit=50
        - NeMo execution rails: intercept tool calls, eg delete db
4. output
    - where: checks llm answer before it reaches user
    - goal: stops hallucinations, PII leakage, toxic responses
    - example
        - microsoft presidio: standard for deterministic PII
            - not good enough for contextual PII (need to pair presidio with llamaguard 4)

## when your llm app has a problem, do you better your llm (probabilistic) or do you bring in external, deterministic tools?
- if the inference answer is unacceptable => use guardrail to deterministically kill the possibility
- if the inference answer is bad => improve your llm via prompt engineering, fine tuning, add/improve rag, switch models, etc.
- modern AI engineering architecture: deterministic sandwich
    - use LLM (probabilisitc model) only when needed: the task rules are too complex to be written deterministically (eg language)
    - and wrap the probabilistic part (embedding/inference models) within deterministic layers (parsing/chunking, guardrails, etc.)