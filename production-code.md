# logging
- log on...
    - entry
    - exit (due to exception or return)
    - i/o (disk, db, api)
        - log llm api duration, tokens, and cost
    - slow computes (log the duration)

# retries
- retry on i/o (disk, other servers eg db, api)

# error handling
- if you don't handle an error, it kills the app/process on the server
- global error handler (the "Pokemon Catcher" at the framework level)
    - catches any unhandled errors, logs them, and sends a generic 500 error to the user
    - prevents the server from crashing
    - STOPS the current execution flow (drops the request completely)
        - but keeps the app/process alive for other requests/users
- try/catch (the specific safety nets)
    - why use it if we have a global handler?
        1. to CONTINUE execution
            - eg in a loop of 100 users, if 1 fails, catch it so the other 99 still process
        2. to have a PLAN B
            - eg if primary db fails, catch it and query backup db
        3. to give specific, helpful errors to the user instead of a generic 500
            - eg return "400 Insufficient Funds"
    - when to use:
        - i/o: retry first -> then catch
        - invalid state caused by user: user withdraws more money than they have
        - user input: used to need try/catch; now just use zod/pydantic instead
