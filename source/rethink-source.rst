Kafka Connect ReThink
=====================

A Connector and Source to write events from ReThinkDB to Kafka. The connector subscribes to changefeeds on tables and
streams the records to Kafka.

Prerequisites
-------------

- Confluent 3.0.1
- RethinkDb 2.3.3
- Java 1.8
- Scala 2.11

Setup
-----

Rethink Setup
~~~~~~~~~~~~~

Download and install RethinkDb. Follow the instruction `here <https://rethinkdb.com/docs/install/>`__ dependent on your
operating system.


Confluent Setup
~~~~~~~~~~~~~~~

.. sourcecode:: bash

    #make confluent home folder
    ➜  mkdir confluent

    #download confluent
    ➜  wget http://packages.confluent.io/archive/3.0/confluent-3.0.1-2.11.tar.gz

    #extract archive to confluent folder
    ➜  tar -xvf confluent-3.0.1-2.11.tar.gz -C confluent

    #setup variables
    ➜  export CONFLUENT_HOME=~/confluent/confluent-3.0.1

Start the Confluent platform.

.. sourcecode:: bash

    #Start the confluent platform, we need kafka, zookeeper and the schema registry
    ➜  bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    ➜  bin/kafka-server-start etc/kafka/server.properties &
    ➜  bin/schema-registry-start etc/schema-registry/schema-registry.properties &

Build the Connector and CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The prebuilt jars can be taken from `here <https://github.com/datamountaineer/stream-reactor/releases>`__ and
`here <https://github.com/datamountaineer/kafka-connect-tools/releases>`__
or from `Maven <http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22kafka-connect-cli%22>`__

If you want to build the connector, clone the repo and build the jar.

.. sourcecode:: bash

    ##Build the connectors
    ➜  git clone https://github.com/datamountaineer/stream-reactor
    ➜  cd stream-reactor
    ➜  gradle fatJar

    ##Build the CLI for interacting with Kafka connectors
    ➜  git clone https://github.com/datamountaineer/kafka-connect-tools
    ➜  cd kafka-connect-tools
    ➜  gradle fatJar

Sink Connector QuickStart
-------------------------

Next we will start the connector in distributed mode. Connect has two modes, standalone where the tasks run on only one host
and distributed mode. Usually you'd run in distributed mode to get fault tolerance and better performance. In distributed mode
you start Connect on multiple hosts and they join together to form a cluster. Connectors which are then submitted are
distributed across the cluster.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Sink Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a file called ``rethinkdb-sink.properties`` with the contents below:

.. sourcecode:: bash

    name=rethink-source
    connect.rethink.source.host=localhost
    connect.rethink.source.port=28015
    connector.class=com.datamountaineer.streamreactor.connect.rethink.source.ReThinkSourceConnector
    tasks.max=1
    connect.rethink.source.db=localhost
    connect.rethink.export.route.query=INSERT INTO rethink-topic SELECT * FROM source-test

This configuration defines:

1.  The name of the source.
2.  The name of the rethink host to connect to.
3.  The rethink port to connect to.
4.  The source class.
5.  The max number of tasks the connector is allowed to created. The connector splits and groups the `connect.rethink.import.route.query`
    by the number of tasks to ensure a distribution based on allowed number of tasks and source tables.
6.  The ReThinkDB database to connect to.
7.  The KCQL statement for topic routing.

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Connectors can be deployed distributed mode. In this mode one or many connectors are started on the same or different
hosts with the same cluster id. The cluster id can be found in ``etc/schema-registry/connect-avro-distributed.properties.``

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

Now start the connector in distributed mode. We only give it one properties file for the kafka, zookeeper and
schema registry configurations.

First add the connector jar to the CLASSPATH and then start Connect.

.. note::

    You need to add the connector to your classpath or you can create a folder in ``share/java`` of the Confluent
    install location like, kafka-connect-myconnector and the start scripts provided by Confluent will pick it up.
    The start script looks for folders beginning with kafka-connect.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-rethink-0.2-cp-3.0.1.all.jar

.. sourcecode:: bash

    ➜  confluent-3.0.1/bin/connect-distributed confluent-3.0.1/etc/schema-registry/connect-avro-distributed.properties

Once the connector has started lets use the kafka-connect-tools cli to post in our distributed properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.5-all.jar create rethink-sink < rethink-sink.properties


If you switch back to the terminal you started the Connector in you should see the ReThinkDB Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ java -jar build/libs/kafka-connect-cli-0.5-all.jar ps
    rethink-sink

    ➜ java -jar build/libs/kafka-connect-cli-0.5-all.jar get rethink-sink
    #Connector name=`rethink-source`
    name=rethink-source
    connect.rethink.source.host=localhost
    connect.rethink.source.port=28015
    connector.class=com.datamountaineer.streamreactor.connect.rethink.source.ReThinkSourceConnector
    tasks.max=1
    connect.rethink.source.db=test
    connect.rethink.export.route.query=INSERT INTO rethink-topic SELECT * FROM source-test
    #task ids: 0

