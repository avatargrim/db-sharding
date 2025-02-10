To set up **Citus** on **Docker** for a 3-node cluster, you will need to:

1. **Create a Docker Compose file** to spin up 3 containers: one coordinator node and two worker nodes.
2. **Install Citus** on the coordinator and worker nodes.
3. **Set up the necessary PostgreSQL configurations** for Citus to work across the 3 nodes.

Here’s a step-by-step guide to get a 3-node Citus cluster up and running using Docker:

### Step 1: Docker Compose File
The easiest way to spin up a Citus cluster is by using **Docker Compose**, which allows you to define and manage multi-container applications.

Create a file named `docker-compose.yml`:

```yaml
services:
  citus-coordinator:
    image: citusdata/citus:latest
    environment:
      - POSTGRES_PASSWORD=postgres
    ports:
      - "5432:5432"
    volumes:
      - ./pgdata/citus-coordinator-data:/var/lib/postgresql/data
      - ./data:/data
    networks:
      - citus-net

  citus-worker-1:
    image: citusdata/citus:latest
    environment:
      - POSTGRES_PASSWORD=postgres
    ports:
      - "54321:5432"
    volumes:
      - ./pgdata/citus-worker-1-data:/var/lib/postgresql/data
      - ./data:/data
    networks:
      - citus-net

  citus-worker-2:
    image: citusdata/citus:latest
    environment:
      - POSTGRES_PASSWORD=postgres
    ports:
      - "54322:5432"
    volumes:
      - ./pgdata/citus-worker-2-data:/var/lib/postgresql/data
      - ./data:/data
    networks:
      - citus-net

networks:
  citus-net:
    driver: bridge


```

### Step 2: Explanation of the Docker Compose File

- **citus-coordinator**: This service acts as the coordinator node, where the Citus extension will be installed and where client connections will be directed.
- **citus-worker-1 and citus-worker-2**: These are the worker nodes where the actual data will be stored and processed. Citus will manage the sharding across these nodes.
- **Volumes**: Data volumes are used to persist data between container restarts.
- **Networks**: A custom bridge network (`citus-net`) ensures the containers can communicate with each other.

### Step 3: Launch the Cluster

1. In the directory containing your `docker-compose.yml` file, run:

   ```bash
   docker compose up -d
   ```

   This will download the Citus images and start the coordinator and worker nodes as Docker containers.

2. To check if the containers are running, use:

   ```bash
   docker compose ps
   ```

### Step 4: Initialize the Citus Cluster

Once the containers are up, you'll need to set up the Citus worker nodes by telling the coordinator to add the worker nodes.

1. Connect to the coordinator node:

   ```bash
   docker compose exec -it citus-coordinator psql -U postgres
   ```

2. Inside the `psql` shell, run the following SQL commands to add the worker nodes to the Citus cluster:

   ```sql
   -- Add the first worker node
   SELECT citus_add_node('citus-worker-1', 5432);

   -- Add the second worker node
   SELECT citus_add_node('citus-worker-2', 5432);
   ```

   This will tell the coordinator node about the worker nodes.

### Step 5: Verify the Setup

You can verify if the Citus cluster has been properly set up by running:

```sql
SELECT * FROM pg_catalog.pg_dist_node;
```

This should show the coordinator node and the two worker nodes.

### Step 6: Test Sharding

Now you can test if sharding is working. For example, create a distributed table:

