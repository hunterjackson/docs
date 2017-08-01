Kafka Connect Cassandra CDC
===========================

[TOC]

Kafka Connect Cassandra is a Source Connector for reading Change Data Capture from Cassandra and write the mutations to Kafka.


-----------------------------------------------------------------------
Why
---
We already provide a Kafka Connect Cassandra Source and people might ask us why another source. The main reason is performance. We have noticed our users with large  data in a Cassandra table (column family)
the performance drops. And this is not on the connector but rather how long it takes to pull the records. We have worked around the issue by introducing a limit on the records queried on Cassandra but even
so we are not that much better.

With Apache Cassandra 3.0 the support for CDC(Change Data Capture) has been added and this source implementation is making use of that to capture the changes made to your column families.


-----------------------------------------------------------------------
Cassandra CDC
-------------
Change data capture is designed to capture insert, update, and delete activity applied to tables(column families), and to make the details of the changes available in an easily consumed format.

Cassandra CDC logging is configured per table, with limits on the amount of disk space to consume for storing the CDC logs. CDC logs ``use the same binary format as the commit log``.
Cassandra tables can be created or altered with a table property to use CDC logging.

``` sourcecode:: sql
    CREATE TABLE foo (a int, b text, PRIMARY KEY(a)) WITH cdc=true;

    ALTER TABLE foo WITH cdc=true;

    ALTER TABLE foo WITH cdc=false;
```

CDC logging must be enabled in the cassandra.yaml file to begin logging. You should make sure your yaml file has the following:
```.. sourcecode:: javascript
   cdc_enabled: true
```
You can enable the CDC logging on per node basis.

The Kafka Connect Source consumes the CDC log information and pushes it to Kafka before it deletes it.

> **Note**
Upon flushing the memtable to disk, CommitLogSegments containing data for CDC-enabled tables are moved to the configured cdc_raw directory.
Once the disk space limit is reached, writes to CDC enabled tables will be rejected until space is freed.``

**Four CDC settings are configured in the cassandra.yaml**
cdc_enabled
:	Enables/Disables CDC logging per node

cdc_raw_directory
:	The directory where the CDC log is stored. Default locations:
          Package installations(default): /$CASSANDRA_HOME/cdc_raw
          Tarball installations: install_location/data/cdc_raw

    cdc_total_space_in_mb
    :	Total space available for storing CDC data
      (Default: 4096MB and 1/8th of the total space of the drive where the cdc_raw_directory resides.)
      If space gets above this value, Cassandra will throw WriteTimeoutException on Mutations including tables with CDC enabled.

    cdc_free_space_check_interval_ms
    :  Default: 250 ms
      When the cdc_raw limit is hit and the Consumer is either running behind or experiencing backpressure, this interval is checked to see if any new space for cdc-tracked tables has been made available.

    >**Important**
After changing properties in the cassandra.yaml file, you must restart the node for the changes to take effect. ``

-----------------------------------------------------------------------
Prerequisites
-------------

-  Cassandra **3.0.9+**
-  Confluent **3.2+**
-  Java **1.8**
-  Scala **2.11**

-----------------------------------------------------------------------
Setup
-----

Before we can do anything, including the QuickStart we need to install Cassandra and the Confluent platform.

-----------------------------------------------------------------------
####Cassandra Setup

First download and install Cassandra if you don't have a compatible
cluster available.

``` sourcecode:: bash

    #make a folder for cassandra
    mkdir cassandra

    #Download Cassandra
    wget http://apache.mirror.anlx.net/cassandra/3.11.0/apache-cassandra-3.11.0-bin.tar.gz

    #extract archive to cassandra folder
    tar xvf apache-cassandra-3.11.0-bin.tar.gz -C cassandra --strip-components=1

    #enable the CDC in the yaml configuration
    sed -i -- 's/cdc_enabled: false/cdc_enabled: true/g'  conf/cassandra.yaml

    #Start Cassandra
    cd cassandra
    sudo sh /bin/cassandra
```

