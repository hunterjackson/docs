Kafka Connect Elastic
=====================

A Connector and Sink to write events from Kafka to Elastic Search using `Elastic4s <https://github.com/sksamuel/elastic4s>`__ client.
The connector converts the value from the Kafka Connect SinkRecords to Json and uses Elastic4s's JSON insert functionality to index.

The Sink creates an Index and Type corresponding to the topic name and uses the JSON insert functionality from Elastic4s.

The Sink supports:

1. Auto index creation at start up.
2. :ref:`The KCQL routing querying <kcql>` - Topic to index mapping and Field selection.
3. Auto mapping of the Kafka topic schema to the index.

Prerequisites
-------------

- Confluent 3.3
- Elastic Search 2.2
- Java 1.8
- Scala 2.11

Setup
-----

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Elastic Setup
~~~~~~~~~~~~~

Download and start Elastic search.

.. sourcecode:: bash

    curl -L -O https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.2.0/elasticsearch-2.2.0.tar.gz
    tar -xvf elasticsearch-2.2.0.tar.gz
    cd elasticsearch-2.2.0/bin
    ./elasticsearch --cluster.name elasticsearch


Sink Connector QuickStart
-------------------------

We you start the Confluent Platform, Kafka Connect is started in distributed mode (``confluent start``). 
In this mode a Rest Endpoint on port ``8083`` is exposed to accept connector configurations. 
We developed Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor and Confluent. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Once the Connect has started we can now use the kafka-connect-tools :ref:`cli <kafka-connect-cli>` to post in our distributed properties file for Elastic.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/connect-cli create elastic-sink < conf/elastic-sink.properties

    #Connector name=`elastic-sink`
    connect.progress.enabled=true
    name=elastic-sink
    connector.class=com.datamountaineer.streamreactor.connect.elastic.ElasticSinkConnector
    connect.elastic.url=localhost:9300
    connect.elastic.cluster.name=elasticsearch
    tasks.max=1
    topics=orders-topic
    connect.elastic.kcql=INSERT INTO index_1 SELECT * FROM orders-topic
    #task ids: 0

The ``elastic-sink.properties`` file defines:

1. The name of the connector.
2. The class containing the connector.
3. The name of the cluster on the Elastic Search server to connect to.
4. The max number of task allowed for this connector.
5. The Source topic to get records from.
6. :ref:`The KCQL routing querying. <kcql>`

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
    elastic-sink

