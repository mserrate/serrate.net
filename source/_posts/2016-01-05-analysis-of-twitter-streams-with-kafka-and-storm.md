---
title: Analysis of twitter streams with Kafka and Storm
date: 2016-01-05T02:24:07+00:00
thumbnail: /files/2016/02/TwitterStreaming.png
categories:
  - Architecture
tags:
  - BigData
  - Cassandra
  - Kafka
  - Storm
---
Following my last [post](http://www.serrate.net/2015/12/13/big-data-streams-and-lambdas/), I will present a real-time processing sample with Kafka and Storm using the Twitter Streaming API.

## Overview

{% img /files/2016/02/TwitterStreaming.png 524 247 '"Twitter Streaming"' '"Twitter Streaming"' %}

The solution consists of the following:

  * <a href="https://github.com/mserrate/twitter-streaming-app/tree/master/twitter-kafka-producer" target="_blank">twitter-kafka-producer</a>: A very basic producer that reads tweets from the Twitter Streaming API and stores them in Kafka. 
  * <a href="https://github.com/mserrate/twitter-streaming-app/tree/master/twitter-storm-topology" target="_blank">twitter-storm-topology</a>: A Storm topology that reads tweets from Kafka and, after applying filtering and sanitization, process the messages in parallel for: 
      * **Sentiment Analysis:** Using a sentiment analysis algorithm to classify the tweet into a positive or negative feeling.
      * **Top Hashtags:** Calculates the top 20 hashtags using a sliding window.


### Storm Topology

{% img /files/2016/02/TwitterTopology.png 577 180 '"Twitter Topology"' '"Twitter Topology"' %}

The Storm topology consist of the following elements:

  * **<a href="https://github.com/apache/storm/tree/master/external/storm-kafka" target="_blank">Kafka Spout</a>:** The spout implementation to read messages from Kafka. 

  * **<a href="https://github.com/mserrate/twitter-streaming-app/blob/master/twitter-storm-topology/src/main/java/bolts/TwitterFilterBolt.java" target="_blank">Filtering</a>:** Filtering out all non-english language tweets. 

  * **<a href="https://github.com/mserrate/twitter-streaming-app/blob/master/twitter-storm-topology/src/main/java/bolts/TextSanitizationBolt.java" target="_blank">Sanitization</a>:** Text normalization in order to be processed properly by the sentiment analysis algorithm.

  * **<a href="https://github.com/mserrate/twitter-streaming-app/blob/master/twitter-storm-topology/src/main/java/bolts/SentimentAnalysisBolt.java" target="_blank">Sentiment Analysis</a>:** The algorithm that analyses word by word the text of the tweet, giving a value between -1 to 1. 

  * **<a href="https://github.com/mserrate/twitter-streaming-app/blob/master/twitter-storm-topology/src/main/java/bolts/SentimentAnalysisToCassandraBolt.java" target="_blank">Sentiment Analysis to Cassandra</a>:** Stores the tweets and its sentiment value in Cassandra.

  * **<a href="https://github.com/mserrate/twitter-streaming-app/blob/master/twitter-storm-topology/src/main/java/bolts/HashtagSplitterBolt.java" target="_blank">Hashtag Splitter</a>:** Splits the different hashtags appearing in a tweet. 

  * **<a href="https://github.com/mserrate/twitter-streaming-app/blob/master/twitter-storm-topology/src/main/java/bolts/HashtagCounterBolt.java" target="_blank">Hashtag Counter</a>:** Counts hashtag occurrences.

  * **<a href="https://github.com/mserrate/twitter-streaming-app/blob/master/twitter-storm-topology/src/main/java/bolts/TopHashtagBolt.java" target="_blank">Top Hashtag</a>:** Does a ranking of the top 20 hashtags given a sliding windows (using the _Tick Tuple_ feature from Storm). 

  * **<a href="https://github.com/mserrate/twitter-streaming-app/blob/master/twitter-storm-topology/src/main/java/bolts/TopHashtagToCassandraBolt.java" target="_blank">Top Hashtag to Cassandra</a>:** Stores the top 20 hashtags in Cassandra.


### Summary

In this post we have seen the benefits of using **Apache Kafka** & **Apache Storm** to ingest and process streams of data, on next posts will look at the implementation details and will provide some analytical insight from the data stored in **Cassandra**.

The sample can be found on Github: <https://github.com/mserrate/twitter-streaming-app>