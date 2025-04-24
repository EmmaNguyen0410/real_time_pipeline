

1. Install Apache Airflow
pip install apache-airflow
pip install kafka-python
pip install spark pyspark

In Maven repository search for spark cassandra connector to get the jar file for spark and cassandra 
com.datastax.spark:spark-cassandra-connector_<scala>_<version>
2. Permission denied 
The error you're seeing — exec /opt/airflow/script/entrypoint.sh: permission denied — means that the entrypoint.sh script doesn't have executable permissions. You need to make sure that entrypoint.sh is executable on the host machine, so that when it's mounted into the container, it retains the executable permission.
Download jar file and include it in your spark repository
venv/lib/pyspark/jars/
So even though the jar is available to Spark at runtime, Ivy still tries to download it during dependency resolution, and fails.
When you manually added all jars to lib/pyspark/jars and removed the .config("spark.jars.packages", ...) line (or didn’t depend on Ivy), Spark just launched and used the jars — no Ivy, no download, no problem.


```
chmod +x ./script/entrypoint.sh
```

```
this code "  webserver:
    image: apache/airflow:2.6.0-python3.9
    command: webserver
    entrypoint: ['/opt/airflow/script/entrypoint.sh']
    depends_on:
      - postgres
    environment:
      - LOAD_EX=n
      - EXECUTOR=Sequential
      - AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres:5432/airflow
      - AIRFLOW_WEBSERVER_SECRET_KEY=this_is_a_very_secured_key
    logging:
      options:
        max-size: 10m
        max-file: "3"
    volumes:
      - ./dags:/opt/airflow/dags
      - ./script/entrypoint.sh:/opt/airflow/script/entrypoint.sh
      - ./requirements.txt:/opt/airflow/requirements.txt
    ports:
      - "8080:8080"
    healthcheck:
      test: ['CMD-SHELL', "[ -f /opt/airflow/airflow-webserver.pid ]"]
      interval: 30s
      timeout: 30s
      retries: 3
    networks:
      - confluent" throws "exec /opt/airflow/script/entrypoint.sh: permission denied"
```

interval 30s and timeout 30s so it might be shorter than the actual amount of time needed to initialise this one. => wait then start the scheduler manually 

3. Executor Sequential means task runs in sequentially not parallel

4. Check the newly create table in Cassandra
```
docker exec -it cassandra cqlsh -u cassandra -p cassandra localhost 9042
describe spark_streams.created_users;
select * from spark_streams.created_users;
```

5. Ivy resolution 
  .config('spark.jars.packages', "com.datastax.spark:spark-cassandra-connector_2.13:3.4.1,"
                                           "org.apache.spark:spark-sql-kafka-0-10_2.13:3.4.1") \ 
triggers dependency resolution. commons-logging not found in the /Users/thaonguyen/.ivy2/cache
/Users/thaonguyen/.ivy2/jars, it will pull from the central. 

What's the diff between jars in lib and in ivy ???

6. Make sure that scala and spark compatible 
Spark version 3.5.5 and the scala is 2.12

The whole Spark ecosystem must agree on Scala version:

PySpark (Python frontend) expects a Spark runtime (backend) - JVM in Docker compiled with Scala 2.12.
Spark runtime in Docker should be compiled with Scala 2.12.
External JARs you pass in (e.g., Kafka) should be compiled with Scala 2.12 too.

Spark is the engine, Pyspark is a python API that runs over Spark. There are 5 languages which can be used with Spark (Java, Scala, R, SQL and Python), and each of them get decompiled into bytecode that runs on the JVM, so the performance difference is negligible. Pyspark and SQL are probably the most popular