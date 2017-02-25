Kafka Connect Azure DocumentDb Sink
========================

The Azure DocumentDb Sink allows you to write events from Kafka to your DocumentDb instance. The connector converts the Kafka
Connect SinkRecords to DocumentDb Documents and will do an insert or upsert, depending on the configuration you chose. If the database doesn't exist
it can be created automatically - if the configuration flag is set to true (See Configurations section below).
The targeted collections will be created if they don't already exist.

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>` - Topic to measure mapping and Field selection.
2. Schema registry support for Connect/Avro with a schema.
3. Schema registry support for Connect and no schema (schema set to Schema.String)
4. Json payload support, no Schema Registry.
5. Error policies.
6. Schema.Struct and payload Struct, Schema.String and Json payload and Json payload with no schema.

The Sink supports three Kafka payloads type:

**Connect entry with Schema.Struct and payload Struct.** If you follow the best practice while producing the events, each
message should carry its schema information. Best option is to send Avro. Your connect configurations should be set to
``value.converter=io.confluent.connect.avro.AvroConverter``.
You can fnd an example `here <https://github.com/confluentinc/kafka-connect-blog/blob/master/etc/connect-avro-standalone.properties>`__.
To see how easy is to have your producer serialize to Avro have a look at
`this <http://docs.confluent.io/3.0.1/schema-registry/docs/serializer-formatter.html?highlight=kafkaavroserializer>`__.
This requires SchemaRegistry which is open source thanks to Confluent! Alternatively you can send Json + Schema.
In this case your connect configuration should read ``value.converter=org.apache.kafka.connect.json.JsonConverter``.
The difference would be to point your serialization to ``org.apache.kafka.connect.json.JsonSerializer``. This doesn't
require the SchemaRegistry.

**Connect entry with Schema.String and payload json String.** Sometimes the producer would find it easier, despite sending
Avro to produce a GenericRecord, to just send a message with Schema.String and the json string.

**Connect entry without a schema and the payload json String.** There are many existing systems which are publishing json
over Kafka and bringing them in line with best practices is quite a challenge. Hence we added the support

Prerequisites
-------------

-  Azure DocumentDb instance
-  Confluent 3.0.1+
-  Java 1.8
-  Scala 2.11

Setup
-----

Before we can do anything, including the QuickStart we need to install the Confluent platform.
For DocumentDb instance you can either use the emulator provided by Microsoft or provision yourself an instance in Azure.


Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

DocumentDb Setup
~~~~~~~~~~~~~~~

If you already have an instance of Azure DocumentDb running you can skip this step.
Otherwise, please follow `this <https://azure.microsoft.com/en-gb/pricing/details/documentdb/>` to get an Azure account
or use the Emulator.

Sink Connector QuickStart
~~~~~~~~~~~~~~~~~~~~~~~~~

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and updating the configuration. We have developed
a Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

The important configuration for Connect is related to the key and value deserializer. In the first example we default to the
best practice where the source sends Avro messages to a Kafka topic. It is not enough to just be Avro messages but also the producer
must work with the Schema Registry to create the schema if it doesn't exist and set the schema id in the message.
Every message sent will have a magic byte followed by the Avro schema id and then the actual Avro record in binary format.

Here are the entries in the config setting all the above. The are placed in the ``connect-properties`` file Kafka Connect is started with.
Of course if your SchemaRegistry runs on a different machine or you have multiple instances of it you will have to amend the configuration.

.. sourcecode:: bash

    key.converter=io.confluent.connect.avro.AvroConverter
    key.converter.schema.registry.url=http://localhost:8081
    value.converter=io.confluent.connect.avro.AvroConverter
    value.converter.schema.registry.url=http://localhost:8081

Test Database
~~~~~~~~~~~~~

The Sink requires that a database be precreated in DocumentDb.

.. sourcecode:: bash

    #from a new terminal
    ➜

    #list all dbs
    ➜


Starting the Connector
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running the ``start-connect.sh`` script in the ``bin`` folder.

.. sourcecode:: bash

    ➜ bin/start-connect.sh

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for Kudu.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

     ➜  bin/cli.sh create azure-docdb-sink < conf/source.kcql/azure-docdb-sink.properties

    #Connector `azure-docdb-sink`:
    name=azure-docdb-sink
    connector.class=com.datamountaineer.streamreactor.connect.azure.documentdb.sink.DocumentDbSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.documentdb.sink.kcql=INSERT INTO orders SELECT * FROM orders-topic
    connect.documentdb.sink.database=connect
    connect.documentdb.sink.endpoint=localhost:
    connect.documentdb.master.key=C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==
    connect.documentdb.sink.batch.size=10

    #task ids: 0

