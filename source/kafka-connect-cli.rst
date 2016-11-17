.. _cli:

Kafka Connect CLI
=================

This is a tiny command line interface (CLI) around the `Kafka Connect REST Interface
<http://docs.confluent.io/3.0.1/connect/userguide.html#rest-interface>`__
to manage connectors. It is used in a git like fashion where the first program argument indicates the command: it can be one of
``[ps|get|rm|create|run|status|plugins|describe|validate|restart|pause|resume]``.

The CLI is meant to behave as a good unix citizen: input from ``stdin``; output to ``stdout``; out of band info to ``stderr`` and non-zero exit
status on error. Commands dealing with configuration expect or produce data in .properties style: ``key=value`` lines and comments start with a
``#``.

The CLI is bundled as part of the :ref:`Stream Reactor <install>` release. It can be found in the ``bin`` folder or download from `here. <https://github.com/datamountaineer/kafka-connect-tools/releases>`__

.. tip::

    You can override the default endpoint by setting an environment variable ``KAFKA_CONNECT_REST``.

    .. sourcecode:: bash

        export KAFKA_CONNECT_REST="http://myserver:myport"


.. sourcecode:: bash

    kafka-connect-cli 0.7
    Usage: kafka-connect-cli [ps|get|rm|create|run|status|plugins|describe|validate|restart|pause|resume] [options] [<connector-name>]

      --help
            prints this usage text
      -e <value> | --endpoint <value>
            Kafka Connect REST URL, default is http://localhost:8083/

    Command: ps
    list active connectors names.

    Command: get
    get the configuration of the specified connector.

    Command: rm
    remove the specified connector.

    Command: create
    create the specified connector with the .properties from stdin; the connector cannot already exist.

    Command: run
    create or update the specified connector with the .properties from stdin.

    Command: status
    get connector and it's task(s) state(s).

      <connector-name>
            connector name

    Command: plugins
    list the available connector class plugins on the classpath.

    Command: describe
    list the configurations for a connector class plugin on the classpath.

    Command: validate
    validate the connector properties from stdin against a connector class plugin on the classpath.
        <class-name (FQDN)>

    Command: restart
     restart the specified connector.

    Command: pause
    pause the specified connector.

    Command: resume
    resume the specified connector.

Usage
-----

Get Active Connectors
~~~~~~~~~~~~~~~~~~~~~

Command: ``ps``

.. sourcecode:: bash

    $ ./cli ps
    twitter-source

Get Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Command: ``get``

.. sourcecode:: bash

    $ ./cli get twitter-source
    #Connector `twitter-source`:
    name=twitter-source
    tasks.max=1

    (snip)

    track.terms=test
    #task ids: 0

Delete a Connector
~~~~~~~~~~~~~~~~~~

Command: ``rm``

.. sourcecode:: bash

    $ ./cli rm twitter-source

Create a New Connector
~~~~~~~~~~~~~~~~~~~~~~

The connector cannot already exist.

Command: ``create``

.. sourcecode:: bash

    $ ./cli create twitter-source <twitter.properties
    #Connector `twitter-source`:
    name=twitter-source
    tasks.max=1

    (snip)

    track.terms=test
    #task ids: 0

Create or Update a Connector
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Either starts a new connector if it did not exist, or update an existing connector.

Command: ``run``

.. sourcecode:: bash

    $ ./cli run twitter-source <twitter.properties
    #Connector `twitter-source`:
    name=twitter-source
    tasks.max=1

    (snip)

    track.terms=test
    #task ids: 0

Query Connector Status
~~~~~~~~~~~~~~~~~~~~~~

Shows a connector's status and the state of its tasks.

Command: ``status``

.. sourcecode:: bash

    ./cli status my-toy-connector
    connectorState: RUNNING
    numberOfTasks: 3
    tasks:
      - taskId: 0
        taskState: RUNNING
      - taskId: 1
        taskState: FAILED
        trace: java.lang.Exception: broken on purpose
        at java.lang.Thread.run(Thread.java:745)
      - taskId: 2
        taskState: FAILED
        trace: java.lang.Exception: broken on purpose
        at java.lang.Thread.run(Thread.java:745)


Check which Plugins are on the Classpath and available in the Connect Cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

 Shows which Connector classes are available on the classpath.

Command: ``plugins``