>**NOTE**
There can be only one instance of Apache Cassandra node per machine for the connector to run properly.
All nodes should have the same path for the cdc_raw folder

-----------------------------------------------------------------------
####Confluent Setup

Follow the instructions :ref:`here <install>`.


-----------------------------------------------------------------------
####Source Connector

The Cassandra CDC Source connector will read the commit log mutations from the CDC logs and will push them to the target topic.
> **Note** The messages sent to Kafka are in AVRO format.

The record pushed to Kafka populates both the key and the value part. Key will contain the metadata of the change while the value will contain the actual data change.

-----------------------------------------------------------------------
##### Record Key
The key data structure follows this layout:

``` .. sourcecode:: javascript
    {
        "keyspace":  //Cassandra Keyspace name
        "table"   :  //The Cassandra Column Family name
        "changeType": //The type of change in Cassandra
         "keys": {
            "key1":
            "key2":
            ..
         }
         "timestamp" : //the timestamp of when the change was made in Cassandra
         "deleted_columns": //which columns have been deleted. We will expand on the details
    }
```

Based on the mutation information we can identify the following types of changes:

INSERT
:	a record has been inserted/a record columns have been updated. There is no real solution for identifying an UPDATE unless the connector keeps track of all the keys seen.

```.. sourcecode:: sql
    INSERT INTO $keyspace.orders (id, created, product, qty, price) VALUES (1, now(), 'OP-DAX-P-20150201-95.7', 100, 94.2)
```
DELETE
:	an entire record has been deleted (tombstoned)
``` .. sourcecode:: sql
    DELETE FROM datamountaineer.orders where id = 1
```

DELETE_COLUMN
:	Specific columns have been deleted (non PK columns).
```
.. sourcecode:: sql
    DELETE product FROM datamountaineer.orders where id = 1
    DELETE name.firstname FROM datamountaineer.users WHERE id=62c36092-82a1-3a00-93d1-46196ee77204;
```
In this case the ``deleted_columns`` entry will contain "product"/"name.firstname". If more than one column is deleted we will retain that information.

-----------------------------------------------------------------------
##### Value Key

The Kafka message value part contains the actual mutation data. Apart from the primary keys columns all the other columns have an optional schema in avro. The reason for that is because one can set the values on a subset of them during a CQL insert/update. In the QuickStart section we make use of the ``users`` table.  The value AVRO schema associated with it looks like this

``` .. sourcecode:: javascript
{
  "type" : "record",
  "name" : "users",
  "fields" : [ {
    "name" : "name",
    "type" : [ "null", {
      "type" : "record",
      "name" : "fullname",
      "fields" : [ {
        "name" : "firstname",
        "type" : [ "null", "string" ],
        "default" : null
      }, {
        "name" : "lastname",
        "type" : [ "null", "string" ],
        "default" : null
      } ],
      "connect.name" : "fullname"
    } ],
    "default" : null
  }, {
    "name" : "addresses",
    "type" : [ "null", {
      "type" : "array",
      "items" : {
        "type" : "record",
        "name" : "MapEntry",
        "namespace" : "io.confluent.connect.avro",
        "fields" : [ {
          "name" : "key",
          "type" : [ "null", "string" ],
          "default" : null
        }, {
          "name" : "value",
          "type" : [ "null", {
            "type" : "record",
            "name" : "address",
            "namespace" : "",
            "fields" : [ {
              "name" : "street",
              "type" : [ "null", "string" ],
              "default" : null
            }, {
              "name" : "city",
              "type" : [ "null", "string" ],
              "default" : null
            }, {
              "name" : "zip_code",
              "type" : [ "null", "int" ],
              "default" : null
            }, {
              "name" : "phones",
              "type" : [ "null", {
                "type" : "array",
                "items" : [ "null", "string" ]
              } ],
              "default" : null
            } ],
            "connect.name" : "address"
          } ],
          "default" : null
        } ]
      }
    } ],
    "default" : null
  }, {
    "name" : "direct_reports",
    "type" : [ "null", {
      "type" : "array",
      "items" : [ "null", "fullname" ]
    } ],
    "default" : null
  }, {
    "name" : "id",
    "type" : [ "null", "string" ],
    "default" : null
  }, {
    "name" : "other_reports",
    "type" : [ "null", {
      "type" : "array",
      "items" : [ "null", "fullname" ]
    } ],
    "default" : null
  } ],
  "connect.name" : "users"
}
```

