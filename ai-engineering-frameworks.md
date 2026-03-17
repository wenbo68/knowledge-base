# ai agent frameworks
1. LangGraph
    - faster to setup/code
        - langgraph abstracts the db layer
            - you just write 1 line of code (ie checkpointer)
    - difficult to maintain/debug
        - checkpoints are serialized into binary
            - not human readable
        - graph can become large
            - hard to unit test
            - must use langsmith to debug
                - langfuse won't work well
2. PydanticAI + Logfire + Temporal
    - slower to setup/code
        - must implement db layer on your own
        - must run Temporal daemon for durable agent execution
            - eg resume from crash, human in the loop
    - easier to maintain/debug
        - we can just serialize checkpoints to json
        - no graph; just python functions using dep injection
            - easy to test
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
- programming paradigm
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
- cpu core utilization
    - python has global interpreter lock (gil): the python process/program can only use 1 core
    - gunicorn/uvicorn: creates multiple identical processes from your code to use more cores
        - behaves like separate servers
            - they don't share memory: you cannot pass data/tasks btw them
            - has a load balancer to distribute requests among these processes
        - best practice: create 2*#core+1 processes
- parallel processing
    - use parallel processing libs (eg multiprocessing)
        - forks the current process into new processes to use more cores
            - can share memory if you use `multiprocessing.shared_memory`
        - should not be used in web servers like gunicorn/uvicorn
            - gunicorn/uvicorn already creates lots of processes to use more cores
                - no cores left for parallel processing cpu-bound tasks
                - must do time slicing between processes within the same core
    - in web servers, offload cpu-bound tasks out of python completely to background workers
        - eg celery
- db connection
    - driver libs
        - psycopg
            - psycopg2: outdated
            - psycopg3: sync + async
                - but async slower than asyncpg
        - asyncpg
            - only async
    - orm
        - sqlalchemy
    - process-level connection pool
        - where?
            - psycopg/asyncpg: built-in connection pool
                - if you are using sqlalchemy for orm, use sqlalchemy connection pool
                - if you just write raw sql, use psycopg/asyncpg connection pool
            - sqlalchemy: built-in connection pool
        - these are only process level connection pools
            - gunicorn/uvicorn creates multiple processes, each with its own connection pool
                - eg if 1 server has 10 processes each with 20 db connections, that server has 200 connections
                - but postgres only has 100 connections by default
    - pgbouncer: db-level connection pool
        - handles when servers have more db connections than the db has
        - not needed if the db has a built-in pgbouncer
            - eg neon postgres

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