```sql
-- Create a table on the coordinator node
CREATE TABLE companies (
  id bigserial PRIMARY KEY,
  name text NOT NULL,
  image_url text,
  created_at timestamp without time zone NOT NULL,
  updated_at timestamp without time zone NOT NULL
);

CREATE TABLE campaigns (
  id bigserial,       -- was: PRIMARY KEY
  company_id bigint REFERENCES companies (id),
  name text NOT NULL,
  cost_model text NOT NULL,
  state text NOT NULL,
  monthly_budget bigint,
  blacklisted_site_urls text[],
  created_at timestamp without time zone NOT NULL,
  updated_at timestamp without time zone NOT NULL,
  PRIMARY KEY (company_id, id) -- added
);

CREATE TABLE ads (
  id bigserial,       -- was: PRIMARY KEY
  company_id bigint,  -- added
  campaign_id bigint, -- was: REFERENCES campaigns (id)
  name text NOT NULL,
  image_url text,
  target_url text,
  impressions_count bigint DEFAULT 0,
  clicks_count bigint DEFAULT 0,
  created_at timestamp without time zone NOT NULL,
  updated_at timestamp without time zone NOT NULL,
  PRIMARY KEY (company_id, id),         -- added
  FOREIGN KEY (company_id, campaign_id) -- added
    REFERENCES campaigns (company_id, id)
);

CREATE TABLE clicks (
  id bigserial,        -- was: PRIMARY KEY
  company_id bigint,   -- added
  ad_id bigint,        -- was: REFERENCES ads (id),
  clicked_at timestamp without time zone NOT NULL,
  site_url text NOT NULL,
  cost_per_click_usd numeric(20,10),
  user_ip inet NOT NULL,
  user_data jsonb NOT NULL,
  PRIMARY KEY (company_id, id),      -- added
  FOREIGN KEY (company_id, ad_id)    -- added
    REFERENCES ads (company_id, id)
);

CREATE TABLE impressions (
  id bigserial,         -- was: PRIMARY KEY
  company_id bigint,    -- added
  ad_id bigint,         -- was: REFERENCES ads (id),
  seen_at timestamp without time zone NOT NULL,
  site_url text NOT NULL,
  cost_per_impression_usd numeric(20,10),
  user_ip inet NOT NULL,
  user_data jsonb NOT NULL,
  PRIMARY KEY (company_id, id),       -- added
  FOREIGN KEY (company_id, ad_id)     -- added
    REFERENCES ads (company_id, id)
);

-- Distribute the table across the worker nodes based on the 'id' column
SELECT create_distributed_table('companies',   'id');
SELECT create_distributed_table('campaigns',   'company_id');
SELECT create_distributed_table('ads',         'company_id');
SELECT create_distributed_table('clicks',      'company_id');
SELECT create_distributed_table('impressions', 'company_id');
```

You can now insert data, and Citus will distribute it across the worker nodes.

### Add Data
Then, you can go ahead and load the data we downloaded into the tables using the standard PostgreSQL COPY command. Please make sure that you specify the correct file path if you downloaded the file to some other location.

```
-- copy one by one line
\copy companies from '/data/companies.csv' with csv

\copy campaigns from '/data/campaigns.csv' with csv

\copy ads from '/data/ads.csv' with csv

\copy clicks from '/data/clicks.csv' with csv

\copy impressions from '/data/impressions.csv' with csv
```

### Step 7: Scale as Needed

If you want to add more worker nodes, you can simply add another service to the `docker-compose.yml` file and run `docker-compose up -d` again. After that, add the new worker node to the Citus cluster by running the `citus_add_node()` function in the coordinator node.

---

### Troubleshooting

- **Container Connectivity**: If the worker nodes can’t communicate with the coordinator or each other, check the network settings and make sure they are all connected to the same `citus-net` network.
- **PostgreSQL Logs**: You can check the logs of any container by using:

   ```bash
   docker logs <container-id>
   ```

- **Docker Compose Logs**: You can view the logs of all services by running:

   ```bash
   docker-compose logs
   ```

---

That’s it! You now have a 3-node Citus cluster running on Docker, with one coordinator and two worker nodes.



## Running queries

Now that we have loaded data into the tables, let’s go ahead and run some queries. Citus supports standard INSERT, UPDATE and DELETE commands for inserting and modifying rows in a distributed table which is the typical way of interaction for a user-facing application.

For example, you can insert a new company by running:

```
INSERT INTO companies VALUES (5000, 'New Company', 'https://randomurl/image.png', now(), now());
```
If you want to double the budget for all the campaigns of a company, you can run an UPDATE command:
```
UPDATE campaigns
SET monthly_budget = monthly_budget*2
WHERE company_id = 5;
```

Another example of such an operation would be to run transactions which span multiple tables. Let’s say you want to delete a campaign and all its associated ads, you could do it atomically by running:

