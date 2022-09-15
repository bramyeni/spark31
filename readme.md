# Spark 3.1 Cluster with Jupyter Lab and Notebook Classic on a container

Spark 3.1 either single node or Cluster with Jupyter Lab and Notebook classic

The codes can be found here: https://github.com/bramyeni/bramscripts/tree/main/microsoft/spark31

## Assumptions

Master         : 192.168.0.158
Worker Node #1 : 192.168.0.178
Worker Node #2 : 192.168.0.188
Persistent disk: /opt/hdfsvol


## Dockerfile
i have created a dockerfile with based image debian for Spark Master Node and Alpine for Spark Worker nodes
NOTE:
The only reason why i use Arch Linux for Master node is because Musl-libc doesnt work properly with spark running Java 8 or 11, it works running on java other than those 2 versions. The funny thing is , it runs nicely for Spark worker nodes (it may require series of stress test to ensure it doesnt have a problem, i havent performed any stress test yet, so please keep an eye out)
The issue also happen to Hadoop cluster (see my other repository for hadoop docker container)

## How to Build the image for Spark Master Node
Get a dockerfile from this repo, let say you download it into /opt

<pre>
cd /opt
docker build --tag spark -f ./Dockerfile-spark31-arch /tmp
</pre>

## How to Build the image for Spark Worker Node
<pre>
cd /opt
docker build --tag spark -f ./Dockerfile-spark31-worker /tmp
</pre>

## Deploy Spark Master Node 
### Running spark master and expose the following ports
<pre>
docker run -it --name spark3 -p 4040-4048:4040-4048 -p 8888-8999:8888-8999 -v /opt/spark/hdfsvol:/opt/hdfsvol -v /opt/spark/etc:/opt/spark/etc spark3
</pre>

### Running spark master and jupyter notebook and lab with jupyter token : goodstuff
<pre>
docker run -it --name spark3 -p 4040-4048:4040-4048 -p 8888-8999:8888-8999 -e JUPYTER_TOKEN="goodstuff" -v /opt/spark/hdfsvol:/opt/hdfsvol -v /opt/spark/etc:/opt/spark/etc spark3 
</pre>

## Running spark master with Network=Host (it automatically exposes all ports as it uses host networking)
<pre>
docker run -it --name spark3 --network host -v /opt/spark/hdfsvol:/opt/hdfsvol -v /opt/spark/etc:/opt/spark/etc spark3
</pre>

### Running spark Master and automatically runs spark worker nodes
<pre>
docker run -it --name spark3 -p 4040-4048:4040-4048 -p 8888-8999:8888-8999 -v /opt/spark/hdfsvol:/opt/hdfsvol -v /opt/spark/etc:/opt/spark/etc -e SPARK_MASTER="yes" -e SPARK_WORKERS="172.17.0.5,172.17.0.6" -e JUPYTER_TOKEN="sparkisawesome" spark3
</pre>

NOTE: remember that spark worker nodes must be running first (if they are containers then those containers must run sshd, so this spark master will be able to connect and performs the trusted connections (copy public key, start it up automatically through ssh command)

## Connect to other containers that are available here (PostgreSQL, MySQL, etc) 
If you have previously configured the above databases , hadoop or any other containers, then jupyter notebook will be able to connect to it

## The Jupyter notebook/lab in this container is equipped with bunch of python3 modules and features (including delta table)
### Sample codes

### Delta table pyspark session
<pre>
from pyspark.sql import SparkSession
spark = SparkSession \
    .builder \
    .appName("myapp") \
    .master("local[*]") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()

from delta.tables import *
from pyspark.sql.functions import *

spark.sql("CREATE TABLE events ( \
  date DATE,   \
  eventId STRING,    \
  eventType STRING,    \
  data STRING)   \
USING delta  \
PARTITIONED BY (date)")

</pre>

### Connect to PostgreSQL DB
<pre>
#Notebook 1
%load_ext sql

#Notebook 2
%sql postgresql://postgres:password@192.168.0.158/postgres

#Notebook 3
%%sql
create table bla (no int)
   
#Notebook 4
%%sql
insert into bla values (1)

#Notebook 5
%%sql
select * from bla;

### Writing delta table to hadoop

<pre>
x = spark.read.format("delta").load("hdfs://192.168.0.158:9000/")
x.show()
#delete delta table
from delta.tables import *
from pyspark.sql.functions import *

deltaTable = DeltaTable.forPath(spark, "hdfs://192.168.0.158:9000/bram")

deltaTable.delete()  

</pre>

### Loading SQL extension
<pre>
%load_ext sql
</pre>


### Connect to MySQL DB

<pre>
%sql  mysql+pymysql://admin:password@192.168.0.178/mydb
</pre>


