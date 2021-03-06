# presto-trial

## Trying presto with miniofs

Presto is a distributed SQL query engine for running interactive analytic queries against big data sources. Presto queries can run against HDFS, S3 and many other connectors. Connectors are the SPI exposed by Presto for data source providers to implement and this ia analogous to JDBC and drivers.

These are the steps to run presto on a mac to query data on S3 like FS. Minio is run to provide the S3 compatible FS. Presto also requires HIVE metastore to store metadata about the data on S3. 


# 1. Install docker & jdk8

# 2. Run mysql on docker
```
Use a path on your local filesystem instead of /data/mysql8 in the command below so that mysql can persist data to disk

docker run --rm --name mysql8 -e MYSQL_ROOT_PASSWORD=root -d -p 3306:3306 -v /data/mysql8:/var/lib/mysql mysql:8.0.28

```

# 3. Run minio on docker

```
Replace the /data/miniofs in the docker run cmd with a path on your local filesystem so that minio state is retained on restarting containers.

docker run --rm -d -p 19000:9000 -p 19001:9001 --name miniofs -v /data/miniofs:/data -e "MINIO_ROOT_USER=minioadmin" -e "MINIO_ROOT_PASSWORD=minioadmin" minio/minio server /data --console-address ":9001"

```
Navigate to http://localhost:19001 to access the minio admin page and login using the minioadmin/minioadmin credentials 

- Create a new user (secret/key as miniouser) to be used from presto/hive

- Create a bucket with name hive

- Browse the bucket and upload the data folder from this repo. You should now have s3://hive/data/csv/test.csv on minio and this is the file that we would be querying from presto


# 4. Download and unzip Apache Hadoop

```
Download hadoop 2.10.1 and unzip to a local directory
export HADOOP_HOME=/path/to/hadoop-2.10.1

```

# 5 Download and unzip Apache Hive

```
Download Hive 2.3.9 and unzip to a local directory
export HIVE_HOME=/path/to/apache-hive-2.3.9-bin
cd /path/to/apache-hive-2.3.9-bin
cp conf/hive-default.xml.template conf/hive-site.xml
cp conf/hive-env.sh.template conf/hive-env.sh

Append the below line to hive-env.sh
export HIVE_AUX_JARS_PATH=${HADOOP_HOME}/share/hadoop/tools/lib/hadoop-aws-2.10.1.jar:${HADOOP_HOME}/share/hadoop/tools/lib/aws-java-sdk-bundle-1.11.271.jar
export AWS_ACCESS_KEY_ID=miniouser
export AWS_SECRET_ACCESS_KEY=miniouser

Append the below xml snippet to hive-site.xml
  <property>
      <name>fs.s3a.endpoint</name>
      <description>AWS S3 endpoint to connect to.</description>
      <value>http://localhost:19000</value>
  </property>
  <property>
      <name>fs.s3a.access.key</name>
      <description>AWS access key ID.</description>
      <value>miniouser</value>
  </property>
  <property>
      <name>fs.s3a.secret.key</name>
      <description>AWS secret key.</description>
      <value>miniouser</value>
  </property>
  <property>
      <name>fs.s3a.path.style.access</name>
      <value>true</value>
      <description>Enable S3 path style access.</description>
  </property>


Amend the following properties in hive-site.xml (Use root password only in dev environment)
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://localhost:3306/hcatalog?createDatabaseIfNotExist=true</value>
    <description>
      JDBC connect string for a JDBC metastore.
      To use SSL to encrypt/authenticate the connection, provide database-specific SSL flag in the connection URL.
      For example, jdbc:postgresql://myhost/db?ssl=true for postgres database.
    </description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.cj.jdbc.Driver</value>
    <description>Driver class name for a JDBC metastore</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
    <description>Username to use against metastore database</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>root</value>
    <description>password to use against metastore database</description>
  </property>

mkdir -p hcatalog/var/log/
//Initialse the metastoreDB in mysql by using the below command 
bin/schematool --dbType mysql --initSchema
hcatalog/sbin/hcat_server.sh start  

The metastore should initialise successfully at this point 

//Metastore initialized successfully on port[9083].
```

# 6 Download and unzip presto
```
Download and unzip presto-server-0.271
Create an etc folder in the unzipped directory
Copy the contents of the prestconf to the etc directory
Create a data folder on your local filesystem. Update the path to this data folder to the node.properties in the etc directory
bin/launcher start
Download presto-cli to the presto-server-0.271 directory and make it executable and then run the following command
./presto-cli.jar --server localhost:8080 --catalog hive
```

# Query with Presto on presto-cli
Run the following commands on the prest-cli session

- CREATE SCHEMA testdata WITH (LOCATION = 's3a://hive/data');
- use testdata
- CREATE TABLE hive.testdata.employeedata (
    eid VARCHAR,
    ename VARCHAR,
    address VARCHAR,
    dept VARCHAR)
WITH (FORMAT = 'CSV',
    EXTERNAL_LOCATION = 's3a://hive/data/csv')
;
- select * from hive.testdata.employeedata; //this should now show the csv data

### Try running the following commands to create a parquet table and copy data from the above employeedata table
- CREATE SCHEMA testparq WITH (LOCATION = 's3a://hive/data/parq');
- user testparq
- CREATE TABLE hive.testparq.employeedata WITH (format = 'ORC', partitioned_by = ARRAY['dept']) AS SELECT eid, ename, address, dept FROM hive.testdata.employeedata;

You can now see that there are new files created in minio fs by going to the minio console and there would be multiple files dependening on number of partitions in your data.
