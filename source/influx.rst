Kafka Connect Influx
====================

A Connector and Sink to write events from Kafka to InfluxDB. The connector takes the value from the Kafka Connect SinkRecords
and inserts a new entry to InfluxDB.

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>` - Topic to index mapping and Field selection.
2. Auto mapping of the Kafka topic schema to the index.

Prerequisites
-------------

- Confluent 3.0.1
- Java 1.8
- Scala 2.11

Setup
-----

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

InfluxDB Setup
~~~~~~~~~~~~~~

`Download <https://influxdata.com/downloads/#influxdb>`__ and start InfluxDB. Users of OS X 10.8 and higher can install InfluxDB using the Homebrew package manager.
Once brew is installed, you can install InfluxDB by running:

.. sourcecode:: bash

    brew update
    brew install influxdb

.. note::

    InfluxDB starts an Admin web server listening on port 8083 by default. For this quickstart this will collide with Kafka
    Connects default port of 8083. Since we are running on a single node we will need to  edit the InfluxDB config.

    .. sourcecode:: bash

        #create config dir
        sudo mkdir /etc/influxdb
        #dump the config
        influxd config > /etc/influxdb/influxdb.generated.conf

    Now change the following section to a port 8087 or any other free port.

    .. sourcecode:: bash

        [admin]
        enabled = true
        bind-address = ":8087"
        https-enabled = false
        https-certificate = "/etc/ssl/influxdb.pem"

Now start InfluxDB.

.. sourcecode:: bash

    influxd

If you are running on a single node start InfluxDB with the new configuration file we generated.

.. sourcecode:: bash

    influxd -config  /etc/influxdb/influxdb.generated.conf

Sink Connector QuickStart
-------------------------

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and updating the configuration. We have developed
a Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

Test data
~~~~~~~~~

The Sink expects a database to exist in InfluxDB. Use the InfluxDB CLI to create this:

.. sourcecode:: bash

    ➜  ~ influx
    Visit https://enterprise.influxdata.com to register for updates, InfluxDB server management, and monitoring.
    Connected to http://localhost:8086 version v1.0.2
    InfluxDB shell version: v1.0.2

.. sourcecode:: bash

    > CREATE DATABASE mydb


Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running the ``start-connect.sh`` script in the ``bin`` folder.

.. sourcecode:: bash

    ➜ bin/start-connect.sh

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for InfluxDB.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create influx-sink < conf/influxdb-sink.properties

    #Connector name=`influx-sink`
    name=influxdb-sink
    connector.class=com.datamountaineer.streamreactor.connect.influx.InfluxSinkConnector
    tasks.max=1
    topics=influx-topic
    connect.influx.sink.route.query=INSERT INTO influxMeasure SELECT * FROM influx-topic WITHTIMESTAMP sys_time()
    connect.influx.connection.url=http://localhost:8086
    connect.influx.connection.database=mydb
    #task ids: 0

The ``elastic-sink.properties`` file defines:

1. The name of the connector.
2. The class containing the connector.
3. The max number of task allowed for this connector.
4. The Source topic to get records from.
5. :ref:`The KCQL routing querying. <kcql>`
6. The InfluxDB connection URL.
7. The InfluxDB database.

If you switch back to the terminal you started Kafka Connect in you should see the InfluxDB Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    influxdb-sink

