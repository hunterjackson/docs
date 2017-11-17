Kafka Connect Influx
====================

A Connector and Sink to write events from Kafka to InfluxDB. The connector takes the value from the Kafka Connect SinkRecords
and inserts a new entry to InfluxDB.

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>` - Topic to index mapping and Field selection.
2. Auto mapping of the Kafka topic schema to the index.
3. Payload support for Schema.Struct and payload Struct, Schema.String and Json payload and Json payload with no schema

The Sink supports three Kafka payloads type:

**Connect entry with Schema.Struct and payload Struct.** If you follow the best practice while producing the events, each
message should carry its schema information. Best option is to send Avro. Your connect configurations should be set to
``value.converter=io.confluent.connect.avro.AvroConverter``.
You can find an example `here <https://github.com/confluentinc/kafka-connect-blog/blob/master/etc/connect-avro-standalone.properties>`__.
To see how easy is to have your producer serialize to Avro have a look at
`this <http://docs.confluent.io/3.0.1/schema-registry/docs/serializer-formatter.html?highlight=kafkaavroserializer>`__.
This requires the SchemaRegistry which is open source thanks to Confluent! Alternatively you can send Json + Schema.
In this case your connect configuration should be set to ``value.converter=org.apache.kafka.connect.json.JsonConverter``. This doesn't
require the SchemaRegistry.

**Connect entry with Schema.String and payload json String.** Sometimes the producer would find it easier, despite sending
Avro to produce a GenericRecord, to just send a message with Schema.String and the json string.

**Connect entry without a schema and the payload json String.** There are many existing systems which are publishing json
over Kafka and bringing them in line with best practices is quite a challenge. Hence we added the support.

Prerequisites
-------------

- Confluent 3.3
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
    # start influx
    influxd

Versions Prior to 1.3
^^^^^^^^^^^^^^^^^^^^^

InfluxDB prior to version 1.3 starts an Admin web server listening on port 8083 by default. For this quickstart this will collide with Kafka
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

We you start the Confluent Platform, Kafka Connect is started in distributed mode (``confluent start``). 
In this mode a Rest Endpoint on port ``8083`` is exposed to accept connector configurations. 
We developed Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
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

Download, unpack and install the Stream Reactor and Confluent. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Once the Connect has started we can now use the kafka-connect-tools :ref:`cli <kafka-connect-cli>` to post in our distributed properties file for InfluxDB.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/connect-cli create influx-sink < conf/influxdb-sink.properties

    #Connector name=`influx-sink`
    name=influxdb-sink
    connector.class=com.datamountaineer.streamreactor.connect.influx.InfluxSinkConnector
    tasks.max=1
    topics=influx-topic
    connect.influx.kcql=INSERT INTO influxMeasure SELECT * FROM influx-topic WITHTIMESTAMP sys_time()
    connect.influx.url=http://localhost:8086
    connect.influx.db=mydb
    #task ids: 0

The ``influx-sink.properties`` file defines:

1. The name of the connector.
2. The class containing the connector.
3. The max number of task allowed for this connector.
4. The Source topic to get records from.
5. :ref:`The KCQL routing querying. <kcql>`
6. The InfluxDB connection URL.
7. The InfluxDB database.

Use the Confluent CLI to view Connects logs.

.. sourcecode:: bash

    # Get the logs from Connect
    confluent log connect

    # Follow logs from Connect
    confluent log connect -f

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/connect-cli ps
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
        connect.influx.username = root
        connect.influx.db = mydb
        connect.influx.password = [hidden]
        connect.influx.url = http://localhost:8086
        connect.influx.retry.interval = 60000
        connect.influx.kcql = INSERT INTO influxMeasure SELECT * FROM influx-topic WITHTIMESTAMP sys_time()
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
      --property value.schema='{"type":"record","name":"User",
      "fields":[{"name":"company","type":"string"},{"name":"address","type":"string"},{"name":"latitude","type":"float"},{"name":"longitude","type":"float"}]}'

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

1.  Topic to index mapping.
3.  Auto mapping of the Kafka topic schema to the index.
4.  Field selection
5.  Tagging the data points using constants or fields from the payload


Tag
~~~~

InfluxDB allows via the client API to provide a set of tags (key-value) to each point added.
The current connector version allows you to provide them via the KCQL

.. sourcecode:: bash

    INSERT INTO <measure> SELECT <fields> FROM <source topic> WITHTIMESTAMP <field_name>|sys_time() WITHTAG(field|(constant_key=constant_value))

Example:

