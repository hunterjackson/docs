.. StreamReactor documentation master file, created by Andrew Stevenson

Stream Reactor
==============

The Stream Reactor is a set of components to build a reference architecture for streaming data platforms. At its core
is Kafka, with Kafka Connect providing a unified way to stream data in and out of the system.

The actual processing if left to streaming engines and libraries such as Spark Streaming,  Apache Flink, Storm and Kafka
Streams.

DataMountaineer provides a range of supporting components to the main technologies, mainly Kafka, Kafka Connect and the
Confluent Platform.

.. figure:: ../images/stream-reactor-1.jpg
   :alt: 

Download `here. <https://github.com/datamountaineer/stream-reactor/releases/tag/v0.2>`__

Components
==========

.. toctree::
   :maxdepth: 1

   install
   connectors
   tools
   socket-streamer
   ui
   faq
