### HBase workshop

#### Env setup


- OPTIONAL deploy maven/Zeppelin as Ambari services
```
cd /var/lib/ambari-server/resources/stacks/HDP/2.2/services/
git clone https://github.com/randerzander/maven-service
git clone https://github.com/abajwa-hw/zeppelin-stack.git   
service ambari-server restart
#now install mvn and zeppelin service via Ambari add service wizard
```

- Install maven
```
mkdir /usr/share/maven
cd /usr/share/maven
wget http://mirrors.koehn.com/apache/maven/maven-3/3.2.5/binaries/apache-maven-3.2.5-bin.tar.gz
tar xvzf apache-maven-3.2.5-bin.tar.gz
ln -s /usr/share/maven/apache-maven-3.2.5/ /usr/share/maven/latest
echo 'M2_HOME=/usr/share/maven/latest' >> ~/.bashrc
echo 'M2=$M2_HOME/bin' >> ~/.bashrc
echo 'PATH=$PATH:$M2' >> ~/.bashrc
export M2_HOME=/usr/share/maven/latest
export M2=$M2_HOME/bin
export PATH=$PATH:$M2
```

- Compile Phoenix from branch 4.x-HBase-0.98
```
cd
rm -rf phoenix
git clone https://github.com/apache/phoenix
cd phoenix
git checkout 4.x-HBase-0.98
mvn package -DskipTests -Dhadoop.profile=2
```

- Compile simplymeasured phoenix-spark
```
cd
rm -rf phoenix-spark
git clone https://github.com/simplymeasured/phoenix-spark
cd phoenix-spark
mvn package -DskipTests
```

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

- Other
```
cp /usr/hdp/2.2.4.2-2//phoenix/phoenix-server.jar /usr/hdp/2.2.4.2-2//phoenix/phoenix-server.jar.origali
cp /root/phoenix/phoenix-assembly/target/phoenix-4.4.0-HBase-0.98-SNAPSHOT-client.jar /usr/hdp/2.2.4.2-2//phoenix/phoenix-server.jar
cp /root/phoenix/phoenix-assembly/target/phoenix-4.4.0-HBase-0.98-SNAPSHOT-server.jar /usr/hdp/2.2.4.2-2//phoenix/phoenix-server.jar
```
-Restart Hbase

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

#/root/phoenix/bin/sqlline.py sandbox.hortonworks.com:2181:/hbase-unsecure

```

- Run sample query
```
select * from prices order by DATE, TIME limit 20;
!q
```


------------------

#### Try examples from phoenix-spark 
- Try examples from https://github.com/apache/phoenix/tree/master/phoenix-spark

- Start spark shell

```
unset HADOOP_CLASSPATH
export SPARK_CLASSPATH=/etc/hbase/conf:/usr/hdp/2.3.0.0-1754/hbase/lib/hbase-protocol.jar

#start spark shell in yarn-client mode and pass in phoenix-spark and phoenix-assembly jars to classpath
#HDP 2.3

/root/spark-1.3.1-bin-hadoop2.6/bin/spark-shell --master yarn-client --driver-memory 512m --executor-memory 512m --conf hdp.version=$HDP_VER --jars \
/usr/hdp/2.3.0.0-1754/hbase/lib/hbase-protocol.jar,/usr/hdp/2.3.0.0-1754/phoenix/lib/phoenix-spark-4.4.0.2.3.0.0-1754.jar,/usr/hdp/2.3.0.0-1754/phoenix/phoenix-4.4.0.2.3.0.0-1754-client.jar 

/root/spark-1.3.1-bin-hadoop2.6/bin/spark-shell --master yarn-client --driver-memory 512m --executor-memory 512m --conf hdp.version=$HDP_VER --jars \
/usr/hdp/2.3.0.0-1754/phoenix/phoenix-4.4.0.2.3.0.0-1754-client.jar 


/root/spark-1.3.1-bin-hadoop2.6/bin/spark-shell --master yarn-client --driver-memory 512m --executor-memory 512m --jars \
/usr/hdp/2.3.0.0-1754/phoenix/lib/phoenix-spark-4.4.0.2.3.0.0-1754.jar,/usr/hdp/2.3.0.0-1754/phoenix/lib/hbase-client.jar --conf hdp.version=$HDP_VER 

#HDP 2.2.4
/root/spark-1.3.1-bin-hadoop2.6/bin/spark-shell --master yarn-client --driver-memory 512m --executor-memory 512m --jars /root/phoenix/phoenix-spark/target/phoenix-spark-4.4.0-HBase-0.98-SNAPSHOT.jar,/root/phoenix/phoenix-assembly/target/phoenix-4.4.0-HBase-0.98-SNAPSHOT-client.jar --conf hdp.version=$HDP_VER 
/root/spark-1.3.1-bin-hadoop2.6/bin/spark-shell --master yarn-client --driver-memory 512m --executor-memory 512m --jars /usr/hdp/2.2.4.2-2/hbase/lib/hbase-protocol.jar,/root/phoenix/phoenix-spark/target/phoenix-spark-4.4.0-HBase-0.98-SNAPSHOT.jar,/root/phoenix/phoenix-assembly/target/phoenix-4.4.0-HBase-0.98-SNAPSHOT-client.jar --conf hdp.version=$HDP_VER

#extract tgz from /root/phoenix under /usr before this
/root/spark-1.3.1-bin-hadoop2.6/bin/spark-shell --master yarn-client --driver-memory 512m --executor-memory 512m --jars /usr/phoenix-4.4.0-HBase-0.98-SNAPSHOT/lib/hbase-protocol-0.98.12-hadoop2.jar,/root/phoenix/phoenix-spark/target/phoenix-spark-4.4.0-HBase-0.98-SNAPSHOT.jar,/root/phoenix/phoenix-assembly/target/phoenix-4.4.0-HBase-0.98-SNAPSHOT-client.jar --conf hdp.version=$HDP_VER
```



- Load as an RDD, using a Zookeeper URL
```
import org.apache.phoenix.spark._ 
import org.apache.spark.rdd.RDD
val sqlCtx = new org.apache.spark.sql.SQLContext(sc)

