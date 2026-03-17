# system design

## compute (app server) infrastructure: stateless (each request is separate; nothing will span across requests)
1. on-premise: physical server (anything that's not on-premise is cloud)
2. IaaS: infrastructure as a service (aka vps)
    - pros: you do whatever you want; you have root access; you can ssh in
    - cons: you set up and maintain the os/security/scaling
    - example: aws ec2, contabo
3. PaaS: platform as a service (no cool nickname... maybe platform?)
    - pros: you just give the provider your code; they manage the os/security/scaling for you (opinionated)
    - cons: more expensive; less control than iaas (you don't have root access and usually cannot ssh in)
    - example: railway, render
4. Caas: containers as a service (industry standard for AI)
    - provider runs your kubernetes cluster of docker containers (keeps the cluster alive)
    - you give provider detailed yaml files that dictate how many cpu/gpu each service can use and which other services it can talk to
    - so caas is unopinionated unlike paas: you (not the provider) determine how it runs
    - example: aws eks, google kubernetes engine (gke)
5. FaaS: function as a service (aka serverless)
    - pros: scales down to 0 (you pay 0 when no one is using); scales infinitely
    - cons: cold start; not good for something that needs to stay open all the time (eg db connection, websocket); database should be serverless too
    - example: aws lambda, vercel functions, cloudflare workers
    - how?
        1. a user calls an api route (if you deployed on serverless, each api route is a serverless function)
        2. if there's a frozen micro-container for that api route (aka serverless function), the provider thaws it (<10ms) and reuses it
        3. if there's no frozen containers for that api route, the provider spins up a container (<0.5s) and uses it
        4. after the api returns a http response to user, the container is frozen for 5-15 min (depends on provider) and then killed
6. Edge faas
    - cdn nodes (cloudflare has 330+) are more widespread than server warehouses (aws has 35)
    - traditionally, cdn nodes cannot do compute (only store files)
    - but vercel & cloudflare put tiny cpus (128MB RAM) in all their cdn nodes, so now their cdn nodes can do compute
    - you upload code to the provider server -> a copy of your code (1-5MB) is stored in each cdn node
    - certain user requests will go directly to the closest cdn node for much faster responses
    - example: vercel edge (only middleware code uses edge by default; other codes won't use edge unless specified)

## storage (db server) infrastructure: stateful (some info will span across requests, eg session info)
1. on premise
2. iaas
3. paas
    - example: aws rds, railway postgres
4. caas
    - if you have a caas app, you can store your db in the same caas/cluster
    - but need to download Operator software to manage your db within the cluster
    - operator example: CloudNativePG (gold standard), StackGres, ChromaDB Helm Chart (use when chromdb + caas)
5. "serverless" paas
    - specialized paas that handles serverless apps (so that sudden high traffic from serverless app doesn't block your db connections and make it very slow)
    - the providers scale the db server down to 0 (freeze it when no traffic for some time)
    - and up infinitely (but each provider implements differently)
    - example: aws aurora serverless, neon, upstash, pinecone
    
## file caching (not to be confused with data caching)
1. blob storage
    - infinitely scaling disk in cloud
    - eg uploadthing, aws s3, cloudflare r2
2. content delivery network (cdn)
    - servers/disks (ie cdn nodes) spread across the world
        - no compute: don't run code
        - just stores your static files, images, videos, etc.
            - user requests them from closest cdn node via urls

# scaling: when and how
- 3 things to scale: disk (storage), ram (compute), cpu (compute)
- 2 ways to scale: vertical, horizontal
- granularity (can you scale only the part you need?)
    - vertical
        - on prem: disk + ram + cpu
        - iaas: disk + ram/cpu
        - paas: disk/ram/cpu
        - caas: horizontal under the hood...
        - serverless: disk + ram/cpu
    - horizontal: no granularity; you scale disk/ram/cpu together

## app server scaling
- you scale when your app server doesn't have enough RAM/CPU
    - need more **ram/cpu** if...
        - many simple tasks: can be parallelised
            - method: horizontal
        - 1 complex task: cannot be parallelised (eg AI, image)
            - method: vertical
- make sure you delete old logs & docker images so you don't have to worry abt disk space
    - logs shouldn't be on the server anyway: they should be streamed instantly to a centralized logger (eg datadog, ELK)
- need to manage each app server's db connection pool when you scale your app server
    - app server scaled vertically: increase max connections (eg 20 -> 200) of that 1 pool
    - app server scaled horizontally: each app server has 1 pool (eg each app server has 20 db connections)

1. on premise
    - vertical: power off server -> open case -> insert more ram sticks or better cpu-> power back on
    - horizontal: set up another server -> update your physical/software load balancer to include the new ip addr
2. iaas
    - vertical: stop the instance -> change instance type to better one -> start the instance
    - horizontal: spin up another vps -> update your load balancer
3. paas
    - vertical: go to settings -> change plan
    - horizontal: go to settings -> drag the instance count slider from 1 to 5
4. caas
    - vertical: edit yaml config (resources.requests.memory: "4Gi") -> apply -> kubernetes will restart
    - horizontal: run cmd "kubectl scale deployment my-app --replicas=5" OR configure HPA to run cmd automatically based on RAM/CPU%
5. faas/edge
    - vertical: go to settings -> change memory size or cpu
    - horizontal: do nothing... serverless automatically means infinite horizontal scaling

## db server scaling
- you scale db servers when it doesn't have enough disk or enough RAM/CPU
    - need more **disk** if...
        - you have too much data OR too many writes (db must write to disk, and more concurrent disk writes are allowed on larger disk)
        - methods
            - vertical: standard
            - horizontal: sharding (each disk stores different data) -> extremely complicated
    - need more **RAM** if...
        - reads/writes are slow
            - for reads
                - steps
                    1. db reads from ram first (fast)
                    2. if data not in ram, db needs to read from disk (slow) and add to ram
                - if ram is full, some reads will be slow
            - for writes
                - steps
                    1. db saves your write cmd to disk first (fast)
                    2. db finds from ram first and change ram data (fast)
                    3. if data not in ram, db finds from disk and load to ram and change ram (slow)
                    4. db sync disk with ram (slow)
                - if ram is full, db must clean ram by syncing disk with ram first and then work on your new writes
        - methods:
            - vertical: if ram shortage due to...
                - writes (there can only be 1 master; otherwise you have to shard)
                - complex/large reads
                    - db servers needs to load all that's read to ram
                    - cannot be parallelised
            - horizontal: if ram shortage due to many simple reads (there can be many slaves)
    - need more **CPU** if...
        - your reads are too mathematically complex (complex join/sorts) for current cpu
        - methods:
            - vertical: if cpu shortage due to...
                - writes (again there can only be 1 master)
                - complex/large reads
            - horizontal: if cpu shortage due to many simple reads
- partitioning: divide 1 table to more specific tables so that db server reads from smaller tables
    - partitioning is not scaling
    - used against complex/large reads to save ram/cpu

1. on premise
    - vertical (writes): get better server -> pg_dump in old server -> restore data to new server
    - horizontal (reads): complicated...
2. iaas
    - vertical: stop vps -> resize -> start vps
    - horizontal: complicated...
3. paas
    - vertical: change plan
    - horizontal: click add new read replica -> get a read_replica_url -> change app code to send writes to database_url (master) and reads to read_replica_url (replica)
    - some paas (eg planetscale, cockroachdb, etc) will scale and route for you automatically, ie you just use 1 url all the time
4. caas
    - vertical: edit yaml config (increase StatefulSet limits)
    - horizontal: use a Kubernetes Operator (eg CloudNativePG) -> request operator for more db instances -> handles master-slave automatically
5. "serverless" paas: auto-scaling

## connections
- http
    - synchronous request
    - must be fast or will timeout
- websocket
    - can be held open for a long time by the server in ram (doesn't consume cpu)
- webhook