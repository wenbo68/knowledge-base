# running different languages

## terms
- compiler
    - converts source code to byte code, byte code to machine code, or source code to machine code
    - transpiler: converts source code to another source code (eg ts to js)

- interpreter
    - reads byte code and runs the matching pre-defined machine code
        - does not convert byte code to machine code
    - if a loop has 100 loops, interpreter has to go through it 100 times
        - whereas a compiler just turns the loop to fast machine code
        - thus a jit compiler is needed

- jit compiler
    - compiles parts of your byte code frequently accessed by your interpreter into machine code
        - so hotspots run fast

## languages
- C/C++/C#
    - how
        - past
            1. buy different computers/os
            2. download c compiler on each
            3. compile source code to machine code (executable) on each computer
            4. give machine code to user
        - now (also Rust)
            1. 1 computer/os
            2. download a c compiler
            3. compile source code to machine code for any os
                - cross compiling: compile for other os
            4. give machine code to user
    - why not just give source code to user?
        - c must compile and then run
            - aka ahead of time (aot)
        - c is largely for executables
            - source code cannot protect ip; machine code can
- Java
    - how
        - developer
            1. 1 computer/os
            2. download java compiler
                - OR the entire jdk: compiler + jre + dev tools
            3. compile source code to byte code (.class; executable)
            4. give byte code to user
        - user
            1. download a jre: jvm + default java libs
                - jvm: interpreter + jit compiler (HotSpot)
            2. run byte code
            3. interpreter runs the byte code
                - only slightly faster than python due to being strictly typed
            4. jit compiler compiles frequently used byte code to machine code (to make it as fast as c)
    - falling behind
        - computer users hate downloading jvm
            - they hate getting "need to update java" messages
            - phones are fine: most android phones have jvm pre-installed
        - java was made to solve the old C compile problem
            - but now that cross compiling is so easy in c, java is losing its value when it comes to executables (eg games)
                - still fine for saas: jvm is on server
    - IP protection
        - developers send byte code to users
            - can be easily de-compiled back to source code
        - devs can obfuscate the byte code
        - or just do saas: server runs byte code for user
            - user only sees ui code
- Python
    - how
        - developer
            1. give source code to user
        - user
            1. download python: compiler + interpreter + jit compiler (since python 3.13)
            2. run source code
            3. compiler compiles source code to byte code
            4. interpreter (pvm) runs the byte code
    - cpython (default python implementation you download)
        - interpreted languages like python is very slow for slow compute tasks (loops, etc)
        - thus cpython was made
            - python works as the glue while slow compute tasks are done by c libs (eg numpy)
                - but must make sure a data structure has only same data type throughout
    - IP protection
        - devs send source code to users
        - so python is either open source or saas
- js/ts
    - how
        - dev
            1. build source code
                - transpile: ts to js
                - bundling: squash all files to 1 or 2 big files
                - minification: remove spaces/comments and rename vars to a/b/c/etc.
                    - keeps file size small
            2. give source code to user (or to server with nodejs)
        - user
            1. download browser: includes engines like v8 in chrome/node or SpiderMonkey in firefox
            2. engine runs source code within browser
            3. parser converts source code to an abstract syntax tree (ast)
            4. compiler compiles ast to byte code
            5. interpreter (Ignition in V8) starts running the byte code
            6. jit compiler (TurboFan in V8) compiles hotspots that keeps receiving same var types to machine code
                - but when the var type changes, it switches back to interpreting
    - why python can't just copy js jit compilers when both are dynamically typed?
        - cpython owes its success to the c libs
            - an aggressive jit compiler can interfere with c libs' memory management


# memory management in different languages
- list concatenation: combining multiple lists into 1 big list
    - during concat, the small lists and the big list must exist in memory together
        - leads to memory spike
    - also, you're moving data to the small list and then the big list
        - wasted compute
    - solution: only use 1 big list and append directly to big list
        - when needed, created a new list with double the size and copy from old to new list
            - still has memory spike and wasted compute
            - but resizing happens rarely as you double the size every time
        - underlying implementation of ArrayList in java
- python
    - common memory spikes
        - python variables (aka python objects) use lots of memory
            - must avoid creating lots of python variables (eg 1 python var for each data point)
            - solution: create 1 python variable pointing to a numpy array
        - converting a python var (eg list) to numpy array
            - both the python list (lots of ram) and numpy array (smaller but still takes up ram) must exist in ram
        - python list concatenation
            - solution: must implement java ArrayList on your own
    - common slow operations
        - iterating through a list/matrix of same-type vars (eg filtering data)
            - other languages: for/while loop
            - python:
                - if loop is o(n) or slower: python loop = too slow
                    - in this case, we don't loop through the numpy array; we mask the numpy array (ie let c do the looping)
                - if loop is o(log n) or faster: python loop = fine
        - iterating through a list/matrix of non-same-type vars
            - cannot vectorize: cannot use np array (and thus no masking either)

# concurrency in different languages
- python
    - cpython memory management is not thread safe
        - so cpython added gil: only a thread can execute python at a time
            - even if a process creates multiple threads for parallel execution of some cpu-bound task, only 1 thread runs at a time
                - no parallel execution but also no race conditions (no need for locks/mutexes)
            - but if a thread is waiting on i/o, other threads can run python
                - so cpython multi-threading works for i/o-bound tasks
                - but you should just do async programming for i/o-bound tasks using 1 thread
    - thus cpython parallel processing libs create multiple processes instead of threads
        - so cpython parallel processing does not share memory by default unlike other languages
        - if you want to pass data between these processes, you must use inter-process communication (ipc)
            - by deafult, cpython serializes the data, sends them to the target process, which then has to deserialize the data.
            - but serialization/deserialization are slow compute tasks + takes up lots of ram
                - solution: let processes write results to a file but save to ram folders (eg /tmp, /dev/shm)
                - this way the main process can easily access results from all other processes via the shared memory (aka shm)
        - but since 3.8, parallel processing via `multiprocessing.shared_memory` can share memory by letting the os create a shared memory for the processes
            - you cannot change the size of the shared memory
                - meaning you must know how much ram each process needs in advance
            - you can only pass byte or numpy arrays, not python objects
            - you must write locks/mutexes to prevent race conditions like other languages