val rdd: RDD[Map[String, AnyRef]] = sc.phoenixTableAsRDD(
  "PRICES", Seq("TIME", "SYMBOL")
)
rdd.count()

val rdd: RDD[Map[String, AnyRef]] = sc.phoenixTableAsRDD(
  "PRICES", Seq("TIME", "SYMBOL"), zkUrl = Some("localhost:2181:/hbase-unsecure")
)
rdd.count()


val rdd: RDD[Map[String, AnyRef]] = sc.phoenixTableAsRDD(
  "PRICES", Seq("TIME", "SYMBOL"), zkUrl = Some("localhost:2181")
)
rdd.count()
```

- **works in spark local mode:**
```
res14: Long = 5760
```

- Error seen in spark yarn-client mode:
```
ERROR 2007 (INT09): Outdated jars. The following servers require an updated phoenix.jar to be put in the classpath of HBase: region=SYSTEM.CATALOG,,1430178920971.4f1ee8c72ac509956f0c4923dca5d8b7., hostname=sandbox.hortonworks.com,60020,1430178872563, seqNum=5

Caused by: java.sql.SQLException: ERROR 2006 (INT08): Incompatible jars detected between client and server. Ensure that phoenix.jar is put on the classpath of HBase in every region server: org.apache.hadoop.hbase.protobuf.generated.ZooKeeperProtos$MetaRegionServer.hasState()Z


```



--------------------

#### OPTIONAL: other examples

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

- Error seen in spark local mode:
```
[main] mapreduce.PhoenixInputFormat: UseSelectColumns=true, selectColumns=TIME,SYMBOL, selectColumnSet.size()=2, parsedColumns=TIME,SYMBOL
org.apache.phoenix.schema.TableNotFoundException: ERROR 1012 (42M03): Table undefined. tableName="PRICES"
```
- Error seen in spark yarn-client mode:
```
 ERROR 2007 (INT09): Outdated jars. The following servers require an updated phoenix.jar to be put in the classpath of HBase: region=SYSTEM.CATALOG,,1430178920971.4f1ee8c72ac509956f0c4923dca5d8b7., hostname=sandbox.hortonworks.com,60020,1430178872563, seqNum=5
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
- Error seen in spark local mode:
```
[main] mapreduce.PhoenixInputFormat: UseSelectColumns=true, selectColumns=TIME,SYMBOL, selectColumnSet.size()=2, parsedColumns=TIME,SYMBOL
org.apache.phoenix.schema.TableNotFoundException: ERROR 1012 (42M03): Table undefined. tableName="PRICES"
```
- Error seen in spark yarn-client mode:
```
ERROR 2007 (INT09): Outdated jars. The following servers require an updated phoenix.jar to be put in the classpath of HBase: region=SYSTEM.CATALOG,,1430178920971.4f1ee8c72ac509956f0c4923dca5d8b7., hostname=sandbox.hortonworks.com,60020,1430178872563, seqNum=5
```

#### OPTIONAL: Try simplymeasured example

- Try examples from https://github.com/simplymeasured/phoenix-spark

```
#Launch spark shell with phoenix-spark jar from /root/phoenix-spark
/usr/bin/spark-shell --master yarn-client --driver-memory 512m --executor-memory 512m --jars /root/phoenix-spark/target/phoenix-spark-0.0.3-SNAPSHOT.jar,/root/phoenix/phoenix-assembly/target/phoenix-4.4.0-HBase-0.98-SNAPSHOT-client.jar --conf hdp.version=2.2.4.2-2 

import com.simplymeasured.spark.PhoenixRDD
import org.apache.hadoop.conf.Configuration
val sqlCtx = new org.apache.spark.sql.SQLContext(sc)
conf = new Configuration()
val rdd = PhoenixRDD.NewPhoenixRDD(sc, "prices", Array("symbol", "date", "time","close","volume"), conf = conf) 

val count = rdd.count()
val schemaRDD = rdd.toSchemaRDD(sqlCtx)
```

- Error seen:
```
ERROR PhoenixInputFormat: Failed to get the query plan with error [null]
java.lang.RuntimeException: java.lang.NullPointerException
```

----------------

#### Run through Zeppelin

- Edit ~/incubator-zeppelin/conf/zeppelin-env.sh as below and restart Zeppelin
```
export JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk.x86_64
export SPARK_YARN_JAR=hdfs:///tmp/.zeppelin/zeppelin-spark-0.5.0-SNAPSHOT.jar
export MASTER=yarn-client
#export SPARK_HOME=/usr/hdp/current/spark-client/
export SPARK_HOME=/root/spark-1.3.1-bin-hadoop2.6/
export HADOOP_CONF_DIR=/etc/hadoop/conf
export ZEPPELIN_PID_DIR=/var/run/zeppelin-notebook
export ZEPPELIN_JAVA_OPTS="-Dhdp.version=2.2.4.2-2 -Dspark.jars=/root/phoenix/phoenix-spark/target/phoenix-spark-4.4.0-HBase-0.98-SNAPSHOT.jar,/root/phoenix/phoenix-assembly/target/phoenix-4.4.0-HBase-0.98-SNAPSHOT-client.jar"
export ZEPPELIN_LOG_DIR=/var/log/zeppelin
```

- Run same examples as above

- Error seen:
![Image](../master/screenshots/zeppelin-error.png?raw=true)

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
