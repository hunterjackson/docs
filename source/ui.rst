.. _ui:

Fast Data UI's (Landoop)
========================

Landoop has a number of UI's available to visually data in Kafka and schemas in the Schema Registry.

You can either build from their GitHub repo or install and run the docker images.

*   `Kafka Topics Browser <https://github.com/Landoop/kafka-topics-ui>`__
*   `Schema Registry <https://github.com/Landoop/schema-registry-ui>`__

For docker, pull the images:

.. sourcecode:: bash

    docker pull landoop/kafka-topics-ui
    docker pull landoop/schema-registry-ui

To run

.. sourcecode:: bash

    docker run --rm -it -p 8000:8000 \
           -e "SCHEMAREGISTRY_URL=http://confluent-schema-registry-host:port" \
           landoop/schema-registry-ui

    docker run --rm -it -p 8000:8000 \
            -e "KAFKA_REST_PROXY_URL=http://kafka-rest-proxy-host:port" \
               landoop/kafka-topics-ui

    #Start both in one docker
    #docker run --rm -it -p 8000:8000 \
    #       -e "SCHEMAREGISTRY_UI_URL=http://confluent-schema-registry-host:port" \
    #       -e "KAFKA_REST_PROXY_URL=http://kafka-rest-proxy-host:port" \
    #       landoop/kafka-topics-ui


Your schema-registry service will need to allow CORS (!!)

To do that, and in ``/opt/confluent-3.0.0/etc/schema-registry/schema-registry.properties``

.. sourcecode:: bash

    access.control.allow.methods=GET,POST,OPTIONS
    access.control.allow.origin=*

.. figure:: ../images/landoop-topic-1.png
    :alt:

.. figure:: ../images/landoop-topic-2.png
    :alt:

.. figure:: ../images/landoop-schema.gif
    :alt:
