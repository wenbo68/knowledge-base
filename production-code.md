# middleware
- lives in your server (not a reverse proxy)
- use middleware if you want to run some code for every request on entry and/or exit
    - aka cross-cutting concerns
        - reverse proxy also handles cross-cutting concerns
            - edge proxy: ssl termination, global rate limiting (ddos protection)
            - load balancer: routing
            - api gateway: routing
        - middleware handles cross-cutting concerns that want more details/flexibility
            - observability: logging every request, timing metrics, add trace details
            - security: attach CORS headers to every request, CSRF protection
- what about rate limiting and auth?
    - not completely cross cutting
        - each request needs to be handled differently based on what endpoint it's accessing
        - better not use middleware

# observability
- logging: log to stdout/stderr on...
    - entry
    - exit (due to exception or return)
    - i/o (disk, db, api)
        - log llm api duration, tokens, and cost
    - slow computes (log the duration)
- metrics: use middleware to time every request
- traces: need to rely on libs like opentelemetry

# rate limiting
- global rate limiting: ddos protection
    - handled by reverse proxy
- app rate limiting
    - public endpoints: no need to login
        - strict rate limiting via slowapi
    - cheap/fast protected endpoints: eg change username
        - let them do whatever
            - giving them rate limit errors may lead to membership cancellation
        - just ban them later if needed
    - costly/slow-compute endpoints: eg llm api, image processing
        - strict rate limiting to be safe

# error handling
- note: error handling only works if something goes wrong but the server process is still alive
    - eg api call failed but app process is fine -> tenacity will retry
    - eg request caused an error but app process is fine -> try/catch will catch the error and crash that request with json
    - but what if something happens to the server/process?
        - eg aws reboots your server, memory leak caused out-of-memory kill, kubernetes replaces your server node to scale up/down, someone pulls your server plug, etc.
        - result
            - every request will crash with a 500/502 json
            - user must retry manually (hopefully the server is back up when user retries)
- retries
    - retry on i/o (disk, other servers eg db, api)
- try/catch
    - if you don't catch an error, it will keep bubbling up and crash the app (ie crash the process on the server)
        - meaning one request can cause an error that kills the entire app
            - you need a giant try/catch wrapped around the entire app to prevent this
                - aka global error handler
                - implemented by frameworks (eg fastapi, express, spring)
    - global error handler
        - what it does
            1. catches any unhandled errors (errors that bubbled to the very top)
            2. logs them
            3. sends a generic 500 json to the user (end of that request)
                - or a custom json based on error type
        - why we need it
            - only crashes that 1 request instead of crashing the entire app
            - later, devs can check the logs and add try/catch if needed
                - to handle the error and continue that request
                - OR to handle the error and crash that request but with a even more custom json
    - try/catch (the specific safety nets)
        - why use it if we have a global handler?
            1. to continue that request instead of crashing it
            2. to crash that request but with a even more custom json
                - different functions may throw the same type of error
                    - lose context if you purely rely on global error handler customization
        - when to use:
            - i/o: retry first -> then catch
            - invalid state caused by user: user withdraws more money than they have
            - user input: used to need try/catch; now just use zod/pydantic instead
- graceful degradation
    - during server startup (before any requests come in), unhandled errors will fail the server startup
    - in production, you must try/catch startup errors and finish the startup
        - so that you can crash individual requests later instead of having a server that cannot start
    - eg db down during your server startup
        - you can choose to start the server without db connection
        - and then try to connect to db later when requests come in