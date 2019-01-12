---
title: 'The Side Effect of Fetching Kafka Topic Metadata'
excerpt: "Fetching Kafka topic metadata has a side effect that people often overlook. We reproduce it with code examples using kafka-junit and conclude with a very easy to remember note as a reminder."
categories:
  - Technology Tips
tags:
  - Kafka
  - Topic Creation
  - Metadata
---

Recently I have encountered a weird problem where a script for creating Kafka topics stopped working and reported an error of "Topic 'Kafka' already exists" instead. What makes it harder to understand is that I have added existence check in the script before creating the topic and I am pretty sure there are no other scripts running at all that could possibly create the same topic. How come the topic be created then and even if the topic is created, how could the existence check fail to detect that? Think about it before you continue.

As you might have guessed, the Kafka broker config "auto.create.topics.enable" is set to be true in this case. So the topic is created automatically rather than by any other scripts and this could happen just after the existence check but before the script starts creating the topic. 

## Further Investigation

Problem solved? Wait for a second again, the other fact is that while the script is running, I make sure nothing writes to or reads from the topic, what triggers the auto topic creation? Think about this again and I will give the answers below.

The trigger is the metadata fetching and the auto topic creation is exactly the side effect of the metadata fetching. It turns out that there is another service running and fetches partitionInfo metadata for the given topic and that causes the topic to be created automatically. This side effect of metadata fetching is really something that I believe a lot of people would overlook (I am one of those!). And I really want to poit it out here as a good reminder for both myself and others. 

## Reproduce the Problem

Let's take a look an experimental example with more details here.

Here I am using the [kafka-junit](https://github.com/salesforce/kafka-junit/tree/master/kafka-junit4 "A library that wraps Apache Kafka's KafkaServerStartable class") library to start a local Kafka broker and it has auto topic creation enabled. The demo test code:

```java
package com.techpoolx.kakfa;

import java.util.UUID;

import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.junit.ClassRule;
import org.junit.Test;

import com.salesforce.kafka.test.junit4.SharedKafkaTestResource;

public class MainTest {

    @ClassRule
    public static final SharedKafkaTestResource kafka = new SharedKafkaTestResource();
    
    @Test
    public void testFetchingPartitionInfos() {
    	try (KafkaConsumer<String, String> consumer = 
    			kafka.getKafkaTestUtils().getKafkaConsumer(
    					StringDeserializer.class,
    					StringDeserializer.class)) {
    		String topic = "topic_foo_" + UUID.randomUUID();

    		printOutAllTopics();
    		System.out.println("Partitions for topic by listTopics():");
    		System.out.println(consumer.listTopics().get(topic));

    		printOutAllTopics();
    		System.out.println("Partitions for topic by partitonsFor():");
    		System.out.println(consumer.partitionsFor(topic));

    		printOutAllTopics();
    		System.out.println("Partitions for topic by listTopics():");
    		System.out.println(consumer.listTopics().get(topic));
    		
    		printOutAllTopics();
    	}
    }

    private void printOutAllTopics() {
    	System.out.println();
    	System.out.println("All Topics:");
    	System.out.println(kafka.getKafkaTestUtils().getTopicNames());
    	System.out.println();
    }
}
```

The Java example here is fetching partition metadata for a non-existing topic in two different ways using two similar methods. And the following are the actual output:

```java
All Topics:
[]

Partitions for topic by listTopics():
null

All Topics:
[]

Partitions for topic by partitonsFor():
[Partition(topic = topic_foo_33fc52aa-a7e6-45ea-bff6-55eb88643fe1, partition = 0, leader = 1, replicas = [1], isr = [1], offlineReplicas = [])]

All Topics:
[topic_foo_33fc52aa-a7e6-45ea-bff6-55eb88643fe1]

Partitions for topic by listTopics():
[Partition(topic = topic_foo_33fc52aa-a7e6-45ea-bff6-55eb88643fe1, partition = 0, leader = 1, replicas = [1], isr = [1], offlineReplicas = [])]

All Topics:
[topic_foo_33fc52aa-a7e6-45ea-bff6-55eb88643fe1]
```

The result clearly shows a non-existing topic is automatically created after we try to fetch partition metadata for it. The other method listTopics() does not have this side effect though. This makes sense because listTopics() does not have any specific topic names, so there is nothing specific to create. 

## A Simple And Easy to Remember Note 
So lessons learned! To summarize:

If auto topic creation is enabled for Kafka brokers, whenever a Kafka broker sees a specific topic name, that topic will be created if it is not already existing. This includes when writing data to, reading data from and fetching metadata for the topic.
{: .notice}

To help keep this in your mind, compare the following two similar methods again:

```java
/**
 * This will not auto-create topics as there is nothing specific to create
 */
KafkaConsumer#listTopics() 

/**
 * Assuming auto.create.topics.enable is set to be true. The following method will 
 * automatically create the given topic if the topic does not exist.
 */
KafkaConsumer#partitionsFor(String topic)
```

Do you have any other interesting experience of using Kafka? Please feel free to share and leave your comments.

