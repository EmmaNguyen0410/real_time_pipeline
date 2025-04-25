# Realtime data streaming 

This project is based on this project https://github.com/airscholar/e2e-data-engineering with some environment fixes and more detailed documentation. 

## Installation
Using pip:
1. apache-airflow
2. kafka-python
3. spark 
4. pyspark

## Run the program 
1. Pull the images and start containers 
```
docker compose up -d 
```
If you encounter error "exec /opt/airflow/script/entrypoint.sh: permission denied", it means that the entrypoint.sh script doesn't have executable permissions. You need to make sure that entrypoint.sh is executable on the host machine, so that when it's mounted into the container, it retains the executable permission. Run this: 
```
chmod +x ./script/entrypoint.sh
```

2. Check status 
- Confluent kakfa control center: http://localhost:9021/ 
- Spark: http://localhost:9090/
- Airflow: http://localhost:8080/

3. Run DAG 
Trigger DAG to retrieve data from API and sand to Kafka. You should see messages pushed to the users_created topic.

4. Submit job to Spark 
```
spark-submit \
  --packages com.datastax.spark:spark-cassandra-connector_2.12:3.4.1,\
  org.apache.spark:spark-sql-kafka-0-10_2.12:3.4.1,\
  org.apache.kafka:kafka-clients:3.5.1 \
  --master spark://localhost:7077 \
  spark_stream.py
```
5. Check the newly create table and data inserted in Cassandra
```
docker exec -it cassandra cqlsh -u cassandra -p cassandra localhost 9042
describe spark_streams.created_users;
select * from spark_streams.created_users;
```

## Highlights
1. Ivy resolution 

```
.config('spark.jars.packages', "com.datastax.spark:spark-cassandra-connector_2.13:3.4.1,"
                                           "org.apache.spark:spark-sql-kafka-0-10_2.13:3.4.1")
```
This command triggers Apache Ivy (a dependency manager) to resolve dependencies and put the resolved ones in ivy cache and jars. It provides a comma-separated list of Maven coordinates of jars to include on the driver and executor **classpaths**. The coordinates should be ```<groupId>:<artifactId>:<version>```. If spark.jars.ivySettings is given, artifacts will be resolved according to the configuration in the file, otherwise artifacts will be searched for in the local maven repo (~/.m2/respository), then maven central (https://repo1.maven...) and finally any additional remote repositories given by the command-line option --repositories.
packages. 

~/.ivy2/jars stores actual jars after resolution, which could be added to classpath in runtime. 
~/.ivy2/cache is your local repository, similiar to /.m2/repository, that store organized modules, metadata. 

2. Spark and Scala version

The whole Spark ecosystem must agree on Scala version to work properly:
- PySpark (mine is Scala 2.12.18 and Spark 3.5.5).
- Spark runtime in Docker and your host machine (mine is Scala 2.12.18 and Spark 3.5.5). 
- External JARs you pass in (e.g., Kafka) should be compiled with Scala 2.12 too.

**PySpark and Spark are different things**: Spark is the engine, Pyspark is a python API that runs over Spark. There are 5 languages which can be used with Spark (Java, Scala, R, SQL and Python), and each of them get decompiled into bytecode that runs on the JVM.

3. --packages option:
Spark downloads packages into ~/.ivy2/cache and ~/.ivy2/jars (local host).
At runtime, it stages the required jar files into a temp dir. This temporary location is used to upload the file to your Spark master or workers.

## Documentation 
https://spark.apache.org/docs/latest/configuration.html#viewing-spark-properties
https://spark.apache.org/docs/latest/submitting-applications.html#advanced-dependency-management
https://ant.apache.org/ivy/history/2.3.0/use/resolve.html