.. sourcecode:: sql

    #Tagging using constants
    INSERT INTO measureA SELECT * FROM topicA  WITHTAG (DataMountaineer=awesome, Influx=rulz!)

    #Tagging using fields in the payload. Say we have a Payment structure with these fields: amount, from, to, note
    INSERT INTO measureA SELECT * FROM topicA  WITHTAG (from, to)


    #Tagging using a combination of fields in the payload and constants. Say we have a Payment structure with these fields: amount, from, to, note
    INSERT INTO measureA SELECT * FROM topicA  WITHTAG (from, to, provider=DataMountaineer)


.. note::

    At the moment you can only reference the payload fields but if the structure is nested you can't address nested fields.
    Support for such functionality will be provided soon. You can't tag with fields present in the Kafka message key,
    or use the message metadata(partition, topic, index).


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

This is set in the ``connect.influx.kcql`` option.

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

The length of time the Sink will retry can be controlled by using the ``connect.influx.max.retries`` and the
``connect.influx.retry.interval``.

Configurations
--------------

``connect.influx.kcql``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming. For
InfluxDB it allows either setting a default or selecting a field from the topic as the Point measurement.

* Data type : string
* Importance: high
* Optional  : no

``connect.influx.url``

The InfluxDB database url.

* Data type : string
* Importance: high
* Optional  : no

``connect.influx.db``

The InfluxDB database.

* Data type : string
* Importance: high
* Optional  : no

``connect.influx.username``

The InfluxDB username.

* Data type : string
* Importance: high
* Optional  : yes

``connect.influx.password``

The InfluxDB password.

* Data type : string
* Importance: high
* Optional  : yes

``connect.influx.consistency.level``

Specifies the write consistency. If any write operations do not meet the configured consistency guarantees,
an error will occur and the data will not be indexed. The default consistency-level is ALL.
Other available options are ANY, ONE, QUORUM

* Data type : string
* Importance: medium
* Optional  : yes
* Default   : ALL

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


``connect.influx.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.influx.max.retries``
option. The ``connect.influx.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: medium
* Optional: yes
* Default: RETRY


``connect.influx.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.influx.error.policy`` is set to ``retry``.

* Type: string
* Importance: medium
* Optional: yes
* Default: 10


``connect.influx.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.influx.error.policy`` set to **RETRY**.

* Type: int
* Importance: high
* Optional: no
* Default : 60000 (1 minute)

``connect.progress.enabled``

Enables the output for how many records have been processed.

* Type: boolean
* Importance: medium
* Optional: yes
* Default : false

Example
~~~~~~~

.. sourcecode:: bash

    name=influxdb-sink
    connector.class=com.datamountaineer.streamreactor.connect.influx.InfluxSinkConnector
    tasks.max=1
    topics=influx-topic
    connect.influx.kcql=INSERT INTO influxMeasure SELECT * FROM influx-topic WITHTIMESTAMP sys_time()
    connect.influx.url=http://localhost:8086
    connect.influx.db=mydb

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.

Deployment Guidelines
---------------------

Distributed Mode
~~~~~~~~~~~~~~~~

Connect, in production should be run in distributed mode. 

1.  Install the Confluent Platform on each server that will form your Connect Cluster.
2.  Create a folder on the server called ``plugins/streamreactor/libs``.
3.  Copy into the folder created in step 2 the required connector jars from the stream reactor download.
4.  Edit ``connect-avro-distributed.properties`` in the ``etc/schema-registry`` folder where you installed Confluent
    and uncomment the ``plugin.path`` option. Set it to the path you deployed the stream reactor connector jars
    in step 2.
5.  Start Connect, ``bin/connect-distributed etc/schema-registry/connect-avro-distributed.properties``

Connect Workers are long running processes so set an ``init.d`` or ``systemctl`` service accordingly.

Connector configurations can then be push to any of the workers in the Cluster via the CLI or curl, if using the CLI 
remember to set the location of the Connect worker you are pushing to as it defaults to localhost.

.. sourcecode:: bash

    export KAFKA_CONNECT_REST="http://myserver:myport"

Kubernetes
~~~~~~~~~~

Helm Charts are provided at our `repo <https://datamountaineer.github.io/helm-charts/>`__, add the repo to your Helm instance and install. We recommend using the Landscaper
to manage Helm Values since typically each Connector instance has it's own deployment.

Add the Helm charts to your Helm instance:

.. sourcecode:: bash

    helm repo add datamountaineer https://datamountaineer.github.io/helm-charts/


TroubleShooting
---------------

Please review the :ref:`FAQs <faq>` and join our `slack channel <https://slackpass.io/datamountaineers>`_.


