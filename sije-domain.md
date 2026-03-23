# players

- brand
    - job
        - may or may not design clothes
            - brand design -> give production orders to oem vendors
            - brand doesn't design -> choose design proposals from odm vendors -> give production orders to oem vendors
        - market/sell clothes
    - characteristics
        - most brands don't own factories
    - examples
        - nike, adidas, etc
- vendor
    - type
        - oem: receives design from brand and ships finshed products to brand
        - odm: makes designs for brand and ships finshed products (of selected design) to brand
    - job
        1. pay for raw materials
        2. ships raw materials to factory
        3. pay for factory
        4. ships finished products to brand
    - characteristics
        - very money-sensitive
            - must find solution (eg pay again) if something goes wrong within the supply chain
            - gets paid a fixed amount by brand after brand receives finshed product
        - may or may not own factories
    - tool
        - erp: used by office workers to track the office operations (eg money, administrations, etc)
            - what problems does it solve?
                - centralizes data storage across different vendor departments/teams so everyone sees the same numbers
                    - used to be stored on papers or separate databases
            - what problems remain?
                - still required humans to map raw data to erp
                    - may need excel to store the intermediate data (swivel chair problem)
                - not connected to mes
                    - need humans to communicate via gmail
        - scm: used by office workers to track flow of goods (eg shipping, customs, warehousing, etc)
            - what problems does it solve?
            - what problems remain?
                - data provided by external parties (eg shipping companies, customs, etc)
    - examples
        - sije's current customers: youngone, yic
- factory
    - job
        - turns raw material into finished product based on design
    - tool
        - mes: used by factory managers to track physical production
            - what problems does it solve?
                - workers can quickly update status (eg by scanning barcodes)
                    - used to require a supervisor checking manually with devices like stopwatch, boards, etc
            - what problems remain?
                - workers manually upload data to mes
                - not connected to erp
                    - need humans to communicate via gmail
                - hard to determine how factory performance in mes affects vendor money in erp
    - examples
        - in vietnam, etc

# sije's solutions

- monolog: sends clothes production factory data to monolis
    - monolog sensors
        - on actual sensors attached to sewing machines
        - send/receive mqtt messages
    - monolog line
        - on edge server in factory
        - runs mqtt broker to communicate with the sensors
    - monolog main
        - on cloud server
        - runs kafka to aggregate data from all factories
- monoparts: sends raw material (eg button, fabric, etc) production factory data (eg count, bill of material, etc) to monolis
    - TypeORM entities for postgres: in src/entities
    - mongoose schemas for mongodb: in src/schemas
- monolis: contains erp/scm/mes tailored to clothes industry in one place
    - value: reduces erp/scm/mes silos and communication overhead (eg gmail, docs, etc)
    - receives data from monolog and monoparts
    - some cusomters may buy monolis without monolog
        - user will manually upload factory data to monolis
    - monolis has 139+ tables that containsß:
        - ERP Features: Sales Orders, Invoicing, Costing, Payments, Approvals.
        - SCM Features: Suppliers, Purchase Orders, Warehouses, Shipments, Product Loading.
        - MES Features: Factory Sites, Machines, Needles, Tools, Operations Breakdowns, Today's Output.
- monolis + monolog
    - serves 2 purposes
        1. factories cannot lie to vendors
        2. no need for docs and gmails for factories/vendors to communicate
    - latency
        - within monolog
            - seconds: streamed via mqtt/broker
        - across monolis and monolog
            - 5 minutes acceptable

# silo within the monolog/monoparts/monolis ecosystem??

- schema silo
    - different IDs for the same entity across monolog, monoparts, and monolis
- analytical silo
    - what no silo looks like
        - can use 1 sql query to join data from all 3 systems and create bi/dashboards
    - how do we know?
        - separate databases but no ways to query all simultaneously
            - no graphql layer to unify them
            - no etl to aggregate them into a single data warehouse/lakehouse
    - what users probably do
        - exporting csvs from each system and stitching them together in excel for bi
- operational silo
    - what no silo looks like
        - an order is added to erp, which automatically sends a msg to mes and scm.
    - how do we know?
        - no inter-service communication (eg http calls, grpc, brokers) in codebases
            - only broker exists within monolog for communication between sensors, edge, and cloud
    - what users probably do
        - 1 user exports data from erp, another user re-enters the data to scm, then another to mes
