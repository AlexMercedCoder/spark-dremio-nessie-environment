## Spark/Dremio Environment on your Laptop

Some Blogs Covering Similar Exiercises:
- [Postgres --> Iceberg/Dremio --> Superset Dashboard](https://www.dremio.com/blog/from-postgres-to-dashboards-with-dremio-and-apache-iceberg/)
- [SQLServer --> Iceberg/Dremio --> Superset Dashboard](https://www.dremio.com/blog/from-sqlserver-to-dashboards-with-dremio-and-apache-iceberg/)
- [MongoDB --> Iceberg/Dremio --> Superset Dashboard](https://www.dremio.com/blog/from-mongodb-to-dashboards-with-dremio-and-apache-iceberg/)
- [AWS Glue --> Dremio --> Superset Dashboard](https://www.dremio.com/blog/bi-dashboards-with-apache-iceberg-using-aws-glue-and-apache-superset/)
- [Running Graph QUeries on Iceberg Tables with Dremio & Puppygraph](https://www.dremio.com/blog/run-graph-queries-on-apache-iceberg-tables-with-dremio-puppygraph/)

#### With Nessie/Minio/Dremio

**Pre-Reqs:** Docker & Docker-Compose installed

### Step 1 - The docker-compose.yml

Fork & Clone this repo to your local environment. In this repo you'll find a `docker-compose.yml` file with the following:

```yml
version: "3"

services:
  # Nessie Catalog Server Using In-Memory Store
  nessie:
    image: projectnessie/nessie:latest
    container_name: nessie
    networks:
      spark-dremio:
    ports:
      - 19120:19120
  # Minio Storage Server
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
      spark-dremio:
    ports:
      - 9001:9001
      - 9000:9000
    command: ["server", "/data", "--console-address", ":9001"]
  # Dremio
  dremio:
    platform: linux/x86_64
    image: dremio/dremio-oss:latest
    ports:
      - 9047:9047
      - 31010:31010
      - 32010:32010
    container_name: dremio
    networks:
      spark-dremio:
  # Spark
  spark:
    platform: linux/x86_64
    image: alexmerced/spark35notebook:latest
    ports: 
      - 8080:8080  # Master Web UI
      - 7077:7077  # Master Port
      - 8888:8888  # Notebook
    environment:
      - AWS_REGION=us-east-1
      - AWS_ACCESS_KEY_ID=admin #minio username
      - AWS_SECRET_ACCESS_KEY=password #minio password

    container_name: spark
    networks:
      spark-dremio:
networks:
  spark-dremio:

```

The docker-compose.yml will define all the pieces you need in your lakehouse which will include:

- Nessie: Catalog with Git-Like functionality for Apache Iceberg tables

- Minio: S3 Compliant Object Storage software to act as our data lake storage.

- Dremio: Data Lakehouse platform to provide an easy to use and fast point of access for the Apache Iceberg tables stored on Nessie/Minio and other sources we connect.

- Spark: Spark 3.5 running as a single node with a Jupyter Notebook Server running on the same container.

`docker-compose.yml`

Open up a terminal in the same folder as this `docker-compose.yml` file and run the command

```shell
# latest versions of docker-desktop
docker compose up

# older versions
docker-compose up
```

This will create all the containers specified in our `docker-compose.yml` if you ever need to shut them down in another terminal in the same folder just run:

```shell
docker compose down
## or
docker-compose down
```

**NOTE:** To access the Jupyter Notebook server you need to look at the Docker-Compose output

### Step 2 - Setting up our storage bucket

- Open up an internet browser

- Visit minio at `http://localhost:9001`

- login with the username: `admin` and the password: `password` (these were specified in the `docker-compose.yml`)

- Create a bucket, let's call it `warehouse`

### Step 3 - Connect Dremio to Data Lake

#### Adding Minio as a storage source

- To add the object storage as a source, select "s3" as a source
  - Enter "admin" as your access key and "password" as your secret key
  - make sure that "compatibility mode" is checked
  - add the following connection properties
    - `fs.s3a.path.style.access` to `true`
    - `fs.s3a.endpoint` to `minio:9000`
   
#### Adding Nessie as a Metastore source

- Open up a new internet browser tab

- Visit Dremio at `http://localhost:9047`

- Fill out the form to create your account

- Then on the dashboard choose to connect a new source

- Select Nessie as your new source

There are two sections we need to fill out, the **general** and **storage** sections:

##### General (Connecting to Nessie Server)
- Set the name of the source to “nessie”
- Set the endpoint URL to “http://nessie:19120/api/v2”
Set the authentication to “none”

##### Storage Settings 
##### (So Dremio can read and write data files for Iceberg tables)

- For your access key, set “admin”
- For your secret key, set “password”
- Set root path to “/warehouse”
    Set the following connection properties:
    - `fs.s3a.path.style.access` to `true`
    - `fs.s3a.endpoint` to `minio:9000`
    - `dremio.s3.compat` to `true`
- Uncheck “encrypt connection” (since our local Nessie instance is running on http)

### Testing it Out

- Head to the SQL Runner on Dremio

- Run the following SQL statements

```
CREATE TABLE nessie.names (name varchar);
INSERT INTO nessie.names VALUES ('Gnarly the Narwhal');
SELECT * FROM nessie.names;
```

- Go explore you storage on minio, you should see all the Apache Iceberg data & metadata stored in your warehouse bucket.