```
BEGIN;
DELETE FROM campaigns WHERE id = 46 AND company_id = 5;
DELETE FROM ads WHERE campaign_id = 46 AND company_id = 5;
COMMIT;
```

Each statement in a transactions causes roundtrips between the coordinator and workers in multi-node Citus. For multi-tenant workloads, it’s more efficient to run transactions in distributed functions. The efficiency gains become more apparent for larger transactions, but we can use the small transaction above as an example.


Besides transactional operations, you can also run analytics queries using standard SQL. One interesting query for a company to run would be to see details about its campaigns with maximum budget.

```
EXPLAIN ANALYSE SELECT name, cost_model, state, monthly_budget
FROM campaigns
WHERE company_id = 5
ORDER BY monthly_budget DESC
LIMIT 10;
```

Result 

```
                                                                      QUERY PLAN                                                                      
------------------------------------------------------------------------------------------------------------------------------------------------------
 Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=0 width=0) (actual time=143.468..143.505 rows=9 loops=1)
   Task Count: 1
   Tuple data received from nodes: 425 bytes
   Tasks Shown: All
   ->  Task
         Tuple data received from node: 425 bytes
         Node: host=citus-worker-1 port=5432 dbname=postgres
         ->  Limit  (cost=9.51..9.52 rows=2 width=104) (actual time=1.767..1.812 rows=9 loops=1)
               ->  Sort  (cost=9.51..9.52 rows=2 width=104) (actual time=1.705..1.706 rows=9 loops=1)
                     Sort Key: monthly_budget DESC
                     Sort Method: quicksort  Memory: 26kB
                     ->  Bitmap Heap Scan on campaigns_102046 campaigns  (cost=4.16..9.50 rows=2 width=104) (actual time=0.666..0.671 rows=9 loops=1)
                           Recheck Cond: (company_id = 5)
                           Heap Blocks: exact=1
                           ->  Bitmap Index Scan on campaigns_pkey_102046  (cost=0.00..4.16 rows=2 width=0) (actual time=0.208..0.208 rows=9 loops=1)
                                 Index Cond: (company_id = 5)
             Planning Time: 7.624 ms
             Execution Time: 2.458 ms
 Planning Time: 17.079 ms
 Execution Time: 145.013 ms
(20 rows)
```


We can also run a join query across multiple tables to see information about running campaigns which receive the most clicks and impressions.

```
EXPLAIN ANALYSE SELECT campaigns.id, campaigns.name, campaigns.monthly_budget,
       sum(impressions_count) as total_impressions, sum(clicks_count) as total_clicks
FROM ads, campaigns
WHERE ads.company_id = campaigns.company_id
AND ads.campaign_id = campaigns.id
AND campaigns.company_id = 5
AND campaigns.state = 'running'
GROUP BY campaigns.id, campaigns.name, campaigns.monthly_budget
ORDER BY total_impressions, total_clicks;
```