And a json representation of the actual value looks like this:
``` .. sourcecode:: javascript
{
	"id" : "UUID-String",
	"name": {
		"firstname":"String"
		"lastname":"String"
	},
	"direct_reports":[
		{
			"firstname":"String"
			"lastname":"String"
		},
		...
	],
	"other_reports":[
		{
			"firstname":"String"
			"lastname":"String"
		},
		...
	],
	"addresses" : {
		"home" : {
			  "street": "String",
			  "city": "String",
			  "zip_code": "int",
			  "phones": [
				  "+33 ...",
				  "+33 ..."
			  ]
		},
		"work" : {
			  "street": "String",
			  "city": "String",
			  "zip_code": "int",
			  "phones": [
				  "+33 ...",
				  "+33 ..."
			  ]
		},
		...
	}
}
```

-----------------------------------------------------------------------
####Data Types

The Source connector needs to map Apache Cassandra types to Kafka Connect Schema types. For the ones not so savy with connect here is the list of supported Connect Types:
```
INT8,
INT16
INT32
INT64
FLOAT32
FLOAT64
BOOLEAN
STRING
BYTES
ARRAY
MAP
STRUCT
```
Along these primitive types there are the logical types for :
```
Date
Decimal
Time
Timestamp
```
As a result for most Apache Cassandra Types we have an equivalent type for Connect Schema (Remember a Connect Source Record will be marshalled as AVRO when sent to Kafka).

| CQL Type         | Connect Data Type                 |
| :--------------- | :---------------------------------|
|AsciiType         | OPTIONAL STRING                   |
|LongType          | OPTIONAL INT64                    |
|BytesType         | OPTIONAL BYTES                    |
|BooleanType       | OPTIONAL BOOLEAN                  |
|CounterColumnType | OPTIONAL INT64                    |
|SimpleDateType    | OPTIONAL Kafka Connect Date       |
|DoubleType        | OPTIONAL FLOAT64                  |
|DecimalType       | OPTIONAL Kafka Connect Decimal    |
|DurationType      | OPTIONAL STRING                   |
|EmptyType         | OPTIONAL STRING                   |
|FloatType         | OPTIONAL FLOAT32                  |
|InetAddressType   | OPTIONAL STRING                   |
|Int32Type         | OPTIONAL INT32                    |
|ShortType         | OPTIONAL INT16                    |
|UTF8Type          | OPTIONAL STRING                   |
|TimeType          | OPTIONAL KAFKA CONNECT Time       |
|TimestampType     | OPTIONAL KAFKA CONNECT Timestamp  |
|TimeUUIDType      | OPTIONAL STRING                   |
|ByteType          | OPTIONAL INT8                     |
|UUIDType          | OPTIONAL STRING                   |
|IntegerType       | OPTIONAL INT32                    |
|ListType          | OPTIONAL ARRAY of the inner type  |
|MapType           | OPTIONAL MAP of the inner types   |
|SetType           | OPTIONAL ARRAY of the inner type  |
|UserType          | OPTIONAL STRUCT for the user type |

Please note we default to String for the these CQL types: DurationType,InetAddressType,TimeUUIDType,UUIDType


-----------------------------------------------------------------------
#### How does it work

It is expected that Kafka Connect worker will run on the same node as the Apache Cassandra node.
>**Important*
>Only one Apache Cassandra Node should run per machine to have the Connector work properly
>The **cdc_raw** folder location should be the same on all nodes running Apache Cassandra Node
>There should be only one Connector Worker instance per machine. Any more won't have any effect

