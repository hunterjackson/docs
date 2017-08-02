Kafka Connect Cassandra CDC
===========================
.. note::

    This is still beta version


Kafka Connect Cassandra is a Source Connector for reading Change Data Capture from Cassandra and write the mutations to Kafka.


Why
---

We already provide a Kafka Connect Cassandra Source and people might ask us why another source. The main reason is performance. We have noticed our users with large  data in a Cassandra table (column family)
the performance drops. And this is not on the connector but rather how long it takes to pull the records. We have worked around the issue by introducing a limit on the records queried on Cassandra but even
so we are not that much better.

With Apache Cassandra 3.0 the support for CDC(Change Data Capture) has been added and this source implementation is making use of that to capture the changes made to your column families.


Cassandra CDC
-------------

Change data capture is designed to capture insert, update, and delete activity applied to tables(column families), and to make the details of the changes available in an easily consumed format.

Cassandra CDC logging is configured per table, with limits on the amount of disk space to consume for storing the CDC logs. CDC logs ``use the same binary format as the commit log``.
Cassandra tables can be created or altered with a table property to use CDC logging.

.. sourcecode:: sql

    CREATE TABLE foo (a int, b text, PRIMARY KEY(a)) WITH cdc=true;

    ALTER TABLE foo WITH cdc=true;

    ALTER TABLE foo WITH cdc=false;

CDC logging must be enabled in the cassandra.yaml file to begin logging. You should make sure your yaml file has the following:

.. sourcecode:: bash

   cdc_enabled: true

You can enable the CDC logging on per node basis.

The Kafka Connect Source consumes the CDC log information and pushes it to Kafka before it deletes it.

.. warning::

    Upon flushing the memtable to disk, CommitLogSegments containing data for CDC-enabled tables are moved to the
    configured cdc_raw directory. Once the disk space limit is reached, writes to CDC enabled tables will be rejected
    until space is freed.

**Four CDC settings are configured in the cassandra.yaml**


``cdc_enabled``

Enables/Disables CDC logging per node

``cdc_raw_directory``

The directory where the CDC log is stored.
Package installations(default): /$CASSANDRA_HOME/cdc_raw.
Tarball installations: install_location/data/cdc_raw.

``cdc_total_space_in_mb``

Total space available for storing CDC data. The default is 4096MB and 1/8th of the total space of the drive where the
cdc_raw_directory resides. If space gets above this value, Cassandra will throw **WriteTimeoutException** on Mutations
including tables with CDC enabled.

``cdc_free_space_check_interval_ms``

When the cdc_raw limit is hit and the Consumer is either running behind or experiencing back pressure, this interval is
checked to see if any new space for cdc-tracked tables has been made available.

.. warning::

    After changing properties in the cassandra.yaml file, you must restart the node for the changes to take effect. ``


Prerequisites
-------------

-  Cassandra **3.0.9+**
-  Confluent **3.2+**
-  Java **1.8**
-  Scala **2.11**


Setup
-----

Before we can do anything, including the QuickStart we need to install Cassandra and the Confluent platform.


Cassandra Setup
~~~~~~~~~~~~~~~


First download and install Cassandra if you don't have a compatible
cluster available.

.. sourcecode:: bash

    #make a folder for cassandra
    mkdir cassandra

    #Download Cassandra
    wget http://apache.mirror.anlx.net/cassandra/3.11.0/apache-cassandra-3.11.0-bin.tar.gz

    #extract archive to cassandra folder
    tar xvf apache-cassandra-3.11.0-bin.tar.gz -C cassandra --strip-components=1

    #enable the CDC in the yaml configuration
    sed -i -- 's/cdc_enabled: false/cdc_enabled: true/g'  conf/cassandra.yaml

    #set CASSANDRA_HOME
    export CASSANDRA_HOME=$(pwd)/cassandra

    #Start Cassandra
    cd cassandra
    sudo sh /bin/cassandra


