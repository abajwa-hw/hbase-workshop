### HBase workshop

#### Env setup on HDP 2.3.0.0-1754

- Setup a VM running HDP 2.3/Ambari 2.1 build 233 using [this repo file](http://s3.amazonaws.com/dev.hortonworks.com/ambari/centos6/2.x/BUILDS/2.1.0-244/ambaribn.repo)

- Download spark 1.3.1
```
export HDP_VER=`hdp-select status hadoop-client | sed 's/hadoop-client - \(.*\)/\1/'`
echo "export HDP_VER=$HDP_VER" >> ~/.bashrc

wget http://d3kbcqa49mib13.cloudfront.net/spark-1.3.1-bin-hadoop2.6.tgz
tar -xzvf spark-1.3.1-bin-hadoop2.6.tgz
echo "spark.driver.extraJavaOptions -Dhdp.version=$HDP_VER" >> spark-1.3.1-bin-hadoop2.6/conf/spark-defaults.conf
echo "spark.yarn.am.extraJavaOptions -Dhdp.version=$HDP_VER" >> spark-1.3.1-bin-hadoop2.6/conf/spark-defaults.conf
#copy hbase-site.xml
cp /etc/hbase/conf/hbase-site.xml spark-1.3.1-bin-hadoop2.6/conf/
export YARN_CONF_DIR=/etc/hadoop/conf
echo "export YARN_CONF_DIR=$YARN_CONF_DIR" >> ~/.bashrc
```

#### Setup Phoenix table and import data

- Remove Hbase maintenance mode

- Start HBASE
```
curl -u admin:admin -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start HBASE via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://localhost:8080/api/v1/clusters/Sandbox/services/HBASE
```

- Run python code to generate stock price csv
```
wget http://trading.cheno.net/wp-content/uploads/2011/12/google_intraday.py
#make changes if needed
#generate csv of prices
python google_intraday.py > prices.csv
```

- Create sql file to create phoenix table
```
vi ~/prices.sql
drop table if exists PRICES;
drop table if exists prices;

create table PRICES (
 SYMBOL varchar(10),
 DATE   varchar(10),
 TIME varchar(10),
 OPEN varchar(10),
 HIGH varchar(10),
 LOW    varchar(10),
 CLOSE     varchar(10),
 VOLUME varchar(30),
 CONSTRAINT pk PRIMARY KEY (VOLUME)
);
```

- Create phoenix table and populate with csv data
```
/usr/hdp/2*/phoenix/bin/psql.py sandbox.hortonworks.com:2181:/hbase-unsecure ~/prices.sql ~/prices.csv
```

- Connect to hbase via phoenix
```
/usr/hdp/*/phoenix/bin/sqlline.py sandbox.hortonworks.com:2181:/hbase-unsecure

```

- Run sample query
```
select * from prices order by DATE, TIME limit 20;
!q
```


------------------

#### Try examples from phoenix-spark 
- Try examples from https://github.com/apache/phoenix/tree/master/phoenix-spark using both spark local mode and yarn-client mode

- Start spark shell

```
unset HADOOP_CLASSPATH
export SPARK_CLASSPATH=/etc/hbase/conf:/usr/hdp/2.3.0.0-1754/hbase/lib/hbase-protocol.jar

#start spark shell and pass in relevant jars to classpath
#HDP 2.3 yarn-client mode

/root/spark-1.3.1-bin-hadoop2.6/bin/spark-shell --master yarn-client --driver-memory 512m --executor-memory 512m --conf hdp.version=$HDP_VER --jars \
/usr/hdp/2.3.0.0-1754/phoenix/phoenix-4.4.0.2.3.0.0-1754-client.jar 

#HDP 2.3 local mode

/root/spark-1.3.1-bin-hadoop2.6/bin/spark-shell  --driver-memory 512m --executor-memory 512m --conf hdp.version=$HDP_VER --jars \
/usr/hdp/2.3.0.0-1754/phoenix/phoenix-4.4.0.2.3.0.0-1754-client.jar 


```


- Load as an RDD, using a Zookeeper URL
```
import org.apache.phoenix.spark._ 
import org.apache.spark.rdd.RDD
val sqlCtx = new org.apache.spark.sql.SQLContext(sc)

val rdd: RDD[Map[String, AnyRef]] = sc.phoenixTableAsRDD(
  "PRICES", Seq("TIME", "SYMBOL"), zkUrl = Some("localhost:2181:/hbase-unsecure")
)
rdd.count()

```

- **works in spark local mode:**
```
res14: Long = 5760
```

- Error seen on HDP 2.3 in spark yarn-client mode (with spark 1.3.1):
```
2015-04-28 18:27:49,864 WARN  [task-result-getter-0] scheduler.TaskSetManager: Lost task 0.0 in stage 0.0 (TID 0, sandbox.hortonworks.com): java.lang.RuntimeException: java.sql.SQLException: No suitable driver found for jdbc:phoenix:sandbox.hortonworks.com
	at org.apache.phoenix.mapreduce.PhoenixInputFormat.getQueryPlan(PhoenixInputFormat.java:123)
	at org.apache.phoenix.mapreduce.PhoenixInputFormat.createRecordReader(PhoenixInputFormat.java:67)
	at org.apache.spark.rdd.NewHadoopRDD$$anon$1.<init>(NewHadoopRDD.scala:131)
	at org.apache.spark.rdd.NewHadoopRDD.compute(NewHadoopRDD.scala:104)
	at org.apache.spark.rdd.NewHadoopRDD.compute(NewHadoopRDD.scala:66)
	at org.apache.phoenix.spark.PhoenixRDD.compute(PhoenixRDD.scala:52)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:277)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:244)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:35)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:277)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:244)
	at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:61)
	at org.apache.spark.scheduler.Task.run(Task.scala:64)
	at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:203)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
Caused by: java.sql.SQLException: No suitable driver found for jdbc:phoenix:sandbox.hortonworks.com
	at java.sql.DriverManager.getConnection(DriverManager.java:689)
	at java.sql.DriverManager.getConnection(DriverManager.java:208)
	at org.apache.phoenix.mapreduce.util.ConnectionUtil.getConnection(ConnectionUtil.java:91)
	at org.apache.phoenix.mapreduce.util.ConnectionUtil.getInputConnection(ConnectionUtil.java:56)
	at org.apache.phoenix.mapreduce.PhoenixInputFormat.getQueryPlan(PhoenixInputFormat.java:110)
	... 16 more

```



--------------------

#### Other examples

- 1: Load table as a DataFrame using the Data Source API
```
import org.apache.spark.SparkContext
import org.apache.spark.sql.SQLContext
import org.apache.phoenix.spark._

val sqlCtx = new org.apache.spark.sql.SQLContext(sc)

val df = sqlContext.load(
  "org.apache.phoenix.spark", 
  Map("table" -> "PRICES", "zkUrl" -> "sandbox.hortonworks.com:2181")
)
```

- Error seen in both spark local/yarn-client mode:
```
[main] mapreduce.PhoenixInputFormat: UseSelectColumns=true, selectColumns=TIME,SYMBOL, selectColumnSet.size()=2, parsedColumns=TIME,SYMBOL
org.apache.phoenix.schema.TableNotFoundException: ERROR 1012 (42M03): Table undefined. tableName="PRICES"
```


- 2: Load table as a DataFrame directly using a Configuration object
```
import org.apache.hadoop.conf.Configuration
import org.apache.spark.SparkContext
import org.apache.spark.sql.SQLContext
import org.apache.phoenix.spark._

val sqlCtx = new org.apache.spark.sql.SQLContext(sc)

val configuration = new Configuration()
val df = sqlContext.phoenixTableAsDataFrame(
  "PRICES", Array("TIME", "SYMBOL"), conf = configuration
)
```
- Error seen in both spark local/yarn-client mode:
```
[main] mapreduce.PhoenixInputFormat: UseSelectColumns=true, selectColumns=TIME,SYMBOL, selectColumnSet.size()=2, parsedColumns=TIME,SYMBOL
org.apache.phoenix.schema.TableNotFoundException: ERROR 1012 (42M03): Table undefined. tableName="PRICES"
```



----------------

#### Run through Zeppelin

- Edit ~/incubator-zeppelin/conf/zeppelin-env.sh as below and restart Zeppelin
```
export JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk.x86_64
export SPARK_YARN_JAR=hdfs:///tmp/.zeppelin/zeppelin-spark-0.5.0-SNAPSHOT.jar
export MASTER=
export SPARK_HOME=/root/spark-1.3.1-bin-hadoop2.6
export HADOOP_CONF_DIR=/etc/hadoop/conf
export ZEPPELIN_PID_DIR=/var/run/zeppelin-notebook
export ZEPPELIN_JAVA_OPTS="-Dhdp.version=2.3.0.0-1754 -Dspark.files=/etc/hbase/conf/hbase-site.xml -Dspark.jars=/usr/hdp/2.3.0.0-1754/hbase/lib/hbase-protocol.jar,/usr/hdp/2.3.0.0-1754/phoenix/phoenix-4.4.0.2.3.0.0-1754-client.jar,/root/spark-1.3.1-bin-hadoop2.6/lib/spark-assembly-1.3.1-hadoop2.6.0.jar"
export ZEPPELIN_LOG_DIR=/var/log/zeppelin
export SPARK_CLASSPATH=/etc/hbase/conf:/usr/hdp/2.3.0.0-1754/hbase/lib/hbase-protocol.jar:/usr/hdp/2.3.0.0-1754/phoenix/phoenix-4.4.0.2.3.0.0-1754-client.jar
```

- Set Interpreter settings as below
![Image](../master/screenshots/zep-inter-local.png?raw=true)

- Run same examples as above

- Error seen
  - My guess is that SPARK_CLASSPATH (set in zeppelin-env.sh above) is not getting propagated to zeppelin
![Image](../master/screenshots/zep-error-local.png?raw=true)

#### Useful commands

- List and kill Spark jobs
```
yarn application --list
yarn application -kill <id>
```


#### Useful links

- https://github.com/simplymeasured/phoenix-spark
- http://opentsdb.net/overview.html
- http://www.slideshare.net/HBaseCon/case-studies-session-4a-35937605
- http://trading.cheno.net/wp-content/uploads/2011/12/google_intraday.py
- http://trading.cheno.net/downloading-google-intraday-historical-data-with-python/
