---
layout: post
title: The Unified Namespace
---

# The Unified Namespace

The Unified Namespace (often abbreviated as UNS) is a concept related to the Industrial Internet of Things (IIoT). This article intends to clarify some concepts about the UNS and draw some general guidelines on how to implement it.

## Some context about the UNS

It's unclear to me where or when the Unified Namespace concept originated. I give credit to Walker Reynolds (from the YouTube channel 4.0 Solutions) and his incredible set of videos for (if not creating, at least) popularizing the term. Very few companies (like HighByte, ThingsBoard and BaseAutomation) mention the Unified Namespace on their blogs and product pages.

When searching for "Unified Name Space" or "Unified Namespace" on Google (at the moment of writing, December 2021), it returns no more than 11,800 results, which is quite low considering the amount of results other related searches bring ("IIoT" produces 10.7 million results).

If the UNS lies at the core of the 4th Industrial Revolution, as Walker and other few people claim, then it is quite a mystery why it is not more spread.

I personally find the concept quite interesting and useful. When correctly implemented, it can make handling lots of data and data sources easier. This article is just my personal take on the UNS. It remains to be seen whether it becomes more popular (or not) with time.

## What does the UNS contain?

The word Namespace in computer science is generally means "a set of names used to identify objects of various kinds" (definition from Wikipedia). So, as the term suggest, the UNS tries to bring together different objects in the factory or company, regardless of their source, each identified by a unique key.

The UNS proves useful because companies (even small ones) always have tons of data sources of different nature. They can be files, databases, data from sensors, PLCs and machines, data from clients, providers, or other services, and many more. Most tasks performed by engineers and other staff require some action over the data, and much time is lost only searching for the right source.

So the UNS contains, well, data. Like a file system in a computer (which is also a namespace, by the Wikipedia definition), the UNS stores "files" (data) into different "folders" (keys).

## Key structure

Keys in the namespace can, like folders. take a hierarchical structure. This is beneficial not only because it keeps things organized and neat (making it easier to find keys), but because it also makes access control easier. In a tree-like structure, an administrator can assign users (people and applications that can interact with the namespace) to a certain level in the hierarchy, allowing them to see only keys in that level or below. For example, users in one department may only see keys related to their department, while a manager may see keys related to all departments. 

Since not much has been written about the namespace, there are not many recommendations for a general structure. The most accepted proposal is following the guidelines of the ISA-95 standard, a set of terminology standards defined for better communications between manufacturers and suppliers. This standard inherits a previous set of names (from the ISA-88 standard) that is usually used to describe a certain process or unit within an production chain. In general, the structure looks as follows: `[enterprise]/[site]/[area]/[line]/[cell]/[module]` (names for each level can change, a good explanation can be found at https://www.plcacademy.com/isa-88-s88-batch-control-explained/). A key in this structure can identify a single piece of equipment (module), within a machine (cell), on a production line, in an area (division) of a certain facility (site) that belongs to a company (enterprise). 

Naturally, this naming convention is too specific to be applied to any company, specially if it's not in the manufacturing industry. However, it is a good starting point to create a tailored key structure for each use case.

## The UNS in the context of traditional industrial control

In the traditional industrial control model, data flows from the device closest to the machine (sensors and controllers) up to the SCADA (supervision system), then the MES (scheduling and execution system) and then the ERP (business planning and resource administration system). ERPs are usually the ultimate hubs, where all data is collected, analyzed and stored. Decisions taken at this level are then passed to the other systems to adjust the production process.

The general inconvenience with these systems is the reduced interoperability between them, resulting in data silos (data that can't be accessed easily from outside the system) and communication problems.

The Unified Namespace, in a way, removes the boundaries between these systems and puts each data source in an accessible structure. This way, systems have the data readily available without needing a middleman. Also, adding new systems and sources can be done without modifying already working nodes in the structure. 

This "closed loop", where data flows back and forth between systems and sources at every level of the logical structure, is the first and most important challenge to solve in automation. After that, we can focus our efforts on designing better algorithms to make sense of the data and make people's work easier.

## Where is the Namespace? Implementation ideas

So far, the Namespace has been described as just the collection of hierarchical keys that point to certain piece of data within a company. In general, we want a way for different users and applications to share data using this set of keys.

In general, we don't want the Namespace data to exist in a specific location (as in a database). We want a way to access resources from different locations, so the Namespace would be implemented as a server that _routes_ requests to the corresponding resources.

I like to call the server that does this routing the _Namespace Provider_. The idea of the Namespace Provider has a lot of points in common with MQTT, Apache Kafka and other PUB/SUB protocols. These technologies rely on a _broker_ that delivers messages between different clients (producers and consumers), much like the Namespace Provider delivers data from one resource to a user. MQTT topics can even be structured in a hierarchical way just like the set of keys in the Namespace. This similarities make these protocols very good candidates for an implementation of the Namespace Provider.

Choosing a protocol and architecture depends ultimately on the requirements of each project, and there is no final answer. Regardless of our choice, it's better to stick to a single technology. Otherwise we would have to bridge the different protocols and requests, which is not the point of a Unified Namespace.

We will see some models to understand and implement a Unified Namespace under another title.