.. note::

    There can be only one instance of Apache Cassandra node per machine for the connector to run properly. All nodes
    should have the same path for the cdc_raw folder

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.


Source Connector
~~~~~~~~~~~~~~~~

The Cassandra CDC Source connector will read the commit log mutations from the CDC logs and will push them to the target topic.

.. note::

    messages sent to Kafka are in AVRO format.

The record pushed to Kafka populates both the key and the value part. Key will contain the metadata of the change while
the value will contain the actual data change.


Record Key
^^^^^^^^^^

The key data structure follows this layout:

.. sourcecode:: javascript

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


Based on the mutation information we can identify the following types of changes:

``INSERT``

A record has been inserted/a record columns have been updated. There is no real solution for identifying an
UPDATE unless the connector keeps track of all the keys seen.

.. sourcecode:: sql

    INSERT INTO keyspace.orders (id, created, product, qty, price) VALUES (1, now(), 'OP-DAX-P-20150201-95.7', 100, 94.2)

``DELETE``

An entire record has been deleted (tombstoned)

.. sourcecode:: sql

    DELETE FROM datamountaineer.orders where id = 1


``DELETE_COLUMN``

Specific columns have been deleted (non PK columns).

.. sourcecode:: sql

    DELETE product FROM datamountaineer.orders where id = 1
    DELETE name.firstname FROM datamountaineer.users WHERE id=62c36092-82a1-3a00-93d1-46196ee77204;

In this case the ``deleted_columns`` entry will contain "product/name.firstname". If more than one column is deleted we
will retain that information.


Value Key
^^^^^^^^^

The Kafka message value part contains the actual mutation data. Apart from the primary keys columns all the other columns
have an optional schema in avro. The reason for that is because one can set the values on a subset of them during a
CQL insert/update. In the QuickStart section we make use of the ``users`` table.  The value AVRO schema associated with
it looks like this

.. sourcecode:: json

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

And a json representation of the actual value looks like this:

.. sourcecode:: javascript

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


Data Types
^^^^^^^^^^

The Source connector needs to map Apache Cassandra types to Kafka Connect Schema types. For the ones not so familiar
with connect here is the list of supported Connect Types:


*   INT8,
*   INT16
*   INT32
*   INT64
*   FLOAT32
*   FLOAT64
*   BOOLEAN
*   STRING
*   BYTES
*   ARRAY
*   MAP
*   STRUCT

Along these primitive types there are the logical types for :

*   Date
*   Decimal
*   Time
*   Timestamp

As a result for most Apache Cassandra Types we have an equivalent type for Connect Schema. A Connect Source Record will
be marshaled as AVRO when sent to Kafka.

+------------------+-----------------------------------+
| CQL Type         | Connect Data Type                 |
+==================+===================================+
|AsciiType         | OPTIONAL STRING                   |
+------------------+-----------------------------------+
|LongType          | OPTIONAL INT64                    |
+------------------+-----------------------------------+
|BytesType         | OPTIONAL BYTES                    |
+------------------+-----------------------------------+
|BooleanType       | OPTIONAL BOOLEAN                  |
+------------------+-----------------------------------+
|CounterColumnType | OPTIONAL INT64                    |
+------------------+-----------------------------------+
|SimpleDateType    | OPTIONAL Kafka Connect Date       |
+------------------+-----------------------------------+
|DoubleType        | OPTIONAL FLOAT64                  |
+------------------+-----------------------------------+
|DecimalType       | OPTIONAL Kafka Connect Decimal    |
+------------------+-----------------------------------+
|DurationType      | OPTIONAL STRING                   |
+------------------+-----------------------------------+
|EmptyType         | OPTIONAL STRING                   |
+------------------+-----------------------------------+
|FloatType         | OPTIONAL FLOAT32                  |
+------------------+-----------------------------------+
|InetAddressType   | OPTIONAL STRING                   |
+------------------+-----------------------------------+
|Int32Type         | OPTIONAL INT32                    |
+------------------+-----------------------------------+
|ShortType         | OPTIONAL INT16                    |
+------------------+-----------------------------------+
|UTF8Type          | OPTIONAL STRING                   |
+------------------+-----------------------------------+
|TimeType          | OPTIONAL KAFKA CONNECT Time       |
+------------------+-----------------------------------+
|TimestampType     | OPTIONAL KAFKA CONNECT Timestamp  |
+------------------+-----------------------------------+
|TimeUUIDType      | OPTIONAL STRING                   |
+------------------+-----------------------------------+
|ByteType          | OPTIONAL INT8                     |
+------------------+-----------------------------------+
|UUIDType          | OPTIONAL STRING                   |
+------------------+-----------------------------------+
|IntegerType       | OPTIONAL INT32                    |
+------------------+-----------------------------------+
|ListType          | OPTIONAL ARRAY of the inner type  |
+------------------+-----------------------------------+
|MapType           | OPTIONAL MAP of the inner types   |
+------------------+-----------------------------------+
|SetType           | OPTIONAL ARRAY of the inner type  |
+------------------+-----------------------------------+
|UserType          | OPTIONAL STRUCT for the user type |
+------------------+-----------------------------------+