.. sourcecode:: bash

    INFO
      ____        _        __  __                   _        _
     |  _ \  __ _| |_ __ _|  \/  | ___  _   _ _ __ | |_ __ _(_)_ __   ___  ___ _ __
     | | | |/ _` | __/ _` | |\/| |/ _ \| | | | '_ \| __/ _` | | '_ \ / _ \/ _ \ '__|
     | |_| | (_| | || (_| | |  | | (_) | |_| | | | | || (_| | | | | |  __/  __/ |
     |____/ \__,_|\__\__,_|_|  |_|\___/ \__,_|_| |_|\__\__,_|_|_| |_|\___|\___|_|
      ___        __ _            ____  _       ____  _       _ by Stefan Bocutiu
     |_ _|_ __  / _| |_   ___  _|  _ \| |__   / ___|(_)_ __ | | __
      | || '_ \| |_| | | | \ \/ / | | | '_ \  \___ \| | '_ \| |/ /
      | || | | |  _| | |_| |>  <| |_| | |_) |  ___) | | | | |   <
     |___|_| |_|_| |_|\__,_/_/\_\____/|_.__/  |____/|_|_| |_|_|\_\
      (com.datamountaineer.streamreactor.connect.influx.InfluxSinkTask:45)
    [INFO InfluxSinkConfig values:
        connect.influx.retention.policy = autogen
        connect.influx.error.policy = THROW
        connect.influx.connection.user = root
        connect.influx.connection.database = mydb
        connect.influx.connection.password = [hidden]
        connect.influx.connection.url = http://localhost:8086
        connect.influx.retry.interval = 60000
        connect.influx.sink.route.query = INSERT INTO influxMeasure SELECT * FROM influx-topic WITHTIMESTAMP sys_time()
        connect.influx.max.retires = 20
     (com.datamountaineer.streamreactor.connect.influx.config.InfluxSinkConfig:178)


Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``company`` field of type
string a ``address`` field of type string, an ``latitude`` field of type int and a ``longitude`` field of type int.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic influx-topic \
      --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.influx","fields":[{"name":"company","type":"string"},{"name":"address","type":"string"},{"name":"latitude","type":"float"},{"name":"longitude","type":"float"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"company": "DataMountaineer","address": "MontainTop","latitude": -49.817964,"longitude": -141.645812}

Check for records in InfluxDB
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this:

.. sourcecode:: bash

    INFO Setting newly assigned partitions [influx-topic-0] for group connect-influx-sink (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:231)
    INFO Received 1 record(-s) (com.datamountaineer.streamreactor.connect.influx.InfluxSinkTask:81)
    INFO Writing 1 points to the database... (com.datamountaineer.streamreactor.connect.influx.writers.InfluxDbWriter:45)
    INFO Records handled (com.datamountaineer.streamreactor.connect.influx.InfluxSinkTask:83)


Check in InfluxDB.

.. sourcecode:: bash

    ✗ influx
    Visit https://enterprise.influxdata.com to register for updates, InfluxDB server management, and monitoring.
    Connected to http://localhost:8086 version v1.0.2
    InfluxDB shell version: v1.0.2
    > use mydb;
    Using database mydb
    > show measurements;
    name: measurements
    ------------------
    name
    influxMeasure

    > select * from influxMeasure;
    name: influxMeasure
    -------------------
    time			address		async	company		latitude		longitude
    1478269679104000000	MontainTop	true	DataMountaineer	-49.817962646484375	-141.64581298828125


Features
--------

1. Topic to index mapping.
3. Auto mapping of the Kafka topic schema to the index.
4. Field selection

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`__
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The Influx Sink supports the following:

.. sourcecode:: bash

    INSERT INTO <measure> SELECT <fields> FROM <source topic> WITHTIMESTAMP <field_name>|sys_time()

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to indexA
    INSERT INTO measureA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to indexB, use field Y as the point measurement
    INSERT INTO measureB SELECT x AS a, y AS b and z AS c FROM topicB WITHTIMESTAMP y

    #Insert mode, select 3 fields and rename from topicB and write to indexB, use field Y as the current system time for
    #Point measurement
    INSERT INTO measureB SELECT x AS a, y AS b and z AS c FROM topicB WITHTIMESTAMP sys_time()

This is set in the ``connect.influx.export.route.query`` option.

Error Polices
~~~~~~~~~~~~~

The Sink has three error policies that determine how failed writes to the target database are handled. The error policies
affect the behaviour of the schema evolution characteristics of the sink. See the schema evolution section for more
information.

**Throw**

Any error on write to the target database will be propagated up and processing is stopped. This is the default
behaviour.

**Noop**

Any error on write to the target database is ignored and processing continues.

.. warning::

    This can lead to missed errors if you don't have adequate monitoring. Data is not lost as it's still in Kafka
    subject to Kafka's retention policy. The Sink currently does **not** distinguish between integrity constraint
    violations and or other expections thrown by drivers.

**Retry**

Any error on write to the target database causes the RetryIterable exception to be thrown. This causes the
Kafka connect framework to pause and replay the message. Offsets are not committed. For example, if the table is offline
it will cause a write failure, the message can be replayed. With the Retry policy the issue can be fixed without stopping
the sink.

The length of time the Sink will retry can be controlled by using the ``connect.influx.sink.max.retries`` and the
``connect.influx.sink.retry.interval``.

Configurations
--------------

``connect.influx.export.route.query``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming. For
InfluxDB it allows either setting a default or selecting a field from the topic as the Point measurement.

* Data type : string
* Importance: high
* Optional  : no

``connect.influx.connection.url``

The InfluxDB database url.

* Data type : string
* Importance: high
* Optional  : no

``connect.influx.connection.database``

The InfluxDB database.

* Data type : string
* Importance: high
* Optional  : no

``connect.influx.connection.username``

The InfluxDB username.

* Data type : string
* Importance: high
* Optional  : yes

``connect.influx.connection.password``

The InfluxDB password.

* Data type : string
* Importance: high
* Optional  : yes

``connect.influx.retention.policy``

Determines how long InfluxDB keeps the data - the options for specifying the duration of the retention policy are
listed below. Note that the minimum retention period is one hour. DURATION determines how long InfluxDB keeps the
data - the options for specifying the duration of the retention policy are listed below. Note that the minimum retention
period is one hour.

m minutes
h hours
d days
w weeks
INF infinite

Default retention is `autogen` from 1.0 onwards or `default` for any previous version

* Data type : string
* Importance: medium
* Optional  : yes


``connect.influx.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.influx.sink.max.retries``
option. The ``connect.influx.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: medium
* Optional: yes
* Default: RETRY


``connect.influx.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.influx.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: medium
* Optional: yes
* Default: 10


``connect.influx.sink.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.influx.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: high
* Optional: no
* Default : 60000 (1 minute)

Example
~~~~~~~

.. sourcecode:: bash

    name=elastic-sink
    connector.class=com.datamountaineer.streamreactor.connect.elastic.ElasticSinkConnector
    connect.elastic.url=localhost:9300
    connect.elastic.cluster.name=elasticsearch
    tasks.max=1
    topics=test_table
    connect.elastic.export.route.query=INSERT INTO INDEX_1 SELECT field1, field2 FROM TOPIC1

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.

Elastic Search is very flexible about what is inserted. All documents in Elasticsearch are stored in an index. We do not
need to tell Elasticsearch in advance what an index will look like (eg what fields it will contain) as Elasticsearch will
adapt the index dynamically as more documents are added, but we must at least create the index first. The Sink connector
automatically creates the index at start up if it doesn't exist.

The Elastic Search Sink will automatically index if new fields are added to the Source topic, if fields are removed
the Kafka Connect framework will return the default value for this field, dependent of the compatibility settings of the
Schema registry.


Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO

