.. _python:

.. toctree::
    :maxdepth: 2

Python Schema Client & Serializers and Deserializers
====================================================

DataMountaineer recommends all payloads in Kafka are Avro. Schema Registry provides a serving layer for your metadata.
It provides a RESTful interface for storing and retrieving Avro schemas. It stores a versioned history of all schemas,
provides multiple compatibility settings and allows evolution of schemas according to the configured compatibility setting.
It provides serializers that plug into Kafka clients that handle schema storage and retrieval for Kafka messages that
are sent in the Avro format.

Confluent have provided a `Python <https://github.com/confluentinc/confluent-kafka-python>`__ client built
on `librdkafka <https://github.com/edenhill/librdkafka>`__. To integrated with the Schema Registry we need to encode/decode using
the `schema_id`. This fork is from `Versign <https://github.com/verisign/python-confluent-schemaregistry>`__, we upgraded it to
Python 3.5 for a client.

Install
-------

Run ``setup.py`` from the source root

.. sourcecode:: bash

    python setup.py install

or via pip

.. sourcecode:: bash

    pip3 install datamountaineer-schemaregistry


Usage
-----

.. sourcecode:: python

    from datamountaineer.schemaregistry.client import SchemaRegistryClient
    from datamountaineer.schemaregistry.serializers import MessageSerializer, Util

    # Initialize the client
    client = SchemaRegistryClient(url='http://registry.host')
    # encode/decode a record to put onto kafka
    serializer = MessageSerializer(client)


Schema Operations
~~~~~~~~~~~~~~~~~

.. sourcecode:: python

    # register a schema for a subject
    schema_id = client.register('my_subject', avro_schema)

    # fetch a schema by ID
    avro_schema = client.get_by_id(schema_id)

    # get the latest schema info for a subject
    schema_id,avro_schema,schema_version = client.get_latest_schema('my_subject')

    # get the version of a schema
    schema_version = client.get_version('my_subject', avro_schema)

    # Compatibility tests
    is_compatible = client.test_compatibility('my_subject', another_schema)

    # One of NONE, FULL, FORWARD, BACKWARD
    new_level = client.update_compatibility('NONE','my_subject')
    current_level = client.get_compatibility('my_subject')


Encoding to write back to Kafka. Encoding by id is the most efficent as it avoids an extra trip to the Schema Registry to
lookup the schema id.


Writing Messages
~~~~~~~~~~~~~~~~

Encode by schema only.

.. sourcecode:: python

    # use an existing schema and topic
    # this will register the schema to the right subject based
    # on the topic name and then serialize
    encoded = serializer.encode_record_with_schema('my_topic', avro_schema, record)

    from confluent_kafka import Producer

    p = Producer({'bootstrap.servers': 'mybroker,mybroker2'})
    p.produce('my_topic', encoded)
    p.flush()


Reading Messages
~~~~~~~~~~~~~~~~

.. sourcecode:: python

    # decode a message from kafka

    from confluent_kafka import Consumer, KafkaError

    c = Consumer({'bootstrap.servers': 'mybroker', 'group.id': 'mygroup',
                  'default.topic.config': {'auto.offset.reset': 'smallest'}})
    c.subscribe(['my_topic'])
    running = True
    while running:
        msg = c.poll()
        if not msg.error():
            decoded = serializer.decode_message(message)
        elif msg.error().code() != KafkaError._PARTITION_EOF:
            print(msg.error())
            running = False
    c.close()