.. sourcecode:: bash

    [2016-10-05 12:09:35,414] INFO     ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
        ____     ________    _       __   ____  ____ _____
       / __ \___/_  __/ /_  (_)___  / /__/ __ \/ __ ) ___/____  __  _______________
      / /_/ / _ \/ / / __ \/ / __ \/ //_/ / / / __  \__ \/ __ \/ / / / ___/ ___/ _ \
     / _, _/  __/ / / / / / / / / / ,< / /_/ / /_/ /__/ / /_/ / /_/ / /  / /__/  __/
    /_/ |_|\___/_/ /_/ /_/_/_/ /_/_/|_/_____/_____/____/\____/\__,_/_/   \___/\___/

     By Andrew Stevenson (com.datamountaineer.streamreactor.connect.rethink.source.ReThinkSourceTask:48)
    [2016-10-05 12:09:35,420] INFO ReThinkSourceConfig values:
        connect.rethink.source.port = 28015
        connect.rethink.source.host = localhost
        connect.rethink.import.route.query = insert into rethink-topic select * from source-test
        connect.rethink.source.db = test


Test Records
^^^^^^^^^^^^

Go to the ReThink Admin console `<http://localhost:8080/#tables>`__ and add a database called `test` and table
called `source-test`. Then on the Data Explorer tab insert the following and hit run to insert the record into the table.

.. sourcecode:: javascript

    r.table('source_test').insert([
        { name: "datamountaineers-rule", tv_show: "Battlestar Galactica",
          posts: [
            {title: "Decommissioning speech3", content: "The Cylon War is long over..."},
            {title: "We are at war", content: "Moments ago, this ship received word..."},
            {title: "The new Earth", content: "The discoveries of the past few days..."}
          ]
        }
    ])


Check for records in Kafka
~~~~~~~~~~~~~~~~~~~~~~~~~~

Check Kafka with the console consumer

.. sourcecode:: bash

 ➜  confluent confluent-3.0.1/bin/kafka-avro-console-consumer \
    --zookeeper localhost:2181 \
    --topic rethink-topic \
    --from-beginning

    {"state":{"string":"initializing"},"old_val":null,"new_val":null,"type":{"string":"state"}}
    {"state":{"string":"ready"},"old_val":null,"new_val":null,"type":{"string":"state"}}
    {"state":null,"old_val":null,"new_val":{"string":"{tv_show=Battlestar Galactica, name=datamountaineers-rule, id=ec9d337e-ee07-4128-a830-22e4f055ce64, posts=[{title=Decommissioning speech3, content=The Cylon War is long over...}, {title=We are at war, content=Moments ago, this ship received word...}, {title=The new Earth, content=The discoveries of the past few days...}]}"},"type":{"string":"add"}}



Features
--------

The ReThinkDb source writes change feed records from RethinkDb to Kafka.

The Source supports:

1. Table to topic routing
2. Initialization (Read feed from start)
3. ReThinkDB type (add, delete, update)
4. ReThinkDB initial states

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The ReThink Source supports the following:

.. sourcecode:: bash

    INSERT INTO <target table> SELECT <fields> FROM <source topic> <INITIALIZE>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select all fields from topicA and write to tableA, read from start
    INSERT INTO tableA SELECT * FROM topicA INITIALIZE


Configurations
--------------

``connect.rethink.import.route.query``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming. Fields
to be used as the row key can be set by specifing the ``PK``. The below example uses field1 as the primary key.

* Data type : string
* Importance: high
* Optional  : no

Examples:

.. sourcecode:: sql

    INSERT INTO TOPIC1 SELECT * FROM TABLE1;INSERT INTO TOPIC2 SELECT * FROM TABLE2

``connect.rethink.source.host``

Specifies the rethink server.

* Data type : string
* Importance: high
* Optional  : no

``connect.rethink.source.port``

Specifies the rethink server port number.

* Data type : int
* Importance: high
* Optional  : yes

Example
~~~~~~~

.. sourcecode:: bash

    name=rethink-source
    connect.rethink.source.db=localhost
    connect.rethink.source.port=28015
    connector.class=com.datamountaineer.streamreactor.connect.rethink.source.ReThinkSourceConnector
    tasks.max=1
    connect.rethink.export.route.query=INSERT INTO rethink-topic SELECT * FROM source-test

Schema Evolution
----------------

The schema is fixed. The following schema is used:

+---------+---------+---------+
| Name    | Type    | Optional|
+---------+---------+---------+
| state   | string  | yes     |
+---------+---------+---------+
| new_val | string  | yes     |
+---------+---------+---------+
| old_val | string  | yes     |
+---------+---------+---------+
| type    | string  | yes     |
+---------+---------+---------+


Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
