# Concurrency
- why care?
    - when a user uses your app, your app server needs to run the code using its cpu
    - now if a 2nd user uses your app at the same time, that 2nd user has to wait until the server's cpu is free
        - why? 2 possibilities:
            1. the server's cpu is busy running code (ie doing compute)
                - most code finishes fast (nanoseconds): eg creating a uuid
                    - making the 2nd user (and all users behind the 2nd) waiting nanoseconds is fine
                    - **solution: none (blocking fast computes are fine)**
                - some code takes a long time (seconds): eg image processing, llm inference
                    - if 50 users run slow code at the same time, the 1st user will start immediately
                    - but the 2nd user might wait 5s, the 3rd user 10s, the 4th 15s, etc
                    - **solution: multi-threading (to enable non-blocking slow computes)**
            2. the server's cpu is just idly waiting for something
                - cpu might be waiting for your disk, a db server, another app server, etc (but not your ram)
                - **solution: asynchronous programming (to enable non-blocking I/O)**
        - what if your code has both slow compute and I/O?
            - **solution: use both threads (for non-blocking slow computes) and async (for non-blocking I/O)**
- asynchronous programming
    - use case
        - during 1 task, your cpu/ram becomes idle b/c the code is I/O, not compute
        - so your cpu/ram starts working on another task while waiting
    - characteristics
        - all done within 1 thread (aka the event loop)
        - less ram usage than multi-threading
    - key caveats
        - typical async functions have both compute and I/O code
            - your cpu/ram is not free when it's running the compute part of your async function
            - only when awaiting I/O code does your cpu/ram become free to attend other computes
        - awaiting vs not awaiting the async functions
            - await: the cpu works on async function A's compute part -> cpu starts A's I/O -> cpu works on others in the event loop -> when A's I/O is done, "continue A" is added to the very end of the event loop
            - no await: just add entire async function A (both compute and i/o) to end of event loop
- threading (multi-threading)
    - use case
        - many tasks all needs your server's cpu/ram and you want to progress them together
        - so each task uses a thread
    - characteristics
        - each thread uses about 8mb ram: too many threads will overload your ram and crash your server
            - so only a set number of threads in thread pool (eg 40 in python's default thread pool)
            - meaning if each user needs a thread, your server can only support 40 concurrent users
                - the 41st user cannot login, etc. (needs to wait)
    - 2 hardware-level realizations:
        - time slicing: 1 cpu core quickly switching among different threads
        - parallelism: you have many cpus; each cpu works on 1 thread

# Concurrency: Nodejs vs Python
- both uses an event loop (1 thread)
    - but different await async function syntax
        1. nodejs: awaiting vs not awaiting async functions
            - await: cpu/ram starts executing your async function; subsequent code will wait
            - no await: cpu/ram adds the async function to the end of the event loop and starts exeuting subsequent code
        2. python
            - await: same behavior
            - no await: does nothing
                - in python, you must either await or manually put the async function into a scheduler like fastapi backgroundtasks
            - manually put async functions into a scheduler
1. but python/fastapi has a thread pool for multi-threading
    - runs api/helper async functions using main event loop
        - it trusts that your async function is fast compute + i/o so that it's safe to in the cheap event loop
    - for sync funtions
        - runs api sync function in a thread
            - it trusts that your sync function is compute (fast or slow) so it makes it non-blocking using thread
        - for helper sync function with slow compute
            - you manually put it in a thread
2. nodejs doesn't have a thread pool available for devs
    - nodejs is only for fast compute and i/o
    - nodejs: how to handle slow compute tasks?
        1. deploy on faas: each api call is a separate mini-server/container
            - users (in fact, each api call) cannot block one another
            - but the container will only last a short time so the compute might get cut off in the middle
        2. create and manage your own thread pool
            - hard and complicated
        3. have a separate backend server for multi-threading the slow compute tasks

# confusion
- cpu-bound tasks = slow compute tasks
- i/o-bound tasks = i/o tasks

# process vs thread
- process
    - characteristics
        - os isolates processes to have independent memory, etc
            - one process crashing doesn't affect others
        - creating a new process or switching between processes is slow/heavy
    - example
        - most apps are 1 process
        - for browsers, the browser is a process and each tab is a process
- thread
    - characteristics
        - os creates threads for a process when requested
            - threads share the process's memory
        - creating a new thread or switching between threads is fast/light
    - example
        - each thread does 1 task for an app
        - microsoft word may have a thread for ui, spell check, auto save, etc.