Please note we default to String for the these CQL types: DurationType, InetAddressType, TimeUUIDType, UUIDType.


How does it work
~~~~~~~~~~~~~~~~

It is expected that Kafka Connect worker will run on the same node as the Apache Cassandra node.

.. important::

    Only one Apache Cassandra Node should run per machine to have the Connector work properly
    The **cdc_raw** folder location should be the same on all nodes running Apache Cassandra Node
    There should be only one Connector Worker instance per machine. Any more won't have any effect

Cassandra supports  a master-less "ring" architecture. Each of the node in the Cassandra ring cluster will be
responsible for storing the table records. The partition key hash and number of rings in the cluster determines the where
each record is stored. (we leave aside replication from this discussion).

Upon flushing the memtable to disk, all the commit log segments containing data for CDC-enabled tables are moved to the
configured cdc_raw directory. It is only at this point the connector will pick up the changes.

.. important::

    Changes in Cassandra are not picked up immediately. The memtables need to be flushed for the CDC commit logs to be available.
    You can use nodetool to flush the tables

Once a file lands in the CDC folder the Connector will pick it up and read the mutations. A CDC file can contain mutations
for more than one table. Each mutation for the subscribed tables will be translated into a Kafka Connect Source Record
which will be sent by the Connect framework to the topic configured in the connector properties.

The Connect source will process the files in the order they were created and one by one. This ensures the change
sequence is retained. Once the records have been pushed to Kafka the CDC file is deleted.

The connector will only be able to read the mutations for the subscribed tables. Via configuration you can express which
tables to consider and what topic should receive those mutations information.

.. sourcecode:: sql

    INSERT INTO ordersTopic SELECT * FROM datamountaineer.orders


.. important::

    Enabling CDC on a new table means you need to restart the connector for the changes to be picked up. The connector is
    driven by the configurations and not by the list of all the tables with CDC enabled. (Might be a feature, change to do)

Below you can find a flow diagram describing the process mentioned above.

.. sourcecode:: bash

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