.. sourcecode:: bash

    [2016-05-08 20:56:52,241] INFO

        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
           ________           __  _      _____ _       __
          / ____/ /___ ______/ /_(_)____/ ___/(_)___  / /__
         / __/ / / __ `/ ___/ __/ / ___/\__ \/ / __ \/ //_/
        / /___/ / /_/ (__  ) /_/ / /__ ___/ / / / / / ,<
       /_____/_/\__,_/____/\__/_/\___//____/_/_/ /_/_/|_|


    by Andrew Stevenson
           (com.datamountaineer.streamreactor.connect.elastic.ElasticSinkTask:33)

    [2016-05-08 20:56:52,327] INFO [Hebe] loaded [], sites [] (org.elasticsearch.plugins:149)
    [2016-05-08 20:56:52,765] INFO Initialising Elastic Json writer (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:31)
    [2016-05-08 20:56:52,777] INFO Assigned List(test_table) topics. (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:33)
    [2016-05-08 20:56:52,836] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@69b6b39 finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)

Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``id`` field of type int
and a ``random_field`` of type string.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
     --broker-list localhost:9092 --topic orders-topic \
     --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"id","type":"int"},
    {"name":"random_field", "type": "string"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"id": 999, "random_field": "foo"}
    {"id": 888, "random_field": "bar"}


Check for records in Elastic Search
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now if we check the logs of the connector we should see 2 records being inserted to Elastic Search:

.. sourcecode:: bash

    [2016-05-08 21:02:52,095] INFO Flushing Elastic Sink (com.datamountaineer.streamreactor.connect.elastic.ElasticSinkTask:73)
    [2016-05-08 21:03:52,097] INFO No records received. (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:63)
    [2016-05-08 21:03:52,097] INFO org.apache.kafka.connect.runtime.WorkerSinkTask@69b6b39 Committing offsets (org.apache.kafka.connect.runtime.WorkerSinkTask:187)
    [2016-05-08 21:03:52,097] INFO Flushing Elastic Sink (com.datamountaineer.streamreactor.connect.elastic.ElasticSinkTask:73)
    [2016-05-08 21:04:20,613] INFO Elastic write successful for 2 records! (com.datamountaineer.streamreactor.connect.elastic.ElasticJsonWriter:77)

If we query Elastic Search for ``id`` 999:

.. sourcecode:: bash

    curl -XGET 'http://localhost:9200/INDEX_1/_search?q=id:999'

    {
        "took": 45,
        "timed_out": false,
        "_shards": {
            "total": 5,
            "successful": 5,
            "failed": 0
        },
        "hits": {
            "total": 1,
            "max_score": 1.2231436,
            "hits": [{
                "_index": "INDEX_1",
                "_type": "INDEX_1",
                "_id": "AVMY4eZXFguf2uMZyxjU",
                "_score": 1.2231436,
                "_source": {
                    "id": 999,
                    "random_field": "foo"
                }
            }]
        }
    }

Features
--------

1. Auto index creation at start up.
2. Topic to index mapping.
3. Auto mapping of the Kafka topic schema to the index.
4. Field selection

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`__
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The Elastic Sink supports the following:

.. sourcecode:: bash

    INSERT INTO <index> SELECT <fields> FROM <source topic> 
    [WITHDOCTYPE=<your_document_type>] 
    [WITHINDEXSUFFIX=<your_suffix>]
    [PK field]

`WITHDOCTYPE` allows you to associate a document type to the document inserted.
`WITHINDEXSUFFIX` allows you to specify a suffix to your index and we support date format. All you have to say is '_suffix_{YYYY-MM-dd}'

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to indexA
    INSERT INTO indexA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to indexB
    INSERT INTO indexB SELECT x AS a, y AS b and z AS c FROM topicB PK y

This is set in the ``connect.elastic.kcql`` option.

Auto Index Creation
~~~~~~~~~~~~~~~~~~~

The Sink will automatically create missing indexes at startup. The Sink use elastic4s, more details can be found
`here <https://github.com/sksamuel/elastic4s>`__

Configurations
--------------

``connect.elastic.url``

Url of the Elastic cluster.

* Data Type : string
* Importance: high
* Optional  : no


``connect.elastic.kcql``

Kafka connect query language expression. Allows for expressive table to topic routing, field selection and renaming.

Examples:

.. sourcecode:: sql

    INSERT INTO INDEX_1 SELECT field1, field2 FROM TOPIC1

* Data type : string
* Importance: high
* Optional  : no

``connect.elastic.write.timeout``

Specifies the wait time for pushing the records to ES.

* Data type : long
* Importance: low
* Optional  : yes
* Default   : 300000 (5mins)

``connect.elastic.throw.on.error``

Throws the exception on write failure. Default is 'true'

* Data type : long
* Importance: low
* Optional  : yes
* Default:  : true


``connect.progress.enabled``

Enables the output for how many records have been processed.

* Type: boolean
* Importance: medium
* Optional: yes
* Default : false

Example
~~~~~~~

.. sourcecode:: bash

    name=elastic-sink
    connector.class=com.datamountaineer.streamreactor.connect.elastic.ElasticSinkConnector
    connect.elastic.url=localhost:9300
    connect.elastic.cluster.name=elasticsearch
    tasks.max=1
    topics=test_table
    connect.elastic.kcql=INSERT INTO INDEX_1 SELECT field1, field2 FROM TOPIC1

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
