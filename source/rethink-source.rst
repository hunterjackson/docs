Kafka Connect ReThink
=====================

A Connector and Source to write events from ReThinkDB to Kafka. The connector subscribes to changefeeds on tables and
streams the records to Kafka.

The Source supports:

1. :ref:`The KCQL routing querying <kcql>` - Table to topic routing
2. Initialization (Read feed from start) via KCQL.
3. ReThinkDB type (add, delete, update).
4. ReThinkDB initial states.


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

Follow the instructions :ref:`here <install>`.

Sink Connector QuickStart
-------------------------

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and updating the configuration. We have developed
a Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running the ``start-connect.sh`` script in the ``bin`` folder.

.. sourcecode:: bash

    ➜ bin/start-connect.sh

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for ReThinkDB.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create rethink-source < conf/quickstarts/rethink-source.properties
    #Connector name=`rethink-source`
    name=rethink-source
    connect.rethink.source.host=localhost
    connect.rethink.source.port=28015
    connector.class=com.datamountaineer.streamreactor.connect.rethink.source.ReThinkSourceConnector
    tasks.max=1
    connect.rethink.source.db=test
    connect.rethink.sink.kcql=INSERT INTO rethink-topic SELECT * FROM source-test
    #task ids: 0

The ``rethink-source.properties`` file defines:

1.  The name of the source.
2.  The name of the rethink host to connect to.
3.  The rethink port to connect to.
4.  The Source class.
5.  The max number of tasks the connector is allowed to created. The connector splits and groups the `connect.rethink.source.kcql`
    by the number of tasks to ensure a distribution based on allowed number of tasks and Source tables.
6.  The ReThinkDB database to connect to.
7.  :ref:`The KCQL routing querying. <kcql>`

If you switch back to the terminal you started the Connector in you should see the ReThinkDB Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    rethink-source

.. sourcecode:: bash

    [2016-10-05 12:09:35,414] INFO
        ____        __        __  ___                  __        _
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
        connect.rethink.source.kcql = insert into rethink-topic select * from source-test
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

The ReThinkDb Source writes change feed records from RethinkDb to Kafka.

The Source supports:

1. Table to topic routing
2. Initialization (Read feed from start)
3. ReThinkDB type (add, delete, update)
4. ReThinkDB initial states

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L**, :ref:`KCQL <kcql>` allows for routing and mapping using a SQL like syntax,
consolidating typically features in to one configuration option.

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

``connect.rethink.source.kcql``

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
    connect.rethink.sink.kcql=INSERT INTO rethink-topic SELECT * FROM source-test

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
