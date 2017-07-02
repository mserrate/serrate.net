---
title: Preview of Kafka Streams
date: 2016-03-15T02:02:29+00:00
thumbnail: /files/2016/03/KafkaStreams-2.png
categories:
  - Architecture
tags:
  - BigData
  - Kafka
  - Storm
---
The preview of **Kafka Streams**, which is one of the main features of the upcoming **Apache Kafka 0.10**, was <a href="http://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple" target="_blank">announced by Jay Kreps this week</a>.

### Kafka joins the Stream Processing club

Kafka Streams is a library to build streaming applications using Kafka topics as input/output. Kafka Streams is in the same league as other streaming systems such as: **Apache Storm**, **Apache Flink** and, not surprisingly, **Apache Samza** which also uses Kafka for input or output of streams.

One of the main advantages is that if you&#8217;re already using Apache Kafka and you need real-time processing, you just need to use the library and you are good to go.
  
Other important features are: **stateful** processing, **windowing** and ability to be deployed using your preferred solution: a simple command line, **Mesos**, **YARN** or **kubernetes** and **docker** if you&#8217;re a container party boy.

### Streams and Tables

One of the key concepts in Kafka Streams is the support of **KStream** and **KTable**.
  
That isn&#8217;t a new concept if you come from the **<a href="http://www.serrate.net/2015/02/17/speaking-at-dotnetspain-conference/" target="_blank">Event Sourcing</a>** world: the KStream is the append-only event store where its state is given by replaying the events from the beginning of time until the last event whereas KTable is the snapshot or projection of the current state of the stream given a point in time.

### Example: Twitter Hashtags Job

{% img /files/2016/03/KafkaStreams-2.png 749 303 '"Kafka Streams"' '"Kafka Streams: KStream and KTable"' %}

### Show me the code!

You can find the complete example here: <a href="https://github.com/mserrate/kafka-streams-app" target="_blank">https://github.com/mserrate/kafka-streams-app</a>

{% codeblock lang:java %}
KStream<String, JsonNode> source = builder.stream(stringDeserializer, jsonDeserializer, "streams-hashtag-input");

KTable<String, Long> counts = source
        .filter(new HashtagFilter())
        .flatMapValues(new HashtagSplitter())
        .map(new HashtagMapper())
        .countByKey(stringSerializer, longSerializer, stringDeserializer, longDeserializer, "Counts");

counts.to("streams-hashtag-count-output", stringSerializer, longSerializer);
{% endcodeblock %}

For this example I&#8217;ve been using a simple TweetProducer who connects to the **Twitter Streaming API** and sends JSON events to a Kafka topic.
  
This topic is read as a KStream and then we begin the process:

  1. Filter out the tweets without hashtags
  2. Apply a flatMapValues (we are just interested in the values, not the keys) to split the different hashtags in a tweet
  3. Apply a map to return a key (hashtag) value (hashtag) as we want to aggregate by hashtag
  4. Aggregate the streams per key (the hashtag) and count them

&nbsp;

Finally we send the KTable to the output queue.