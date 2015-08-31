---
layout:     post
title:      "Spark2Elasticsearch"
date:       2016-04-26 00:00:00
summary:    "https://github.com/jparkie/Spark2Elasticsearch"
categories: spark scala elasticsearch
---

A few months ago, I was asked to help improve the writing of a lot of data into various [Elasticsearch](https://www.elastic.co/) clusters for different teams using [Apache Spark](http://spark.apache.org/).

First, I found that a lot of the teams were using the official product released by Elasticsearch called [*ES-Hadoop*](https://github.com/elastic/elasticsearch-hadoop). Reading its source code on GitHub, I soon learned that it utilized the REST API. The REST API is a great idea since it provides a wide version compatibility with both `1.X` and `2.X` major-minor versions, and composability with various Hadoop-related tools.

Unfortunately, the REST API was not efficient enough even with various tuning and optimizations both on the client and the server.

As a result, I developed and published a Scala library called *Spark2Elasticsearch* to write large data volumes through Spark to Elasticsearch 2.0 and Elasticsearch 2.1 clusters. The GitHub repository is available at https://github.com/jparkie/Spark2Elasticsearch. Meanwhile, artifacts are cross-published to the Sonatype Central Repository [spark2elasticsearch_2.10](http://mvnrepository.com/artifact/com.github.jparkie/spark2elasticsearch_2.10) and [spark2elasticsearch_2.11](http://mvnrepository.com/artifact/com.github.jparkie/spark2elasticsearch_2.11).

Utilizing the TCP transport layer through the use of the Java API, the library serializes `DataFrame`s to create `UPSERT`s for Elasticsearch. This method provided an order of magnitude increase in performance. Accordingly, I hope this library may help others who are facing similar performance issues.