Cassandra supports  a masterless "ring" architecture. Each of the node in the Cassandra ring cluster will be responsible for storing the table records. The partition key hash and number of rings in the cluster determines the where each record is stored. (we leave aside replication from this discussion).

Upon flushing the memtable to disk, all the commit log segments containing data for CDC-enabled tables are moved to the configured cdc_raw directory. It is only at this point the connector will pick up the changes.

>**Important**
>Changes in Cassandra are not picked up immediately. The memtables need to be flushed for the CDC commit logs to be available.
>You can use nodetool to flush the tables

Once a file lands in the CDC folder the Connector will pick it up and read the mutations. A CDC file can contain mutations for more than one table. Each mutation for the subscribed tables will be translated into a Kafka Connect Source Record which will be sent by the Connect framework to the topic configured in the connector properties.

The Connect source will process the files in the order they were created and one by one. This ensures the change sequence is retained. Once the records have been pushed to Kafka the CDC file is deleted.

The connector will only be able to read the mutations for the subscribed tables. Via configuration you can express which tables to consider and what topic should receive those mutations information.

```.. sourcecode:: sql
    INSERT INTO ordersTopic SELECT * FROM datamountaineer.orders
```

>**Important**
>Enabling CDC on a new table means you need to restart the connector for the changes to be picked up. The connector is driven by the configurations and not by the list of all the tables with CDC enabled. (Might be a feature, change to do)

Below you can find a flow diagram describing the process mentioned above.

```flow
st=>start: Cassandra Client/Cqlsh
e=>end:
op1=>operation: Cassandra Ring Node | current
cond=>condition: SST tables flushed?
io=>inputoutput: Commit Log Dropped in CDC_RAW Folder
op2=>operation: Kafka Connect CDC
sub1=>subroutine: File Watcher
op3=>operation: Kafka

st->op1->cond
cond(yes)->io
io->sub1->op2
op2->op3
```


-----------------------------------------------------------------------

###Source Connector QuickStart

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and updating the configuration. We have developed
a Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.


Once you have installed and started Cassandra create a table to capture the mutations. We will use a bit more complex column family structure to show case what we support so far.

Let's start the cql shell tool to create our table
```.. sourcecode:: bash

    $./bin/cqlsh
    Connected to Test Cluster at 127.0.0.1:9042.
	[cqlsh 5.0.1 | Cassandra 3.11.0 | CQL spec 3.4.4 | Native protocol v4]
	Use HELP for help.
	cqlsh>
```
Now let's create a keyspace and a users column family(table)

```
.. sourcecode:: sql

CREATE KEYSPACE datamountaineer WITH REPLICATION = {'class' : 'SimpleStrategy', 'replication_factor' : 3};

CREATE TYPE datamountaineer.address(
  street text,
  city text,
  zip_code int,
  phones set<text>
);

CREATE TYPE datamountaineer.fullname (
  firstname text,
  lastname text
);

CREATE TABLE datamountaineer.users(
	id uuid PRIMARY KEY,
	name  fullname,
	direct_reports set<frozen <fullname>>,
	other_reports list<frozen <fullname>>,
	addresses map<text, frozen <address>>);
ALTER TABLE datamountaineer.users WITH cdc=true;
```

Starting the Connector (Distributed)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running the ``start-connect.sh`` script in the ``bin`` folder.

```.. sourcecode:: bash
    ./bin/start-connect.sh
```
Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for Cassandra.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to connect to the Rest API of Kafka Connect of your container.
```
.. sourcecode:: bash
$export KAFKA_CONNECT_REST="http://myserver:myport"
```
```.. sourcecode:: bash
$./bin/cli.sh create cassandra-source-orders < conf/cassandra-source-cdc.properties
    #Connector `cassandra-source-cdc`:
name=cassandra-source-cdc       connector.class=com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector
connect.cassandra.source.kcql=INSERT INTO users-topic SELECT * FROM users connect.cassandra.contact.points=localhost

```

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.
```.. sourcecode:: bash
    #check for running connectors with the CLI
$./bin/cli.sh ps
   cassandra-cdc-source
```
//TODO

