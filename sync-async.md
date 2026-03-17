# sync vs async
- broad meaning
    - sync: blocking
    - async: non-blocking
- sync/async can live in 3 layers of your app
    1. code level: concurrency
        - sync: i/o-bound task blocks the thread from doing other things
        - async: i/o-bound task doesn't block the thread from doing other things
            - benefits
                - thread can do other things while waiting for i/o
            - done via asyncio in python and async/await in js/ts
            - but the task may still block the request from returning
    2. network level: only client/server requests/responses
        - sync: requested task blocks the request from returning
        - async: requested task doesn't block the request from returning
            - benefits
                - request can return immediately
                    - user doesn't wait for task to finish
                - decoupled
                    - task cannot crash the request
            - done via background tasks
                ie decoupling the requested task from the request
    3. archtecture level: distributed systems (only server/server requests/responses)
        - sync: server B processing the task blocks server A from dropping the connection
            - regular distributed systems: servers call each other directly
                - eg vercel nextjs server waits for railway backend, which waits for neon db
        - async: server B processing the task doesn't block server A from dropping the connection
            - benefits
                - server A doesn't need to hold the connection open
                    - minor: the i/o just sits longer in the event loop of server A
                - decoupled
                    - db server doesn't cause a backend error and crash the request
                    - db server doesn't crash when backend receives lots of requests
            - done via: server A just puts the task somewhere for server B to check
                - cron jobs: server A writes the task to db and server B runs cron job to check db
                - queue + workers: server A writes the task to a queue and workers (in another server) instantly process the task
                    - eg redis queue + celery workers
                - durable execution: server A writes the complex/multi-step task to durable execution server, which instantly processes the task
                    - eg temporal, dbos
                    - why needed?
                        - orchestrates the multi-step: would require multiple queues and lots of logic otherwise
                        - durable: writes the progress (which step it's on) to disk instead of just ram
                            - so that even if the durable execution server crashes, it can resume from where it left off
                        - perfect for multi-step, expensive, yet flaky tasks
                            - eg ai agents

# worker vs queue vs broker?
- worker: a while loop polling the queue constantly
- queue: a list of tasks waiting to be processed
- broker: a server that manages the queue
    - eg redis, rabbitmq, kafka

- todo
    - finish layer 3
    - then checks chat history to see if you can understand everything with the above mental model