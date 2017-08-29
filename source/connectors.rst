Kafka Connect
=============

Kafka Connect is a tool to rapidly stream events in and out of Kafka. It has a narrow focus on data ingress in and egress
out of the central nervous system of modern streaming frameworks. It is not an ETL and this separation of concerns allows
developers to quickly build robust, durable and scalable pipelines in and out of Kafka.

Kafka connect forms an integral component in an ETL pipeline when combined with Kafka and a stream processing framework.

Modes
-----

Kafka Connect can run either as a standalone process for testing and one-off jobs, or as a distributed, scalable, fault
tolerant service supporting an entire organisations. This allows it to scale down to development, testing, and small
production deployments with a low barrier to entry and low operational overhead, and to scale up to support a large
organisations data pipeline.

Usually you'd run in distributed mode to get fault tolerance and better performance. In distributed mode you start
Connect on multiple hosts and they join together to form a cluster. Connectors which are then submitted are distributed
across the cluster.

For workers to join a Connect cluster, set the ``group.id`` in the ``$CONFLUENT_HOME/etc/schema-registry/connect-avro-distributed.properties``
file.

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

Schema Registry Support
-----------------------

DataMountaineer recommends all payloads in Kafka are Avro. Schema Registry provides a serving layer for your metadata.
It provides a RESTful interface for storing and retrieving Avro schemas. It stores a versioned history of all schemas,
provides multiple compatibility settings and allows evolution of schemas according to the configured compatibility setting.
It provides serializers that plug into Kafka clients that handle schema storage and retrieval for Kafka messages that
are sent in the Avro format.

All our Connectors support Avro and use the Confluent provided converters to translate the Avro into Kafka Connects internal
``Struct`` type to determine the schema and how to map onto the target sink store.

We have found some of the clients have already an infrastructure where they publish pure json on the topic and obviously the jump to follow
the best practice and use schema registry is quite an ask. So we offer support for them as well for the following Sinks:

*   ReThinkDB
*   MongoDB
*   InfluxDB
*   DSE Cassandra Sink
*   JMS - TextMessages only
*   CoAP Sink
*   MQTT Sink
*   Elastic2x and 5x Sinks

We are upgrading the remaining Connectors. This allows plain text payloads with a json string.

Connectors
----------

.. toctree::
   :maxdepth: 2

   kcql
   source-connectors
   sink-connectors


