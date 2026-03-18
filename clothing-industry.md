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
        - erp: used by office workers to track schedule/money/etc
            - what problems does it solve?
                - centralizes data storage across different vendor departments/teams
                    - used to be stored on papers or separate databases
            - what problems remain?
                - still required humans to map raw data to erp
                    - may need excel to store the intermediate data
                - not connected to mes
                    - need humans to communicate via gmail
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
    - examples
        - in vietnam, etc

# sije's solutions
- monolog: replaces mes
    - automatic
        - tracks production status in factories
        - uploads data to monolis
- monolis: replaces erp?
    - what exactly does it do?
- tense ai: graphrag chatbot
    - helps vendor understand existing data
        - user can chat
    - predicts future state
        - automatic and on-demand
    - identify problems based on data
        - and suggest decisions

# todo
- figure out what monolis and tense ai must do
- determine new data pipeline for source data (or current db) -> knowledge graph
    - do we stream or batch?
    - do we create a new/clean db to act as an intermediate btw old db & graph db?