.. sourcecode:: bash

    ./cli plugins
    Class name: com.datamountaineeer.streamreactor.connect.blockchain.source.BlockchainSourceConnector
    Class name: com.datamountaineer.streamreactor.connect.elastic.ElasticSinkConnector
    Class name: com.datamountaineer.streamreactor.connect.druid.DruidSinkConnector
    Class name: io.confluent.connect.hdfs.HdfsSinkConnector
    Class name: io.confluent.connect.jdbc.JdbcSourceConnector
    Class name: com.datamountaineer.streamreactor.connect.hbase.HbaseSinkConnector
    Class name: org.apache.kafka.connect.file.FileStreamSourceConnector
    Class name: com.datamountaineer.streamreactor.connect.hazelcast.sink.HazelCastSinkConnector
    Class name: com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraSinkConnector
    Class name: com.datamountaineer.streamreactor.connect.rethink.source.ReThinkSourceConnector
    Class name: com.datamountaineer.streamreactor.connect.jms.sink.JMSSinkConnector
    Class name: com.datamountaineer.streamreactor.connect.influx.InfluxSinkConnector
    Class name: com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkConnector
    Class name: com.datamountaineer.streamreactor.connect.bloomberg.BloombergSourceConnector
    Class name: com.datamountaineer.streamreactor.connect.yahoo.source.YahooSourceConnector
    Class name: com.datamountaineer.streamreactor.connect.kudu.sink.KuduSinkConnector
    Class name: org.apache.kafka.connect.file.FileStreamSinkConnector
    Class name: com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector
    Class name: com.datamountaineer.streamreactor.connect.voltdb.VoltSinkConnector
    Class name: com.datamountaineer.streamreactor.connect.mongodb.sink.MongoSinkConnector
    Class name: com.datamountaineer.streamreactor.connect.rethink.sink.ReThinkSinkConnector
    Class name: io.confluent.connect.hdfs.tools.SchemaSourceConnector

Describe the configuration of a Connector
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Describes the configuration parameters for a Connector.

Command: ``describe``

.. sourcecode:: bash

    ./cli describe com.datamountaineer.streamreactor.connect.rethink.sink.ReThinkSinkConnector
    {
      "name": "com.datamountaineer.streamreactor.connect.rethink.sink.ReThinkSinkConnector",
      "error_count": 3,
      "groups": ["Common", "Connection"],
      "configs": [{
        "definition": {
          "name": "connector.class",
          "display_name": "Connector class",
          "importance": "HIGH",
          "order": 2,
          "default_value": "",
          "dependents": [],
          "type": "STRING",
          "required": true,
          "group": "Common"
        },
        "value": {
          "name": "connector.class",
          "recommended_values": [],
          "errors": ["Missing required configuration \"connector.class\" which has no default value."],
          "visible": true
        }
      }, {
    ...........

Validate a Connectors properties file against the required Configurations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Given a properties file for an instance of a Connector validate it against the Connector configuration.

Command: ``validate``

.. sourcecode:: bash

    ./cli validate com.datamountaineer.streamreactor.connect.rethink.sink.ReThinkSinkConnector < ../conf/quickstarts/quickstarts/rethink-sink.properties
    ..............
     "definition": {
            "name": "connect.rethink.sink.port",
            "display_name": "connect.rethink.sink.port",
            "importance": "MEDIUM",
            "order": 3,
            "default_value": "28015",
            "dependents": [],
            "type": "INT",
            "required": false,
            "group": "Connection"
          },
          "value": {
            "name": "connect.rethink.sink.port",
            "visible": true,
            "errors": [],
            "recommended_values": [],
            "value": "28015"
          }
        }]
      }
      Validation failed.
      Missing required configuration "connect.rethink.sink.sink.kcql" which has no default value.]:

Pause a Connector
~~~~~~~~~~~~~~~~~

Command: ``pause``

.. sourcecode:: bash

    ./cli pause cassandra-sink
    Waiting for pause
    connectorState:  RUNNING
    workerId: 10.0.0.9:8083
    numberOfTasks: 1
    tasks:
    - taskId: 0
        taskState: RUNNING
        workerId: 10.0.0.9:8083

Resume a Connector
~~~~~~~~~~~~~~~~~~

Command: ``resume``

.. sourcecode:: bash

    ./cli resume cassandra-sink
    Waiting for resume
    connectorState:  RUNNING
    workerId: 10.0.0.9:8083
    numberOfTasks: 1
    tasks:
     - taskId: 0
       taskState: RUNNING
       workerId: 10.0.0.9:8083

Restart a Connector
~~~~~~~~~~~~~~~~~~~

Command: ``restart``

.. sourcecode:: bash

    ./cli restart cassandra-sink
    Waiting for restart
    connectorState:  RUNNING
    workerId: 10.0.0.9:8083
    numberOfTasks: 1
    tasks:
     - taskId: 0
       taskState: RUNNING
       workerId: 10.0.0.9:8083