Result
```
                                                                            QUERY PLAN                                                                            
------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Custom Scan (Citus Adaptive)  (cost=0.00..0.00 rows=0 width=0) (actual time=5.553..5.557 rows=3 loops=1)
   Task Count: 1
   Tuple data received from nodes: 145 bytes
   Tasks Shown: All
   ->  Task
         Tuple data received from node: 145 bytes
         Node: host=citus-worker-1 port=5432 dbname=postgres
         ->  Sort  (cost=14.94..14.94 rows=1 width=112) (actual time=0.272..0.324 rows=3 loops=1)
               Sort Key: (sum(ads.impressions_count)), (sum(ads.clicks_count))
               Sort Method: quicksort  Memory: 25kB
               ->  HashAggregate  (cost=14.91..14.93 rows=1 width=112) (actual time=0.243..0.296 rows=3 loops=1)
                     Group Key: campaigns.id, campaigns.name, campaigns.monthly_budget
                     Batches: 1  Memory Usage: 24kB
                     ->  Hash Join  (cost=9.52..14.46 rows=36 width=64) (actual time=0.138..0.234 rows=25 loops=1)
                           Hash Cond: (ads.campaign_id = campaigns.id)
                           ->  Seq Scan on ads_102078 ads  (cost=0.00..4.75 rows=73 width=32) (actual time=0.039..0.067 rows=73 loops=1)
                                 Filter: (company_id = 5)
                                 Rows Removed by Filter: 67
                           ->  Hash  (cost=9.51..9.51 rows=1 width=56) (actual time=0.056..0.105 rows=3 loops=1)
                                 Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                 ->  Bitmap Heap Scan on campaigns_102046 campaigns  (cost=4.16..9.51 rows=1 width=56) (actual time=0.031..0.034 rows=3 loops=1)
                                       Recheck Cond: (company_id = 5)
                                       Filter: (state = 'running'::text)
                                       Rows Removed by Filter: 6
                                       Heap Blocks: exact=1
                                       ->  Bitmap Index Scan on campaigns_pkey_102046  (cost=0.00..4.16 rows=2 width=0) (actual time=0.014..0.014 rows=9 loops=1)
                                             Index Cond: (company_id = 5)
             Planning Time: 0.424 ms
             Execution Time: 0.454 ms
 Planning Time: 1.557 ms
 Execution Time: 5.599 ms
(31 rows)
```


We can see where each schema resides with the following query:

```
  select nodename,nodeport, table_name, pg_size_pretty(sum(shard_size))
    from citus_shards
group by nodename,nodeport, table_name;
```
Result 

```
    nodename    | nodeport | table_name  | pg_size_pretty 
----------------+----------+-------------+----------------
 citus-worker-2 |     5432 | campaigns   | 496 kB
 citus-worker-1 |     5432 | clicks      | 5656 kB
 citus-worker-1 |     5432 | ads         | 1272 kB
 citus-worker-2 |     5432 | impressions | 19 MB
 citus-worker-2 |     5432 | companies   | 496 kB
 citus-worker-1 |     5432 | impressions | 22 MB
 citus-worker-1 |     5432 | companies   | 496 kB
 citus-worker-1 |     5432 | campaigns   | 496 kB
 citus-worker-2 |     5432 | ads         | 1160 kB
 citus-worker-2 |     5432 | clicks      | 4872 kB
(10 rows)
```

# Useful Diagnostic Queries

## Finding which shard contains data for a specific tenant

To find the worker node holding the data for store id=4, ask for the placement of rows whose distribution column has value 4:
```
SELECT shardid, shardstate, shardlength, nodename, nodeport, placementid
  FROM pg_dist_placement AS placement,
       pg_dist_node AS node
 WHERE placement.groupid = node.groupid
   AND node.noderole = 'primary'
   AND shardid = (
     SELECT get_shard_id_for_distribution_column('companies', 4)
   );
```

Result 

```
 shardid | shardstate | shardlength |    nodename    | nodeport | placementid 
---------+------------+-------------+----------------+----------+-------------
  102016 |          1 |           0 | citus-worker-1 |     5432 |           9
(1 row)
```

## Finding the distribution column for a table

Each distributed table in Citus has a “distribution column.” For more information about what this is and how it works, see Distributed Data Modeling. There are many situations where it is important to know which column it is. Some operations require joining or filtering on the distribution column, and you may encounter error messages with hints like, “add a filter to the distribution column.”

The pg_dist_* tables on the coordinator node contain diverse metadata about the distributed database. In particular pg_dist_partition holds information about the distribution column (formerly called partition column) for each table. You can use a convenient utility function to look up the distribution column name from the low-level details in the metadata. Here’s an example and its output:

```
-- get distribution column name for products table

SELECT column_to_column_name(logicalrelid, partkey) AS dist_col_name
  FROM pg_dist_partition
 WHERE logicalrelid='campaigns'::regclass;
 
```

Result 

```
 dist_col_name 
---------------
 company_id
(1 row)

```

## Querying the size of all distributed tables

This query gets a list of the sizes for each distributed table plus the size of their indices.

```
SELECT table_name, table_size
  FROM citus_tables;
Example output:
```