```
.. sourcecode:: bash

    ➜  $CONFLUENT_HOME/bin/kafka-avro-console-consumer \
    --zookeeper localhost:2181 \
    --topic users-topic \
    --from-beginning
```



-----------------------------------------------------------------------
Features
--------

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Both connectors support **K** afka **C** onnect **Q** uery **L** anguage found here
`GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_ allows for routing and mapping using
a SQL like syntax, consolidating typically features in to one configuration option.

..  sourcecode:: sql

    INSERT INTO <topic> SELECT * FROM <KEYSPACE>.<TABLE>

    #Select all mutations for datamountaineer.orders
    INSERT INTO ordersTopic SELECT * FROM datamountaineer.orders

    #while KSQL allows for fields (column) addressing the CDC Source will not use them. It pushes the entire Cassandra mutation information on the topic



-----------------------------------------------------------------------
Configurations
--------------
Here is a full list of configuration entries the connector knows about.

| Name             | Description                 | Data Type |  Optional|Default|
| :--------------- | :---------------------------------| :------| :----| ---------:|
|connect.cassandra.contact.points|Contact points (hosts) in Cassandra cluster.|string|no| |
|connect.cassandra.port| Cassandra Node Client connection port|int|yes|9042|
|connect.cassandra.username|Username to connect to Cassandra with if ``connect.cassandra.authentication.mode`` is set to *username_password*|string|yes||
|connect.cassandra.password|Password to connect to Cassandra with if ``connect.cassandra.authentication.mode`` is set to *username_password*.|string|yes||
|connect.cassandra.ssl.enabled|Enables SSL communication against SSL enable Cassandra cluster.|boolean|yes|false|
|connect.cassandra.trust.store.password|Password for truststore.|string|yes||
|connect.cassandra.key.store.path|Path to truststore.|string|yes||
|connect.cassandra.key.store.password|Password for key store.|string|yes||
|connect.cassandra.ssl.client.cert.auth|Path to keystore.|string|yes||
|connect.cassandra.kcql|Kafka connect query language expression. Allows for expressive table to topic routing. It describes which CDC tables are monitored and the target Kafka topic for the Cassandra CDC information.|string|no||
|connect.cassandra.cdc.path.url| The location of the Cassandra Yaml file in URL format:file://{THE_PATH_TO_THE_YAML_FILE}. The connector reads the file to get the cdc folder but also to set the internal of Cassandra API allowing it to read the CDC files|string|no||
|connect.cassandra.cdc.file.watch.interval|The delay time in milliseconds before the connector checks for Cassandra CDC files.  We poll the CDC folder for new files.|long|yes|2000|
|connect.cassandra.cdc.mutation.queue.size|The maximum number of Cassandra mutation to buffer. As it reads from the Cassandra CDC files the mutations are buffered before  they  are handed over to Kafka Connect when the framework calls for new records.|int|yes|1000000|
|connect.cassandra.cdc.enable.delete.while.read | The worker CDC thread will read a CDC file and then check if any of the processed files are ready to be deleted (that means the records have been sent to Kafka). Rather than waiting for a read to complete we can delete the files while reading a CDC file.Default value is true. You can disable it for faster reads by setting the value to false.| boolean |yes|false|
|connect.cassandra.cdc.single.instance.port|Kafka Connect framework doesn't allow yet configuration where you are saying runnning only one task per worker. If you allocate more tasks than workers then some workers will spin up more tasks. With Cassandra nodes we want one worker and one task - not more. To ensure this we allow the first task to grab a port - subsequent calls to open the port will fail thus not allowing multiple instance running at once|int|yes|64101|
|connect.cassandra.cdc.decimal.scale|When reading the column family metadata we don't have details about the decimal scale.|int|yes|18|



----------------------------------------------------------------------
Deployment Guidelines
---------------------

TODO

----------------------------------------------------------------------

TroubleShooting
---------------

TODO