If you switch back to the terminal you started Kafka Connect in you should see the DocumentDb Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    azure-docdb-sink


.. sourcecode:: bash

        //TODO

Test Records
^^^^^^^^^^^^

.. hint::

    If your input topic doesn't match the target use Kafka Streams to transform in realtime the input. Also checkout the
    `Plumber <https://github.com/rollulus/kafka-streams-plumber>`__, which allows you to inject a Lua script into
    `Kafka Streams <http://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple>`__ to do this,
    no Java or Scala required!

Now we need to put some records it to the orders-topic. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema matches the table created earlier.

.. sourcecode:: bash

    bin/kafka-avro-console-producer \
     --broker-list localhost:9092 --topic orders-topic \
     --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"id","type":"int"},
    {"name":"created", "type": "string"}, {"name":"product", "type": "string"}, {"name":"price", "type": "double"}]}'

Now the producer is waiting for input. Paste in the following (each on a line separately):

.. sourcecode:: bash

    {"id": 1, "created": "2016-05-06 13:53:00", "product": "OP-DAX-P-20150201-95.7", "price": 94.2}
    {"id": 2, "created": "2016-05-06 13:54:00", "product": "OP-DAX-C-20150201-100", "price": 99.5}
    {"id": 3, "created": "2016-05-06 13:55:00", "product": "FU-DATAMOUNTAINEER-20150201-100", "price": 10000}
    {"id": 4, "created": "2016-05-06 13:56:00", "product": "FU-KOSPI-C-20150201-100", "price": 150}

Now if we check the logs of the connector we should see 2 records being inserted to DocumentDB:

.. sourcecode:: bash

    //TODO

.. sourcecode:: bash

    #Verify the records were inserted
    ➜   //TODO
        >
        { "_id" : ObjectId("581fb21b09690a24b63b35bd"), "id" : 1, "created" : "2016-05-06 13:53:00", "product" : "OP-DAX-P-20150201-95.7", "price" : 94.2 }
        { "_id" : ObjectId("581fb2f809690a24b63b35c2"), "id" : 2, "created" : "2016-05-06 13:54:00", "product" : "OP-DAX-C-20150201-100", "price" : 99.5 }
        { "_id" : ObjectId("581fb2f809690a24b63b35c3"), "id" : 3, "created" : "2016-05-06 13:55:00", "product" : "FU-DATAMOUNTAINEER-20150201-100", "price" : 10000 }
        { "_id" : ObjectId("581fb2f809690a24b63b35c4"), "id" : 4, "created" : "2016-05-06 13:56:00", "product" : "FU-KOSPI-C-20150201-100", "price" : 150 }


Bingo, our 4 rows!


Legacy topics (plain text payload with a json string)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We have found some of the clients have already an infrastructure where they publish pure json on the topic and obviously the jump to follow
the best practice and use schema registry is quite an ask. So we offer support for them as well.

This time we need to start the connect with a different set of settings.

.. sourcecode:: bash

      #create a new configuration for connect
      ➜ cp  etc/schema-registry/connect-avro-distributed.properties etc/schema-registry/connect-avro-distributed-json.properties
      ➜ vi etc/schema-registry/connect-avro-distributed-json.properties

Replace the following 4 entries in the config

.. sourcecode:: bash

      key.converter=io.confluent.connect.avro.AvroConverter
      key.converter.schema.registry.url=http://localhost:8081
      value.converter=io.confluent.connect.avro.AvroConverter
      value.converter.schema.registry.url=http://localhost:8081

with the following

.. sourcecode:: bash

      key.converter=org.apache.kafka.connect.json.JsonConverter
      key.converter.schemas.enable=false
      value.converter=org.apache.kafka.connect.json.JsonConverter
      value.converter.schemas.enable=false

Now let's restart the connect instance:

.. sourcecode:: bash

      #start a new instance of connect
      ➜   $bin/start-connect.sh


Use the ``CLI`` to remove the old DocumentDb Sink:

.. sourcecode:: bash

    ➜ bin/cli.sh rm  azure-docdb-sink

and start the new sink with the json properties files to read from the a different topic with json as the payload.

.. sourcecode:: bash

     #start the connector for DocumentDb
    ➜   bin/cli.sh create azure-docdb-sink-json < azure-docdb-sink-json.properties