Source Connector QuickStart
~~~~~~~~~~~~~~~~~~~~~~~~~~~

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and
updating the configuration. We have developed a Command Line Interface to make interacting with the Connect Rest API
easier. The CLI can be found in the Stream Reactor download under the ``bin`` folder. Alternatively the Jar can be
pulled from our GitHub `releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.


Once you have installed and started Cassandra create a table to capture the mutations. We will use a bit more complex
column family structure to show case what we support so far.

Let's start the cql shell tool to create our table

.. sourcecode:: bash

    $./bin/cqlsh
    Connected to Test Cluster at 127.0.0.1:9042.
	[cqlsh 5.0.1 | Cassandra 3.11.0 | CQL spec 3.4.4 | Native protocol v4]
	Use HELP for help.
	cqlsh>

Now let's create a keyspace and a users column family(table)

.. sourcecode:: bash

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


Starting the Connector (Distributed)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running:

.. sourcecode:: bash

    #make sure you have $CASSANDRA_HOME variable setup (see  Cassandra setup)
    $CONFLUENT_HOME/bin/connect-distributed.sh $CONFLUENT_HOME/etc/schema-registry/connect-avro-distributed.properties


.. sourcecode:: bash

    #make sure you have $CASSANDRA_HOME variable setup (see Cassandra setup)
    $ cat <<EOF > cassandra-cdc-source.json
    {
      "name": "cassandra-connect-cdc",
      "config":{
        "name":"cassandra-connct-cdc",
        "tasks": 1,
        "connector.class":"com.datamountaineer.streamreactor.connect.cassandra.cdc.CassandraCdcSourceConnector",
        "connect.cassandra.kcql":"INSERT INTO users-topic SELECT * FROM datamountaineer.users",
        "connect.cassandra.yaml.path.url": "$CASSANDRA_HOME/conf/cassandra.yaml",
        "connect.cassandra.port": "9042",
        "connect.cassandra.contact.points":"localhost"
      }
    }
    EOF


If you visualize the file it should print something like (I used the `..` because you might have a different path to where you stored Cassandra)

.. sourcecode:: json

    {
      "name": "cassandra-connect-cdc",
      "config":{
        "name":"cassandra-connct-cdc",
        "tasks": 1,
        "connector.class":"com.datamountaineer.streamreactor.connect.cassandra.cdc.CassandraCdcSourceConnector",
        "connect.cassandra.kcql":"INSERT INTO orders-topic SELECT * FROM datamountaineer.orders",
        "connect.cassandra.yaml.path.url": "../cassandra/conf/cassandra.yaml",
        "connect.cassandra.port": "9042",
        "connect.cassandra.contact.points":"localhost"
      }
    }

Next step is to spin up the connector. And for that we run the following bash script:

.. sourcecode:: bash

    curl -X POST -H "Content-Type: application/json" --data @cassandra-cdc-source.json http://localhost:8083/connectors

The output from the connect-distributed should read something similar to this:


.. sourcecode:: bash

    [2017-08-01 22:34:35,653] INFO Acquiring port 64101 to enforce single instance being run on Stepi (com.datamountaineer.streamreactor.connect.cassandra.cdc.CassandraCdcSourceTask:53)
    [2017-08-01 22:34:35,660] INFO
      ____        _        __  __                   _        _
     |  _ \  __ _| |_ __ _|  \/  | ___  _   _ _ __ | |_ __ _(_)_ __   ___  ___ _ __
     | | | |/ _` | __/ _` | |\/| |/ _ \| | | | '_ \| __/ _` | | '_ \ / _ \/ _ \ '__|
     | |_| | (_| | || (_| | |  | | (_) | |_| | | | | || (_| | | | | |  __/  __/ |
     |____/ \__,_|\__\__,_|_|  |_|\___/ \__,_|_| |_|\__\__,_|_|_| |_|\___|\___|_|
       ____   by Stefan Bocutiu          _              ____ ____   ____
      / ___|__ _ ___ ___  __ _ _ __   __| |_ __ __ _   / ___|  _ \ / ___|
     | |   / _` / __/ __|/ _` | '_ \ / _` | '__/ _` | | |   | | | | |
     | |__| (_| \__ \__ \ (_| | | | | (_| | | | (_| | | |___| |_| | |___
      \____\__,_|___/___/\__,_|_| |_|\__,_|_|  \__,_|  \____|____/ \____|

     (com.datamountaineer.streamreactor.connect.cassandra.cdc.CassandraCdcSourceTask:65)
    [2017-08-01 22:34:36,088] INFO CDC path is not set in Yaml. Using the default location (com.datamountaineer.streamreactor.connect.cassandra.cdc.logs.CdcCassandra:55)
    [2017-08-01 22:34:36,411] INFO Detected Guava >= 19 in the classpath, using modern compatibility layer (com.datastax.driver.core.GuavaCompatibility:132)


Let's go back to the cqlsh terminal and insert some records into the users table and then perform some updates and deletes.

.. sourcecode:: bash

    INSERT INTO datamountaineer.users(id, name) VALUES (62c36092-82a1-3a00-93d1-46196ee77204,{firstname:'Marie-Claude',lastname:'Josset'});

    UPDATE datamountaineer.users
      SET
      addresses = addresses + {
      'home': {
          street: '191 Rue St. Charles',
          city: 'Paris',
          zip_code: 75015,
          phones: {'33 6 78 90 12 34'}
      },
      'work': {
          street: '81 Rue de Paradis',
          city: 'Paris',
          zip_code: 7500,
          phones: {'33 7 12 99 11 00'}
      }
    }
    WHERE id=62c36092-82a1-3a00-93d1-46196ee77204;

    INSERT INTO datamountaineer.users(id, direct_reports) VALUES (11c11111-82a1-3a00-93d1-46196ee77204,{{firstname:'Jean-Claude',lastname:'Van Damme'}, {firstname:'Arnold', lastname:'Schwarzenegger'}});

    INSERT INTO datamountaineer.users(id, other_reports) VALUES (22c11111-82a1-3a00-93d1-46196ee77204,[{firstname:'Jean-Claude',lastname:'Van Damme'}, {firstname:'Arnold', lastname:'Schwarzenegger'}]);

    DELETE name.firstname FROm datamountaineer.users WHERE id=62c36092-82a1-3a00-93d1-46196ee77204;


You will notice from the logs there are no new CDC files picked up and if you navigate to the CDC ouput folder you will see it is empty. The memtables needs to fill up to be flushed to disk.
Let's use the tool provided by Apache Cassandra to flush the table:``nodetool``

.. sourcecode:: bash

    $ $CASSANDRA_HOME/bin/nodetool drain

Once this completes your connect distributed log should print something along these lines:

.. sourcecode:: bash

    [2017-08-01 23:10:42,842] INFO Reading mutations from the CDC file:/home/stepi/work/programs/cassandra/data/cdc_raw/CommitLog-6-1501625205002.log. Checking file is still being written... (com.datamountaineer.streamreactor.connect.cassandra.cdc.logs.CdcCassandra:158)
    [2017-08-01 23:10:43,352] INFO Global buffer pool is enabled, when pool is exhausted (max is 0.000KiB) it will allocate on heap (org.apache.cassandra.utils.memory.BufferPool:230)
    [2017-08-01 23:10:43,355] INFO Maximum memory usage reached (0.000KiB), cannot allocate chunk of 1.000MiB (org.apache.cassandra.utils.memory.BufferPool:91)
    [2017-08-01 23:10:43,390] ERROR [Control connection] Cannot connect to any host, scheduling retry in 4000 milliseconds (com.datastax.driver.core.ControlConnection:153)
    [2017-08-01 23:10:43,569] INFO 5 changes detected in /home/stepi/work/programs/cassandra/data/cdc_raw/CommitLog-6-1501625205002.log (com.datamountaineer.streamreactor.connect.cassandra.cdc.logs.CdcCassandra:173)

Let's see what was sent over to the users topic. We will run ``kafka-avro-console-consumer`` to read the records


.. sourcecode:: bash

    $ ./bin/kafka-avro-console-consumer     --zookeeper localhost:2181     --topic users-topic     --from-beginning --property print.key=true
    SLF4J: Class path contains multiple SLF4J bindings.
    SLF4J: Found binding in [jar:file:/home/stepi/work/programs/confluent-3.2.2/share/java/kafka-serde-tools/slf4j-log4j12-1.7.6.jar!/org/slf4j/impl/StaticLoggerBinder.class]
    SLF4J: Found binding in [jar:file:/home/stepi/work/programs/confluent-3.2.2/share/java/schema-registry/slf4j-log4j12-1.7.6.jar!/org/slf4j/impl/StaticLoggerBinder.class]
    SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
    SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
    Using the ConsoleConsumer with old consumer is deprecated and will be removed in a future major release. Consider using the new consumer by passing [bootstrap-server] instead of [zookeeper].
    {"keyspace":"datamountaineer","table":"users","changeType":"INSERT","deleted_columns":null,"keys":{"id":{"string":"62c36092-82a1-3a00-93d1-46196ee77204"}},"timestamp":1501625394958965}	{"name":{"fullname":{"firstname":{"string":"Marie-Claude"},"lastname":{"string":"Josset"}}},"addresses":null,"direct_reports":null,"id":{"string":"62c36092-82a1-3a00-93d1-46196ee77204"},"other_reports":null}
    {"keyspace":"datamountaineer","table":"users","changeType":"INSERT","deleted_columns":null,"keys":{"id":{"string":"62c36092-82a1-3a00-93d1-46196ee77204"}},"timestamp":1501625395008930}	{"name":null,"addresses":{"array":[{"key":{"string":"work"},"value":{"address":{"street":{"string":"81 Rue de Paradis"},"city":{"string":"Paris"},"zip_code":{"int":7500},"phones":{"array":[{"string":"33 7 12 99 11 00"}]}}}},{"key":{"string":"home"},"value":{"address":{"street":{"string":"191 Rue St. Charles"},"city":{"string":"Paris"},"zip_code":{"int":75015},"phones":{"array":[{"string":"33 6 78 90 12 34"}]}}}}]},"direct_reports":null,"id":{"string":"62c36092-82a1-3a00-93d1-46196ee77204"},"other_reports":null}
    {"keyspace":"datamountaineer","table":"users","changeType":"INSERT","deleted_columns":null,"keys":{"id":{"string":"11c11111-82a1-3a00-93d1-46196ee77204"}},"timestamp":1501625395013654}	{"name":null,"addresses":null,"direct_reports":{"array":[{"fullname":{"firstname":{"string":"Arnold"},"lastname":{"string":"Schwarzenegger"}}},{"fullname":{"firstname":{"string":"Jean-Claude"},"lastname":{"string":"Van Damme"}}}]},"id":{"string":"11c11111-82a1-3a00-93d1-46196ee77204"},"other_reports":null}
    {"keyspace":"datamountaineer","table":"users","changeType":"INSERT","deleted_columns":null,"keys":{"id":{"string":"22c11111-82a1-3a00-93d1-46196ee77204"}},"timestamp":1501625395015668}	{"name":null,"addresses":null,"direct_reports":null,"id":{"string":"22c11111-82a1-3a00-93d1-46196ee77204"},"other_reports":{"array":[{"fullname":{"firstname":{"string":"Jean-Claude"},"lastname":{"string":"Van Damme"}}},{"fullname":{"firstname":{"string":"Arnold"},"lastname":{"string":"Schwarzenegger"}}}]}}
    {"keyspace":"datamountaineer","table":"users","changeType":"DELETE_COLUMN","deleted_columns":{"array":["name.firstname"]},"keys":{"id":{"string":"62c36092-82a1-3a00-93d1-46196ee77204"}},"timestamp":1501625395018481}	{"name":null,"addresses":null,"direct_reports":null,"id":{"string":"62c36092-82a1-3a00-93d1-46196ee77204"},"other_reports":null}


Exactly what was changed!!!

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


Configurations
--------------
Here is a full list of configuration entries the connector knows about.


+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| Name                                           | Description                                  | Data    | Optio | Default   |
|                                                |                                              | Type    | nal   |           |
+================================================+==============================================+=========+=======+===========+
| connect.cassandra.contact.points               | Contact points (hosts) in                    | string  | no    |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.port                         | Cassandra Node Client connection port        | int     | yes   | 9042      |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.username                     | Username to connect to Cassandra with if     | string  | yes   |           |
|                                                | ``connect.cassandra.authentication.mode``    |         |       |           |
|                                                | is set to *username\_password*               |         |       |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.password                     | Password to connect to Cassandra with if     | string  | yes   |           |
|                                                | ``connect.cassandra.authentication.mode``    |         |       |           |
|                                                | is set to *username\_password*.              |         |       |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.ssl.enabled                  | Enables SSL communication against SSL enabled| boolean | yes   | false     |
|                                                | Cassandra cluster.                           |         |       |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.trust.store.password         | Password for truststore.                     | string  | yes   |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.key.store.path               | Path to truststore.                          | string  | yes   |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.key.store.password           | Password for key store.                      | string  | yes   |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.ssl.client.cert.auth         | Path to keystore.                            | string  | yes   |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.kcql                         | Kafka connect query language expression.     | string  | no    |           |
|                                                | Allows for expressive table to topic         |         |       |           |
|                                                | routing. It  describes which CDC tables      |         |       |           |
|                                                | are monitored and the target Kafka topic for |         |       |           |
|                                                | Cassandra CDC information.                   |         |       |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.cdc.path.url                 | The location of the Cassandra Yaml file in   | string  | no    |           |
|                                                | URL format:file://path.                      |         |       |           |
|                                                | The connector reads the file to get the cdc  |         |       |           |
|                                                | folder but also sets the internals of        |         |       |           |
|                                                | the Cassandra API allowing it to read the    |         |       |           |
|                                                | CDC files                                    |         |       |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.cdc.file.watch.interval      | The delay time in milliseconds               | long    | yes   | 2000      |
|                                                | before the connector checks for              |         |       |           |
|                                                | Cassandra CDC files. We poll                 |         |       |           |
|                                                | the CDC folder for new files.                |         |       |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.cdc.mutation.queue.size      | The maximum number of Cassandra mutation     | int     | yes   | 1000000   |
|                                                | to buffer. As it reads from the Cassandra CDC|         |       |           |
|                                                | files the mutations are buffered before they |         |       |           |
|                                                | are handed over to Kafka Connect.            |         |       |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.cdc.enable.delete.while.read | The worker CDC thread will read a CDC file   | boolean | yes   | false     |
|                                                | checking if any of the processed files are   |         |       |           |
|                                                | ready to be deleted (that means the records  |         |       |           |
|                                                | have been sent to kafka). Rather than waiting|         |       |           |
|                                                | for a read to complete we can delete the     |         |       |           |
|                                                | files while reading a CDC file.              |         |       |           |
|                                                | a CDC file. You can disable it for faster    |         |       |           |
|                                                | faster reads by setting the value to false.  |         |       |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.cdc.single.instance.port     | Kafka Connect framework doesn’t allow yet    | int     | yes   | 64101     |
| ra.cdc.single.i                                | configuration where you are running only     |         |       |           |
|                                                | one task per worker. If you allocate more    |         |       |           |
|                                                | tasks than workers then some will spin up    |         |       |           |
|                                                | more tasks. With Cassandra nodes we want one |         |       |           |
|                                                | want one worker and one task - not more.     |         |       |           |
|                                                | To ensure this we allow the first task to    |         |       |           |
|                                                | grab a port  subsequent calls to open the    |         |       |           |
|                                                | port will fail thus not allowing multiple    |         |       |           |
|                                                | instance running at once                     |         |       |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+
| connect.cassandra.cdc.decimal.scale            | When reading the column family metadata we   |         |       |           |
|                                                | don’t have details about decimal scale       |         |       |           |
+------------------------------------------------+----------------------------------------------+---------+-------+-----------+

Deployment Guidelines
---------------------

TODO


TroubleShooting
---------------

TODO
