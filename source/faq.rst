.. faq:

FAQS
====

**I have JSON. Can I still use DataMountaineer Sink Connectors?**

Kafka Connect has two converters for both the key and payload from Kafka. These are Json and Avro, the Json converter is
part of the Kafka distribution and the Avro converter from Confluents schema registry. These converters convert the records
in Kafka to SinkRecords, most of our Sinks rely of the data in Kafka being Avro and written with the Confluent Avro Serializers.
This is `best practice`. This allows the Connectors to receive SinkRecords with schemas for the payloads to mapping, filtering
can take place based on the :ref:`The KCQL routing querying <kcql>` provided.

However, it is possible sink Json messages from Kafka with some of our Sinks by using the JsonConverter from Kafka. If your Json messages
have a `schema` field the converter will deliver the records to the Sink with a schema. If no `schema` tag is present the
records will be delivered with a schema of type ``SCHEMA.String``.

We are currently working on support for schemaless json records.


**Can I run on multiple nodes?**

Yes, Kafka Connect has two modes, standalone and distributed. Both allow for scaling by setting the `max.tasks` property.

In distributed mode each work joins a Connect cluster defined in the `etc/schema-registry/connect-avro-distributed.properties`
file that is part of the Confluent distribution. Within this file a property called ``group.id`` controls this.

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

**ClassNotFoundException**

The ``start-connect.sh`` in the Stream Reactor download adds all the jars from the `libs` folder to the CLASSPATH
automatically. If you are not using this start script it is more than likely you have not add the relevant Connector
jar to the CLASSPATH properly.

Explicit add the relevant jar to the classpath and restart Kafka Connect

.. sourcecode:: bash

    export CLASSPATH=my_connector.jar

You can ask a running instance of Kafka Connect what Connector classes are on the classpath with the `CLI <cli>`

.. sourcecode:: bash

    bin/cli loaded


**Guava version**

The Elastic Search and HBase use different versions of guava, as does the Hive libraries supplied by Confluent with the
HDFS Connector. This can cause version clashes.

Explicit add the relevant jar to the classpath and restart Kafka Connect. This sets our jars first and should solve the
issues.

.. sourcecode:: bash

    #For Elastic
    #export CLASSPATH=kafka-connect-elastic-0.2.3-3.0.1.jar

    #For HBASE
    #export CLASSPATH=kafka-connect-hbase-0.2.3-3.0.1.jar


**Redis authentication**

If your redis server is requiring the connection to be authenticated you will need to provide an extra setting:

    .. sourcecode:: bash

        connect.redis.sink.connection.password=$REDIS_PASSWORD

Don't set the value to empty if no password is required.