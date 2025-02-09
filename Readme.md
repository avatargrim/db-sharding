To set up **Citus** on **Docker** for a 3-node cluster, you will need to:

1. **Create a Docker Compose file** to spin up 3 containers: one coordinator node and two worker nodes.
2. **Install Citus** on the coordinator and worker nodes.
3. **Set up the necessary PostgreSQL configurations** for Citus to work across the 3 nodes.

Here’s a step-by-step guide to get a 3-node Citus cluster up and running using Docker:

### Step 1: Docker Compose File
The easiest way to spin up a Citus cluster is by using **Docker Compose**, which allows you to define and manage multi-container applications.

Create a file named `docker-compose.yml`:

```yaml
version: '3.7'

services:
  citus-coordinator:
    image: citusdata/citus:latest
    environment:
      - POSTGRES_PASSWORD=postgres
    ports:
      - "5432:5432"
    volumes:
      - citus-coordinator-data:/var/lib/postgresql/data
      - ./data:/data
    networks:
      - citus-net

  citus-worker-1:
    image: citusdata/citus:latest
    environment:
      - POSTGRES_PASSWORD=postgres
    volumes:
      - citus-worker-1-data:/var/lib/postgresql/data
      - ./data:/data
    networks:
      - citus-net

  citus-worker-2:
    image: citusdata/citus:latest
    environment:
      - POSTGRES_PASSWORD=postgres
    volumes:
      - citus-worker-2-data:/var/lib/postgresql/data
      - ./data:/data
    networks:
      - citus-net

volumes:
  citus-coordinator-data:
  citus-worker-1-data:
  citus-worker-2-data:

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
SELECT name, cost_model, state, monthly_budget
FROM campaigns
WHERE company_id = 5
ORDER BY monthly_budget DESC
LIMIT 10;
```

We can also run a join query across multiple tables to see information about running campaigns which receive the most clicks and impressions.

```
SELECT campaigns.id, campaigns.name, campaigns.monthly_budget,
       sum(impressions_count) as total_impressions, sum(clicks_count) as total_clicks
FROM ads, campaigns
WHERE ads.company_id = campaigns.company_id
AND ads.campaign_id = campaigns.id
AND campaigns.company_id = 5
AND campaigns.state = 'running'
GROUP BY campaigns.id, campaigns.name, campaigns.monthly_budget
ORDER BY total_impressions, total_clicks;
```