You should see in the terminal where you started Kafka Connect the following entries in the log:

.. sourcecode:: bash

        //TODO

Now it's time to produce some records. This time we will use the simple kafka-consoler-consumer to put simple json on the topic:

.. sourcecode:: bash

    ➜ ${CONFLUENT_HOME}/bin/kafka-console-producer --broker-list localhost:9092 --topic orders-topic-json

    {"id": 1, "created": "2016-05-06 13:53:00", "product": "OP-DAX-P-20150201-95.7", "price": 94.2}
    {"id": 2, "created": "2016-05-06 13:54:00", "product": "OP-DAX-C-20150201-100", "price": 99.5}
    {"id": 3, "created": "2016-05-06 13:55:00", "product": "FU-DATAMOUNTAINEER-20150201-100", "price":10000}

Following the command you should have something similar to this in the logs for your connect:

.. sourcecode:: bash

    //TODO

Let's check the DocumentDb database for the new records:

.. sourcecode:: bash

    #
    ➜
        >  //TODO
        { "_id" : ObjectId("581fc5fe53b2c9318a3c1004"), "created" : "2016-05-06 13:53:00", "id" : NumberLong(1), "product_name" : "OP-DAX-P-20150201-95.7", "value" : 94.2 }
        { "_id" : ObjectId("581fc5fe53b2c9318a3c1005"), "created" : "2016-05-06 13:54:00", "id" : NumberLong(2), "product_name" : "OP-DAX-C-20150201-100", "value" : 99.5 }
        { "_id" : ObjectId("581fc5fe53b2c9318a3c1006"), "created" : "2016-05-06 13:55:00", "id" : NumberLong(3), "product_name" : "FU-DATAMOUNTAINEER-20150201-100", "value" : NumberLong(10000) }


Bingo, our 3 rows!

Features
--------

The sink connector will translate the SinkRecords to json and will insert each one in the database. We support to insert modes:
INSERT and UPSERT. All of this can be expressed via KCQL (our own SQL like syntax for configuration. Please see below the section
for Kafka Connect Query Language)

The sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to have choose selection of fields
   or all fields written to DocumentDb.
2. Topic to table routing. Your sink instance can be configured to handle multiple topics and collections. All you have to do is to set
   your configuration appropriately. Below you will find an example

.. sourcecode:: bash

    connect.documentdb.sink.kcql = INSERT INTO orders SELECT * FROM orders-topic; UPSERT INTO customers SELECT * FROM customer-topic PK customer_id; UPSERT INTO invoiceid as invoice_id, customerid as customer_id, value a SELECT invoice_id, FROM invoice-topic

3. Error policies for handling failures.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L**, :ref:`KCQL <kcql>` allows for routing and mapping using a SQL like syntax,
consolidating typically features in to one configuration option.

The sink supports the following:

.. sourcecode:: bash

    INSERT INTO <database>.<target collection> SELECT <fields> FROM <source topic> <PK field name>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO collectionA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB with primary key as the field id from the topic
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB PK id


Error Polices
~~~~~~~~~~~~~

The sink has three error policies that determine how failed writes to the target database are handled. The error policies
affect the behaviour of the schema evolution characteristics of the sink. See the schema evolution section for more
information.

**Throw**

Any error on write to the target database will be propagated up and processing is stopped. This is the default
behaviour.

**Noop**

Any error on write to the target database is ignored and processing continues.

.. warning::

    This can lead to missed errors if you don't have adequate monitoring. Data is not lost as it's still in Kafka
    subject to Kafka's retention policy. The sink currently does **not** distinguish between integrity constraint
    violations and or other exceptions thrown by drivers..

**Retry**

Any error on write to the target database causes the RetryIterable exception to be thrown. This causes the
Kafka connect framework to pause and replay the message. Offsets are not committed. For example, if the database is offline
it will cause a write failure, the message can be replayed. With the Retry policy the issue can be fixed without stopping
the sink.

The length of time the sink will retry can be controlled by using the ``connect.documentdb.sink.max.retires`` and the
``connect.documentdb.sink.retry.interval``.

Topic Routing
^^^^^^^^^^^^^

The sink supports topic routing that maps the messages from topics to a specific collection. For example map
a topic called "bloomberg_prices" to a collection called "prices". This mapping is set in the ``connect.documentdb.kcql`` option.
You don't need to set up multiple sinks for each topic or collection. The same sink instance can be configured to handle multiple collections.
For example your configuration in this case:


.. sourcecode:: bash

    connect.documentdb.sink.kcql = INSERT INTO orders SELECT * FROM orders-topic; UPSERT INTO customers SELECT * FROM customer-topic PK customer_id; UPSERT INTO invoiceid as invoice_id, customerid as customer_id, value a SELECT invoice_id, FROM invoice-topic

Field Selection
^^^^^^^^^^^^^^^

The sink supports selecting fields from the source topic or selecting all. There is an option to rename a field as well.
All of this can be easily expressed with KCQL:

 -  Select all fields from topic fx_prices and insert into the fx collection: ``INSERT INTO fx SELECT * FROM fx_prices``.

 -  Select all fields from topic fx_prices and upsert into the fx collection, The assumption is there will be a ticker field in the incoming json:
    ``UPSERT INTO fx SELECT * FROM fx_prices PK ticker``.


 -  Select specific fields from the topic sample_topic and insert into the sample collection:
    ``INSERT INTO sample SELECT field1,field2,field3 FROM sample_topic``.

 -  Select specific fields from the topic sample_topic and upsert into the sample collection:
    ``UPSERT INTO sample SELECT field1,field2,field3 FROM sample_fopic PK field1``.

 -  Rename some fields while selecting all from the topic sample_topic and insert into the sample collection:
    ``INSERT INTO sample SELECT *, field1 as new_name1,field2 as new_name2 FROM sample_topic``.

 -  Rename some fields while selecting all from the topic sample_topic and upsert into the sample collection:
    ``UPSERT INTO sample SELECT *, field1 as new_name1,field2 as new_name2 FROM sample_topic PK new_name1``.

 -  Select specific fields and rename some of them from the topic sample_topic and insert into the sample collection:
    ``INSERT INTO sample SELECT field1 as new_name1,field2, field3 as new_name3 FROM sample_topic``.

 -  Select specific fields and rename some of them from the topic sample_topic and upsert into the sample collection:
    ``INSERT INTO sample SELECT field1 as new_name1,field2, field3 as new_name3 FROM sample_fopic PK new_name3``.


Configurations
--------------

Configurations parameters:

``connect.documentdb.sink.database``

The Azure DocumentDb target database.

* Data type: string
* Optional : no

``connect.documentdb.endpoint``

The service endpoint to use to create the client.

* Data type: string
* Optional : no

``connect.documentdb.master.key``

The connection master key

* Data type: string
* Optional : no

``connect.documentdb.sink.consistency.level``

Determines the write visibility. There are four possible values: Strong,BoundedStaleness,Session or Eventual

* Data type: string
* Optional : yes
* Default  : Session


``connect.documentdb.sink.database.create``

If set to true it will create the database if it doesn't exist. If this is set to default(false) an exception will be raised

* Data type: Boolean
* Optional : true
* Default  : false

``connect.documentdb.proxy``

Specifies the connection proxy details.

* Data type: String
* Optional : yes



``connect.documentdb.batch.size``

The number of records the sink would push to DocumentDb at once (improved performance)

* Data type: int
* Optional : yes
* Default: 100

``connect.documentdb.kcql``

Kafka connect query language expression. Allows for expressive topic to collectionrouting, field selection and renaming.

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT field1, field2, field3 as renamedField FROM TOPIC2


* Data Type: string
* Optional : no

``connect.documentdb.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **NOOP**, the error is swallowed, **THROW**, the error is allowed to propagate and retry.
For **RETRY** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.documentdb.max.retires``
option. The ``connect.documentdb.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high
* Default: ``throw``

``connect.documentdb.max.retires``

The maximum number of times a message is retried. Only valid when the ``connect.documentdb.error.policy`` is set to ``TRHOW``.

* Type: string
* Importance: high
* Default: 10

``connect.documentdb.retry.interval``

The interval, in milliseconds between retries if the sink is using ``connect.documentdb.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Default : 60000 (1 minute)

Example
~~~~~~~

.. sourcecode:: bash

    name=azure-docdb-sink
    connector.class=com.datamountaineer.streamreactor.connect.azure.documentdb.sink.DocumentDbSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.documentdb.sink.kcql=INSERT INTO orders SELECT * FROM orders-topic
    connect.documentdb.sink.database=connect
    connect.documentdb.sink.endpoint=localhost:
    connect.documentdb.master.key=C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==
    connect.documentdb.sink.batch.size=10

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal or fields,
data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules. More information
can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.


Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
