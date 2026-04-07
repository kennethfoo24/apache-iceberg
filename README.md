# Apache Iceberg — Part 1: Prerequisites and Setup

## Prerequisites

- [Docker](https://docker.com) installed and running

## Setup

### 1. Create the Docker Compose file

In a blank directory, create a file called `docker-compose.yml` with the following content:
```yaml
version: "3"

# Notebook · Iceberg · Nessie Setup

services:

  # Nessie Catalog Server (In-Memory Store)
  nessie:
    image: projectnessie/nessie:latest
    container_name: nessie
    networks:
      - iceberg
    ports:
      - 19120:19120

  # MinIO Storage Server
  minio:
    image: minio/minio:latest
    container_name: minio
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=password
      - MINIO_DOMAIN=storage
      - MINIO_REGION_NAME=us-east-1
      - MINIO_REGION=us-east-1
    networks:
      - iceberg
    ports:
      - 9001:9001
      - 9000:9000
    command: ["server", "/data", "--console-address", ":9001"]

  # Dremio
  dremio:
    platform: linux/x86_64
    image: dremio/dremio-oss:latest
    container_name: dremio
    environment:
      - DREMIO_JAVA_SERVER_EXTRA_OPTS=-Dpaths.dist=file:///opt/dremio/data/dist
    networks:
      - iceberg
    ports:
      - 9047:9047
      - 31010:31010
      - 32010:32010

networks:
  iceberg:
```

### 2. Start the services

Open a terminal in the project directory and start each service:
```bash
docker-compose up dremio
docker-compose up minio
docker-compose up nessie
```

> You can run these in separate terminal windows, or add `-d` to run them in the background.

### 3. Configure MinIO

1. Go to [localhost:9000](http://localhost:9000) and log in with `admin` / `password`
2. Navigate to **Buckets** in the left menu and create a bucket named `warehouse`

### 4. Configure Dremio

1. Go to [localhost:9047](http://localhost:9047) and complete the initial account setup
2. From the dashboard, click **Add Source** and select **Nessie**
3. Under the **General** tab, set:
   - **Name:** `nessie`
   - **Endpoint URL:** `http://nessie:19120/api/v2`
   - **Authentication:** None
4. Under the **Storage** tab, set:
   - **Access key:** `admin`
   - **Secret key:** `password`
   - **Root path:** `/warehouse`
   - **Connection properties:**

     | Property | Value |
     |---|---|
     | `fs.s3a.path.style.access` | `true` |
     | `fs.s3a.endpoint` | `minio:9000` |
     | `dremio.s3.compat` | `true` |

5. Uncheck **Encrypt connection** (the local Nessie instance runs over HTTP)

## Part 2: Running Queries

Now that the source is connected, you can begin using Apache Iceberg from Dremio. All tables created here are also accessible by any tool that supports Nessie catalogs, including Apache Flink, Apache Spark, Presto, and Trino.

> **Note:** Query performance in this exercise reflects your local hardware in a single-node setup and is not representative of a deployed multi-node Dremio cluster.

### 1. Create a Table

In Dremio, click **SQL Runner** in the left menu and run the following query. Before running, click **Context** in the upper-right of the editor and set it to your Nessie source — otherwise you'll need to prefix table names with `nessie.` (e.g. `nessie.people`).
```sql
CREATE TABLE people (
  id INT,
  first_name VARCHAR,
  last_name VARCHAR,
  age INT
) PARTITION BY (truncate(1, last_name));
```

The `truncate(1, last_name)` clause is an Apache Iceberg **partition transform**, part of a feature called [Hidden Partitioning](https://iceberg.apache.org/docs/latest/partitioning/). It partitions the table by the first letter of `last_name` without creating an extra column.

You can verify the initial metadata was written by checking your `warehouse` bucket in MinIO at [localhost:9000](http://localhost:9000).

### 2. Insert Data
```sql
INSERT INTO people (id, first_name, last_name, age) VALUES
  (1,  'John',    'Doe',      28),
  (2,  'Jane',    'Smith',    34),
  (3,  'Alice',   'Johnson',  22),
  (4,  'Bob',     'Williams', 45),
  (5,  'Charlie', 'Brown',    30),
  (6,  'David',   'Jones',    25),
  (7,  'Eve',     'Garcia',   32),
  (8,  'Frank',   'Miller',   29),
  (9,  'Grace',   'Lee',      27),
  (10, 'Henry',   'Davis',    38);
```

After running this, check MinIO again — you'll see partition folders labeled by the first letter of each `last_name`.

### 3. Partition Evolution

Normally, changing a table's partitioning requires rewriting all existing data. Apache Iceberg's **Partition Evolution** feature lets you update partitioning rules without touching existing data. Add a second partition on `first_name`:
```sql
ALTER TABLE people ADD PARTITION FIELD truncate(1, first_name);
```

Now insert more records:
```sql
INSERT INTO people (id, first_name, last_name, age) VALUES
  (11, 'Isabella', 'Rodriguez', 40),
  (12, 'Jack',     'Martinez',  36),
  (13, 'Kylie',    'Hernandez', 24),
  (14, 'Leo',      'Lopez',     26),
  (15, 'Mia',      'Gonzalez',  35),
  (16, 'Nolan',    'Perez',     29),
  (17, 'Olivia',   'Wilson',    31),
  (18, 'Paul',     'Anderson',  33),
  (19, 'Quinn',    'Thomas',    27),
  (20, 'Rebecca',  'Taylor',    28);
```

Back in MinIO, you'll see that new records inside existing `last_name` partition folders are now further organized into `first_name` sub-folders. For example, `0_A/0_P/` contains the Parquet file for Paul Anderson.

### 4. Query and Save a View

Run a filtered query:
```sql
SELECT * FROM nessie.people WHERE last_name LIKE 'G%';
```

To save this for later, click the **Save As** dropdown in the upper-right corner and select **Save As View**, then save it into the Nessie catalog.

Nessie and Dremio Arctic catalogs can store views alongside tables, making them portable across any tool that connects to the catalog. On full Dremio deployments, you can also apply user/role access controls and column or row masking rules to views, making Dremio a central hub for data access and governance.

## Part 3: Branching and Merging

One of Nessie's standout features is the ability to **branch the catalog** — similar to branching in Git — so you can isolate changes before merging them into your main branch.

### 1. Create and Switch to a Branch
```sql
CREATE BRANCH ingest IN nessie;
USE BRANCH ingest;
```

### 2. Insert Data on the Branch
```sql
INSERT INTO nessie.people (id, first_name, last_name, age) VALUES
  (21, 'Samuel',  'Graham',  42),
  (22, 'Tina',    'Gray',    37),
  (23, 'Ursula',  'Green',   45),
  (24, 'Victor',  'Gibson',  29),
  (25, 'Wendy',   'Gates',   31),
  (26, 'Xavier',  'Graves',  28),
  (27, 'Yasmine', 'Gomez',   30),
  (28, 'Zane',    'Goodman', 33),
  (29, 'Aria',    'Guthrie', 25),
  (30, 'Brock',   'Garner',  40);
```

The `USE BRANCH` statement scopes the session to the `ingest` branch. The insert runs only against that branch — `main` is unaffected.

### 3. Verify Branch Isolation
```sql
USE BRANCH ingest;
SELECT * FROM nessie.people;   -- returns 30 records

USE BRANCH main;
SELECT * FROM nessie.people;   -- returns 20 records
```

The `ingest` branch sees all 30 records, while `main` still only sees the original 20, confirming the changes are fully isolated.

### 4. Merge into Main

Once you're satisfied with the changes, merge them into `main`:
```sql
MERGE BRANCH ingest INTO main IN nessie;
```

The additional 10 records are now visible to anyone querying the `main` branch (the default).

## Part 4: Working with CSV Files

### 1. Generate Sample Data

Head over to [mockaroo.com](https://mockaroo.com), use the default settings, and click **Generate Data** to download a CSV file with 1,000 random records.

### 2. Upload the CSV to Dremio

In the Dremio UI, navigate to the main **Datasets** section, click the **+** icon, and choose **Upload a File**.

> You could also access files directly from a connected storage source like MinIO.

In the formatting screen, make sure to select the correct line delimiter for your file and enable **Extract Field Names**, then click **Save**.

### 3. Fix Data Types

Querying the uploaded CSV will show all fields as text — CSV files don't carry schema information. Dremio makes it easy to fix this: click the menu in any column header to access utilities for converting types, aggregating, creating calculated fields, and more. To fix a numeric column:

1. Select **Convert Data Type** from the column header menu
2. Choose **Integer** on the next screen
3. Click **Apply**

### 4. Save as a View

Save your changes as a view in the Nessie catalog named `MOCK_DATA_CURATED`.

### 5. Convert to an Apache Iceberg Table

To improve query performance — especially on larger datasets — convert the CSV data into an Apache Iceberg table. There are two approaches:

**Option 1 — CTAS (Create Table As Select)**

Create a new Iceberg table directly from the curated view:
```sql
CREATE TABLE MOCK_DATA_CTAS AS SELECT * FROM nessie."MOCK_DATA_CURATED";
```

**Option 2 — COPY INTO**

First create an empty table with a matching schema, then copy the CSV data into it. This requires the CSV to be stored in a connected storage source like MinIO or S3, and also works with JSON files.
```sql
-- Create an empty table with a matching schema
CREATE TABLE MOCK_DATA_COPY AS SELECT * FROM nessie."MOCK_DATA_CURATED" LIMIT 0;

-- Copy the data from the CSV into the table
COPY INTO MOCK_DATA_COPY FROM '@minio' FILES ('MOCK_DATA.csv');
```

---

## Conclusion

Spin down all containers when you're done:
```bash
docker-compose down
```