- are we trying to break the silo with a knowledge graph and agent?
    - meaning that the knowledge graph aggregates data from monolog, monoparts, and monolis?

# latency best practices

- monolog latency
    - data inference latency
        - should not run simple inference (eg iot anomaly detection) on cloud server
            - monolog must send data to cloud server
                - cloud streaming/ingress: costly/slow
        - run ml at the edge for simple inference
            - how?
                - on monolog directly (if they have microprocessors)
                - or on edge server in factory (if monolog only has microcontrollers)
                    - monolog sends data to factory server for inference
        - only stream critical/anomaly data to cloud server via mqtt
            - trigger agent
            - saved to db
            - reflected in ui
    - data storage latency
        - only storing sensor data for audit OR retraining ml models doing the simple inference
            - batch process during off-peak hours
            - dump everything to data lake

# workflow

- vendor workflow
    1. confirm order
    2. plan raw material sourcing
    3. purchase raw material
- factory workflow 4. plan production - evaluated -> boolean 5. receive raw material - boolean 6. prepare production - monitored by monolog 7. production - monitored by monolog 8. quality control - boolean 9. ship finished clothes - boolean

# tense ai

- goals
    1. help with pqcd optimization: palantir aip agent (reason using knowledge graph -> answer OR suggest actions)
        - knowledge graph
            - node properties should contain actions/apis the agent can use?
                - better than listing tools/apis in prompt/skill
                    - fewer tools for agent to choose from
                    - backend can write code so that if api changes, it is updated in db and thus reflected in graph
        - agent
            - when there's problem or when requested via chat or when there's a monolis workflow (must know all user workflows in monolist first), agent reasons
                - based on knowledge graph, what's the best solution
            - agent suggests actions
                - suggest api calls
                    - all existing apis should be included in the graph
                - human can approve to execute that api call
    2. supply chain visibility: data visualization
    3. prediction: run simulations on branched db using mathematical models
        - examples
            - impact analysis: what nodes are affected
            - what-if scenarios: what happens if we change this node
        - how palantir does it?
            - copy/branch the graph so that original graph is not affected
                - palantir uses native primitives like contour/vertex to branch the graph (and then simulate)
            - run simulation on branched graph
                - palantir uses mathematical engines (eg time series prediction models) to do simulations
- 5 ui channels
    - assistant: palantir aip agent
    - control: visualize monolog telemetries and agent tasks/status
    - location: visualize scm data
    - ontology: visualize knowledge graph
    - discuss: a way for agent to reason when it needs to suggest critical actions; and the visualization of that reasoning; but really the right way to reason?
- improvement pipeline
    - agent improvement
        - palantir's agent improvement pipeline
            - doesn't update llm weights
                - in fact, palantir is entirely model agnostic
            - focuses on context (graph) and instructions (prompt)
        - just use your own production-ready agent improvement pipeline
    - ml improvement
        - prediction: ml models have weights
            1. constantly compare prediction vs reality by calculating mean absolute error (mae)
            2. when mae over threshold, retrain model on latest historical data
        - operations research constraint solver: solvers have no weights
            - if agent uses solver and suggests an action but constantly gets rejected, the solver has wrong equations

# medallion architecture + palantir aip agent + palantir mathematical engines/models

- medallion architecture: popularized by databricks to create analytical db out of transactional db
- steps
    1. bronze
        - normalized, transactional
            - safe/fast for updates but slow for reads
        - palantir aip agent will suggest actions that runs on the bronze db
            - in fact, the agent will suggest backend api calls that only have access to the bronze db
    2. silver
        - denormalized/flat, analytical
        - palantir aip agent will do sql queries here when needed
            - eg data aggregation
                - eg total sales by region
                - expensive/slow on graph db because needs to touch many nodes and sum them
        - palantir mathematical engines/models will run here
    3. gold
        - knowledge graph
        - palantir aip agent will do graph queries here when needed
            - eg multi-hop reasoning
                - eg supply chain relationships, impact analysis
- pointers to note when going from bronze to silver