Result
```
 table_name  | table_size 
-------------+------------
 ads         | 2432 kB
 campaigns   | 992 kB
 clicks      | 10 MB
 companies   | 992 kB
 impressions | 41 MB
(5 rows)
```
Identifying unused indices

This query will run across all worker nodes and identify any unused indexes for a given distributed table, designated here with the **companies**:


```
SELECT *
FROM run_command_on_shards('clicks', $cmd$
  SELECT array_agg(a) as infos
  FROM (
    SELECT (
      schemaname || '.' || relname || '##' || indexrelname || '##'
                 || pg_size_pretty(pg_relation_size(i.indexrelid))::text
                 || '##' || idx_scan::text
    ) AS a
    FROM  pg_stat_user_indexes ui
    JOIN  pg_index i
    ON    ui.indexrelid = i.indexrelid
    WHERE NOT indisunique
    AND   idx_scan < 50
    AND   pg_relation_size(relid) > 5 * 8192
    AND   (schemaname || '.' || relname)::regclass = '%s'::regclass
    ORDER BY
      pg_relation_size(i.indexrelid) / NULLIF(idx_scan, 0) DESC nulls first,
      pg_relation_size(i.indexrelid) DESC
  ) sub
$cmd$);

```

Result with non index.

```
 shardid | success | result 
---------+---------+--------
  102008 | t       | 
  102009 | t       | 
  102010 | t       | 
  102011 | t       | 
  102012 | t       | 
  102013 | t       | 
  102014 | t       | 
  102015 | t       | 
  102016 | t       | 
  102017 | t       | 
  102018 | t       | 
  102019 | t       | 
  102020 | t       | 
  102021 | t       | 
  102022 | t       | 
  102023 | t       | 
  102024 | t       | 
  102025 | t       | 
  102026 | t       | 
  102027 | t       | 
  102028 | t       | 
  102029 | t       | 
  102030 | t       | 
  102031 | t       | 
  102032 | t       | 
  102033 | t       | 
  102034 | t       | 
  102035 | t       | 
  102036 | t       | 
  102037 | t       | 
  102038 | t       | 
  102039 | t       | 
(32 rows)
```

Result with indexes

```
shardid | success |                              result                              
---------+---------+------------------------------------------------------------------
  102104 | t       | {"public.clicks_102104##clicks_company_id_idx_102104##32 kB##0"}
  102105 | t       | {"public.clicks_102105##clicks_company_id_idx_102105##32 kB##0"}
  102106 | t       | {"public.clicks_102106##clicks_company_id_idx_102106##16 kB##0"}
  102107 | t       | {"public.clicks_102107##clicks_company_id_idx_102107##32 kB##0"}
  102108 | t       | {"public.clicks_102108##clicks_company_id_idx_102108##16 kB##0"}
  102109 | t       | {"public.clicks_102109##clicks_company_id_idx_102109##16 kB##0"}
  102110 | t       | {"public.clicks_102110##clicks_company_id_idx_102110##32 kB##0"}
  102111 | t       | {"public.clicks_102111##clicks_company_id_idx_102111##16 kB##0"}
  102112 | t       | {"public.clicks_102112##clicks_company_id_idx_102112##40 kB##0"}
  102113 | t       | {"public.clicks_102113##clicks_company_id_idx_102113##16 kB##0"}
  102114 | t       | {"public.clicks_102114##clicks_company_id_idx_102114##16 kB##0"}
  102115 | t       | 
  102116 | t       | {"public.clicks_102116##clicks_company_id_idx_102116##16 kB##0"}
  102117 | t       | {"public.clicks_102117##clicks_company_id_idx_102117##32 kB##0"}
  102118 | t       | {"public.clicks_102118##clicks_company_id_idx_102118##40 kB##0"}
  102119 | t       | {"public.clicks_102119##clicks_company_id_idx_102119##16 kB##0"}
  102120 | t       | {"public.clicks_102120##clicks_company_id_idx_102120##32 kB##0"}
  102121 | t       | {"public.clicks_102121##clicks_company_id_idx_102121##16 kB##0"}
  102122 | t       | 
  102123 | t       | {"public.clicks_102123##clicks_company_id_idx_102123##16 kB##0"}
  102124 | t       | {"public.clicks_102124##clicks_company_id_idx_102124##48 kB##0"}
  102125 | t       | {"public.clicks_102125##clicks_company_id_idx_102125##32 kB##0"}
  102126 | t       | {"public.clicks_102126##clicks_company_id_idx_102126##32 kB##0"}
  102127 | t       | {"public.clicks_102127##clicks_company_id_idx_102127##32 kB##0"}
  102128 | t       | {"public.clicks_102128##clicks_company_id_idx_102128##40 kB##0"}
  102129 | t       | {"public.clicks_102129##clicks_company_id_idx_102129##16 kB##0"}
  102130 | t       | {"public.clicks_102130##clicks_company_id_idx_102130##16 kB##0"}
  102131 | t       | {"public.clicks_102131##clicks_company_id_idx_102131##32 kB##0"}
  102132 | t       | {"public.clicks_102132##clicks_company_id_idx_102132##32 kB##0"}
  102133 | t       | {"public.clicks_102133##clicks_company_id_idx_102133##32 kB##0"}
  102134 | t       | {"public.clicks_102134##clicks_company_id_idx_102134##40 kB##0"}
  102135 | t       | {"public.clicks_102135##clicks_company_id_idx_102135##40 kB##0"}
(32 rows)

```


