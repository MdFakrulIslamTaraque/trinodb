# Integrating Trino with PostgreSQL in Docker
- Index
    + [Prerequisites](#Prerequisites)
    + [Configuring Trino to Connect to PostgreSQL](#configuring-trino-to-connect-to-postgresql)
    + [Troubleshooting Network Connectivity](#troubleshooting-network-connectivity)
    + [Testing the Configuration](#testing-the-configuration)
    + [Conclusion](#conclusion)

# Prerequisites
1. **Docker**: Ensure Docker is installed and running.
2. **Trino and PostgreSQL Images**:
    - *Trino*: trinodb/trino:latest
    - *PostgreSQL*: postgres:13


# Configuring Trino to Connect to PostgreSQL

1. **Trino Catalog Configuration**

    Create a folder in your desired directory path where all your files for this deployment will be stored. I named this directory `trino_postgres`. Inside the `trino_postgres` folder, build a new folder named `catalog`. In this directory, you will create a properties file called `postgresql.properties` for your PostgreSQL properties. You are free to rename it whatever you choose with the same file extension. This is where the information from your database will be kept. To set up the PostgreSQL connector for Trino, create the `catalog/postgresql.properties` file:

    ```
    connector.name=postgresql
    connection-url=jdbc:postgresql://host:port/database
    connection-user=****
    connection-password=****
    ```
    
    Make sure PostgreSQL setup is done properly (in my case, PostgreSQL was installed as a Docker container, and I have made sure the host and port were aligned as per the `docker ps` result), and the database exists in the same connection. Remember you are connecting one database to trino. To connect multiple databases- you need to create properties file for each database.

2. **Docker Compose File**

    I configured trino service  using `docker-compose.yaml`. 

    ```
    version: '3.8'
    services:
    trino:
        image: trinodb/trino:latest
        ports:
        - "8069:8080"  # Mapping Trino's port
        volumes:
        - ./catalog:/etc/trino/catalog

    volumes:
    postgres-data:
    ```

    Here, I am mapping the trino containers' 8080 port to my local hosts' 8069 port. And for volume mapping, the root directory `category` is being mapped in the `/etc/trino/catalog` path of the trino container.
    
    **N.B:** In my case, PostgreSQL setup was already up. If you haven’t installed PostgreSQL, you may put this in the same docker-compose.yaml file. 

    Deploy the services:
    ```
    docker-compose up -d
    ```

3. Verify Running Containers

    Run `docker ps` to confirm both services are running:
    ```
    CONTAINER ID   IMAGE                    PORTS                               NAMES
    30e2c54243a9   trinodb/trino:latest   0.0.0.0:8069->8080/tcp                 trino_postgresql-trino-1
    e60469be8169   postgres:13            0.0.0.0:5432->5432/tcp                 postgres_airflow293
    ```

4. Verify Trino CLI Connection

    - Access the Trino container:
        ```
        $ docker exec -it 30e2 sh 
        sh-5.1$ trino
        ```

    - Confirm the PostgreSQL catalog is listed
        ```
        trino> SHOW CATALOGS; 
        ```
        Output will look like this including `postgresql`:
        
        ```
        Catalog   
        ------------
        postgresql 
        system     
        (2 rows)

        Query 20250110_062741_00003_kviyw, FINISHED, 1 node
        Splits: 11 total, 11 done (100.00%)
        0.30 [0 rows, 0B] [0 rows/s, 0B/s]
        ```

    Run the following commands to:
    
    - List all the schmas of PostgreSQL
        ```
        trino> show schemas from postgresql; 
        ```

    - Output will include all the schemas of postgresql database:
        ```
                Schema       
        --------------------
        information_schema 
        pg_catalog         
        public             
        (3 rows)

        Query 20250110_055920_00000_kviyw, FINISHED, 1 node
        Splits: 11 total, 11 done (100.00%)
        0.72 [3 rows, 49B] [4 rows/s, 69B/s]
        ```


    - List of available tables of any schema:
        ```
        trino> Show tables from postgresql.public; 
        ```

    If the third command fails with a connection error, follow the troubleshooting steps below.

# Troubleshooting Network Connectivity
1. Inspect Container Networks
    Check the networks of the PostgreSQL and Trino containers.
    
    - PostgreSQL :
        ```
        docker inspect -f '{{json .NetworkSettings.Networks }}' postgres_airflow293
        ```
        check the output key of the json result. In my case, output for postgresql was like:

        ```
        {
        "airflow_project_airflow": {
            ...........
            ...........
            "IPAddress": "172.19.0.2"
            ...........
            ...........
            }
        }
        ```

    - trino :
        ```
        docker inspect -f '{{json .NetworkSettings.Networks }}' trino_postgresql-trino-1
        ```
        check the output key of the json result. In my case, output for trino was like:

        ```
        {
            "trino_postgresql_default": {
                "IPAddress": "172.21.0.2"
                ...........
                ...........
                }
        }    
        ```
    The containers are on separate networks in my case. For potgresql, it was `airflow_project_airflow` and for trino it was `trino_postgresql_default` which was causing causing connectivity issues.
2. Connect Containers to the Same Network

    - Connect Trino to the PostgreSQL network:
        ```
        docker network connect airflow_project_airflow trino_postgresql-trino-1
        ```

    - Verify the connection:
        ```
        docker inspect -f '{{json .NetworkSettings.Networks }}' trino_postgresql-trino-1
        ```

    Ensure `airflow_project_airflow` is the key for the trino netwrok result.

3. Restart Trino
    
    Restart the Trino container to sync the changes:

    ```
    docker restart trino_postgresql-trino-1
    ```


# Testing the Configuration
Re-enter the Trino CLI again: 

```
$ docker exec -it 30e2 sh 
sh-5.1$ trino
```

- List all the schmas of PostgreSQL and now you will see the list of all schemas and tables inside any of the schemas. You can now access any table of your postgresql from the trinodb

```
trino> select * from postgresql.public.job limit 10; 
```

# Conclusion

1. Diagnosis: Inspected the network configurations of both containers and identify network settings.

2. Solution: If the containers are with different network setting, then connect the Trino container to the PostgreSQL network using the `docker network connect` command.

3. Validation: Restarted the Trino container and verify the integration using the Trino CLI commands like `SHOW SCHEMAS FROM postgresql;`.

This setup allows Trino to query PostgreSQL seamlessly in a Dockerized environment.


