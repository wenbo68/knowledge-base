# ai agent frameworks
1. LangGraph
    - Who: enterprises building complex, multi-step agents
    - Why: features like state, routers, checkpointers are hard to build from scratch
2. PydanticAI (competes with langchain)
    - Who: startups; developers who wants to stay away from proprietary tools
    - Why: everything is type safe, transparent & easy to debug (whereas langchain needs langsmith for debugging due to non-transparency)
3. LLM Gateways (eg LiteLLM, Portkey)
    - Who: both langgraph and pydanticAI can use this
    - Code: write everything in openai code, then call the gateway, which decides what model to send request to

# langchain/langgraph
- langchain vs langgraph
    - langchain: allows workflow without loops
    - langgraph: allows loops in the workflow (eg model can check its own response and regenerate the response if it was bad)
    - langgraph is built on top of langchain but adds 3 main concepts:
        1. router: allows nonlinear workflow (ie routing to an arbitrary node based on some conditions)
        2. state: all nodes access/update the same state object (stored to db by checkpointer)
        3. checkpointer: autosaves state to db
            - the persistence enables human-in-the-loop (agent shuts down at a node -> human approves state -> agent reboots to proceed)
- langchain inference model selection
    1. Using provider library, eg. from openai import OpenAI
        - Pros: precise control, easier debuggin (you know exactly what is being sent), new features available asap
        - Cons: have to rewrite all API calls when you switch provider
    2. Using langchain standard abstraction layer, eg. from langchain_openai import ChatOpenAI (returns standard langchain chat obj; just run obj.invoke("question") to get a response)
        - Pros: written in langchain code, just import a different class when you switch provider
        - Cons: need multiple imports (or if/else imports) if you want users to select different models
    3. Using langchain unified abstraction layer, eg. from langchain.chat_models import init_chat_model (also returns standard langchain chat obj)
        - Pros: just use a different string when you switch model; standard for langchain/langgraph since late 2024
        - Cons: some milliseconds of latency looking up correct models dynamically
- langchain type safety
    1. Inputs (for tools): should be strictly typed
        - no need to write pydantic obj for tools
        - just declare input/output types for the function
        - then the @tool decorator will use pydantic under the hood
    2. Outputs (inference model responses): should be strictly typed to better integrate with app/business logic
        - just declare a pydantic obj and pass to langchain chat obj (created either via standard/unified abstraction) via with_structure_output()
        - new_chat_obj = langchain_chat_obj.with_structured_output(pydantic_model)
    3. The middle (reasoning/memory): keep it flexible; just let langchain handle it; don't over-engineer the thinking process
        - in langchain, hard to force types on memory
        - in langgraph, can use pydantic/TypedDict to define memory structure
- langgraph memory
    1. short-term memory: memory within 1 conversation (each conversation has a different thread id)
        - use checkpointer
            - snapshots of langgraph state
                - stored in db (PostgresSaver/SqliteSaver/etc) OR in memory (InMemorySaver, only really used if you don't have db set up yet)
            - state can contain anything
                - for chatbots, the state will almost always store msg (chat hist)
    2. long-term memory: memory that spans all conversations
        - 2 types: user info (personalization), doc info (company policies, user uploaded files, etc)
            - user info: new rag for each app in the past; now uses store (InMemoryStore/PostgresStore/etc); dev or agent decides what should be stored in long term memory; rag under the hood (returns relevant info based on a query)
            - doc info: implement your own rag for each app; use either llamaindex VectorStoreIndex or langchain VectorStore
        - tool: mem0

# python
- every file is executed from top to bottom line by line
    - if the line is an import, it will execute the imported file from top to bottom
        - only if the import has not been executed yet
        - if the import has been executed, it will not be executed again
    - everything defined at 0 indent is exported
        - whether it's a class/function/var
- context var? what's the java/typescript equiv?
- python paradigm
    - comparison
        - java: oop + functional (stream/lambda)
        - typescript: mostly procedural (react comp) + functional
            - oop possible
        - python
            - procedural: regular helper function that can change some global state
                - eg api endpoints
            - functional: functions that returns new values instead of changing input
                - eg list processing
            - oop: when to use oop? think in terms of your states/variables
                - constant state: global variable
                - editable states: global variable + procedures
                    - avoid static class (class without init) as the file itself acts like static class in python
                - editable states + complex init: oop (class states + methods)
                    - eg env var config
                - you need many instances: oop
                    - eg db connections

# fastapi
- `uvicorn app.main:app`
    - go to `app` folder `main` file and run the var `app`
- main.py
    - lifespan: execute sth on app startup and shutdown
    - fastapi app
        - middlewares
            - observability: logging, metrics, tracing
            - security: cors
        - custom global error handling: based on error type
        - router
            - nextjs uses automatic file-based routing but in fastapi you must manually set up routing
        - state: storage for easy retrieval elsewhere
            - limiter