Monitoring client connection count

This query will give you the connection count by each type that are open on the coordinator:

```
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
Exxample output:
```


```
 state  | count 
--------+-------
        |     6
 active |     1
(2 rows)
```

## Viewing system queries

### Active queries

The citus_stat_activity view shows which queries are currently executing. You can filter to find the actively executing ones, along with the process ID of their backend:
```
SELECT global_pid, query, state
  FROM citus_stat_activity
 WHERE state != 'idle';

```
### Why are queries waiting

We can also query to see the most common reasons that non-idle queries that are waiting. For an explanation of the reasons, check the PostgreSQL documentation.
```
SELECT wait_event || ':' || wait_event_type AS type, count(*) AS number_of_occurences
  FROM pg_stat_activity
 WHERE state != 'idle'
GROUP BY wait_event, wait_event_type
ORDER BY number_of_occurences DESC;
```

Example output when running pg_sleep in a separate query concurrently:
```
 type | number_of_occurences 
------+----------------------
      |                    1
(1 row)
```
### Index hit rate

This query will provide you with your index hit rate across all nodes. Index hit rate is useful in determining how often indices are used when querying:
```
-- on coordinator
SELECT 100 * (sum(idx_blks_hit) - sum(idx_blks_read)) / sum(idx_blks_hit) AS index_hit_rate
  FROM pg_statio_user_indexes;
```
-- on workers
SELECT nodename, result as index_hit_rate
FROM run_command_on_workers($cmd$
  SELECT 100 * (sum(idx_blks_hit) - sum(idx_blks_read)) / sum(idx_blks_hit) AS index_hit_rate
    FROM pg_statio_user_indexes;
$cmd$);
```

Example output:
```
   index_hit_rate    
---------------------
 50.0000000000000000
(1 row)
```


### Cache hit rate

Most applications typically access a small fraction of their total data at once. Postgres keeps frequently accessed data in memory to avoid slow reads from disk. You can see statistics about it in the pg_statio_user_tables view.

An important measurement is what percentage of data comes from the memory cache vs the disk in your workload:
```
-- on coordinator
SELECT
  sum(heap_blks_read) AS heap_read,
  sum(heap_blks_hit)  AS heap_hit,
  100 * sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_rate
FROM
  pg_statio_user_tables;

-- on workers
SELECT nodename, result as cache_hit_rate
FROM run_command_on_workers($cmd$
  SELECT
    100 * sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_rate
  FROM
    pg_statio_user_tables;
$cmd$);
```

Example output:

```
    nodename    |    cache_hit_rate    
----------------+----------------------
 citus-worker-1 | 100.0000000000000000
 citus-worker-2 | 100.0000000000000000
(2 rows)
```

If you find yourself with a ratio significantly lower than 99%, then you likely want to consider increasing the cache available to your database