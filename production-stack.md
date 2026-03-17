# agent stack

## stateless (app logic)
1. frontend code
    - stack: uses react; python can only create generic UI via streamlit/gradio
        - 2 types of frontend in 2026
            1. client -> backend server
                - client components call backend apis via...
                    - vercel ai sdk: 2026 standard for llm chat
                        - less boilerplate: streaming, chat client cache, etc.
                    - fetch: built-in + supports streaming
                        - use fetch + tanstack query...
                            - if you need more complex client cache than what vercel ai sdk provides
                            - for everything other than llm chat
            2. backend for frontend (bff): client -> nextjs/trpc server -> backend server
                - most teams are forced to use bff b/c they are using:
                    - nextjs server components/actions: already using nextjs server
                    - trpc (for better client cache DX): trpc runs on nextjs server
                    - nextauth: only accessible via nextjs server
                - if you don't use any of the above, you might still consider bff for:
                    - multiple backend servers (ie microservices)
                    - llm streaming control in typescript (closer to UI logic) rather than python
    - deployment: faas (eg vercel)
2. backend code
    - stack: uses python; langchain typescript is usable but llamaindex typescript is not good
        - auth: just use an auth provider (safer and less code)
            - for implementing auth on your own:
                - jwt: stateless (you don't store active user sessions in db)
                    - standard if you don't need to revoke sessions (user can stay logged in until jwt expires, and you can't do anything abt it)
                    - localStorage vs cookie: always use cookie (more secure)
                - session cookies: stateful (you store active user sessions in db)
                    - harder: only use when you need revocation
        - fastapi: better async support than django/flask
            - django/flask: only multithreading (django now supports async)
            - fastapi: async to enable non-blocking i/o; multithreading to enable non-blocking slow compute
            - in javascript, everything is async by default but in python, you have to consciously choose async libraries b/c async was added later on
        - pydantic: runtime external data validation (equivalent to zod)
            - typescript needs zod because types are erased at run time
        - tenacity: retry/resilience
        - slowapi: app/user level rate limiting
        - structlog: structured logging (better than python's default logging)
        - llm evaluation: DeepEval + Ragas
    - deployment:
        - paas (eg railway/render)
        - serverless if the time-out is long enough
            - aws lambda: 15min time out; good enough
            - vercel: 60s time out; mostly not enough (a long langgraph workflow might get shut down)

## stateful (db): almost always serverless in 2026
1. app/user data
    - serverless (eg aws aurora serverless, neon)
    - paas (eg aws aurora)
2. checkpoints
    - serverless (eg aws aurora serverless, neon)
    - paas (eg aws aurora)
3. vectors
    - serverless (eg aurora serverless, neon pgvector, pinecone)
    - paas (eg pinecone legacy, aws aurora)
    - caas/iaas/onpremise (eg chromadb)
4. blob
    - 

## self-hosted models (when you don't wanna use cloud apis)
1. inference
    - ml paas (eg aws sagemaker): regular paas (eg railway) don't have enough GPUs
        - eg aws sagemaker
            - 2 types:
                - sagemaker real-time: the instance is always on (you pay per hour)
                - sagemaker serverless: mostly unusable as cold starts (loading llm weights to vram) can take 30-60s (bad UX)
            - in sagemaker you can select the aws lmi container, which uses the inference engine vllm under the hood
            - steps:
                1. upload adapters to aws s3 (open model weights are usually not stored locally but just pulled from hugging face)
                2. choose an aws inference docker container or make your own via dockerfile/docker-compose
                    - contains dependencies and inference engine
                3. give env vars to the inference container for it to pull base models weights (from hf) and adapters (from s3)
                4. choose instance (based on model size) and count
                5. get the endpoints and use them in your app
2. embedding
    - regular paas
        - just add sentence-transformers to requirements.txt; when your backend container starts, it downloads the embedding model weights and holds them in RAM
            - reduces cost (sagemaker is expensive) and latency (no http/api calls btw backend server and sagemaker center)
        - you only deploy the embedding model as a separate docker container if you are using microservices and all of them need the same embedding model
            - you don't want all microservices to download the same weights individually
            - letting the embedding model have its own container will add latency (there will be http/api calls btw microservice container and embedding model container)
    - ml paas: only for the following reasons
        - massive embedding model that will be too slow on CPU
        - microservices that needs the same embedding models
            - if you don't want to deploy the embedding model as a separate container in your backend server, you can just deploy it on sagemaker (but there will be more latency)
        - devops says they only monitor sagemaker endpoints and don't want to monitor models in docker containers

## devops
1. ci/cd
    - github actions + deepEval
2. observability
    - concepts
        - logs: tell you why your app crashed
            - in production, you log to stdout/stderr and drain/forward them to some server via webhook
            - 3rd party log drains & storage
                - frontend
                - backend: Axiom, Datadog, Loki (provided by Grafana Cloud)
        - metrics: help you predict crashes
            - aka application performance monitoring (apm)
            - eg how long a request took in total
        - traces: waterfall time info explaining the total time a request took
            - how long a request took for each hop and how long it stayed on each server within your network of servers
    - stack
        - paas/faas
            - logs: paas/faas may drain your log and display them for some time
                - if you want to keep your logs for longer, you need to set up your own log drains
            - metrics: paas/faas will show your metrics
            - traces: paas/faas cannot show traces because your code needs some lib to send trace info
                - app traces
                    - use libs like opentelemetry in your code and send to datadog, sentry, or tempo (self-hosted)
                - llm traces
                    - langsmith: native integration with langchain/langgraph
                    - langfuse: open source; self-hostable
        - kubernetes: only used by large scale companies; they just pay datadog
            - logs
            - metrics
            - traces
        - on-prem/iaas: self-host the following if you don't wanna pay datadog or grafana cloud
            - logs
                - Loki: db for text logs
                - promtail: forward logs from server to Loki
            - metrics
                - Prometheus: scrapes your server (your app and cadvisor) for metrics
                - cAdvisor: reads from docker daemon for container metrics
                    - same server as your backend as it needs access to docker daemon
            - traces
                - Tempo: db for trace data
                - opentelemetry: forward traces from server to Tempo
            - dashboard
                - Grafana: displays loki, prometheus, tempo data
                - grafana + loki + prometheus + tempo lives in a separate server
                    - so that they survive when your own server crashes and you can see why your server crashed
3. reverse proxy
    - servers sitting in front of your servers to protect them
        - most paas/faas have them built-in
        - only set up reverse proxies if you use on-prem/iaas
    - types
        - content delivery network (cdn)
            - numerous servers spread across the world
                - caches your files
                - can also cache your data
                    - not widely practiced though because data may require frequent updates
                        - hard to quickly change cache in all cdn nodes
                        - better store data in a centralized place like redis so that it's easy to update
            - eg cloudflare
        - edge reverse proxy
            - sits in front of your everything to face the entire internet
                - protects your from bots and ddos
            - eg cloudflare
        - load balancer
            - sits in front of identical servers to distribute traffic
            - if you have massive traffic like netflix, you would need multiple copies of your frontend, backend/microservice, api/llm gateway servers
                - so you would need a load balancer in front of each
            - eg nginx, aws application load balancer (alb)
        - api gateway
            - only for backend
                - usually for microservices
                    - frontend just points to api gateway instead of microservices
                - but can use for monolith as well
                    - to handle auth, rate limiting, etc and spare backend compute
            - eg kong
        - llm gateway
            - only if you use llm apis
                - just point to llm gateway instead of providers
            - what it does
                - routes to different providers if one is down
                - tracks llm spendings for each provider and user
                - semantic caching (just give them your redis url)
            - eg litellm, portkey
    - request flow
        - the user requests ui
            - browser -> edge proxy -> frontend lb -> frontend server
        - the user requests data
            - browser -> edge proxy -> api gateway lb -> api gateway -> backend lb -> microservice -> llm gateway lb -> llm gateway -> llm provider
                - the api gateway specifies which microservice
                - the backend lb specifies which copy
        - SSR (nextjs): frontend fetches data before returning to user
            - browser -> edge proxy -> frontend lb -> frontend server -> api gateway lb -> api gateway -> backend lb -> microservice -> llm gateway lb -> llm gateway -> llm provider
4. ai guardrails
    - nemo guardrails
        - guardrail framework
            - you run both the user input and inference output through nemo
        - uses both deterministic and probabilistic guardrails
            - deterministic: regex/keyword
            - probabilistic: after deterministic checks, you tell nemo to use an llm to check the input/output
                - llama guard: open guardrail llm (need self-hosting)
                - lakera: managed guardrail llm
5. prompt management/versioning
    - 2 reasons
        1. you might change (update/rollback) the prompt without changing your code
            - in this case you don't wanna go through the entire ci/cd pipeline or do a full git revert
        2. non-coders may need to change the prompt
            - they can just use the prompt cms instead of cloning the repo and pushing the changes
    - you must version all prompt strings
        - system prompts
        - eval prompts: if you write your own eval prompts instead of using prompts from deepeval/ragas
    - 2 ways
        - built-in: langsmith, litellm, portkey
        - standalone: promptlayer, pezzo

## caching
- data caching
    - frontend/client cache
        - lives in user's browser
            - if match, no need to access backend
    - backend/server cache
        - lives in ram, eg redis (db in ram)
            - if match, no need to access db
        - nextjs/vercel has a built-in server cache that doesn't reply on ram db
            - so that developers don't need to learn redis
            - but trpc bypasses nextjs server cache
    - semantic cache
        - lives in vector db, eg redis can store vectors now
            - if match, no need to access llm
        - llm gateways have semantic caching built-in
            - just give them your redis url
- file caching
    - content delivery network (cdn)
        - eg cloudflare, aws cloudfront

## notes
- fastapi and langgraph should be different servers
- what if 