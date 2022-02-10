---
layout: post
title: The Connection-Service Model
---

# The Connection-Service Model: an idea for a Unified Namespace implementation

We discussed some ideas about the implementation of the Unified Namespace in a previous post. Here, I will go down somewhat deeper, describing a model I call the Connection-Service Model.

## The need for a Namespace Provider

The Unified Namespace is treated as a collection of key - data source pair. Each key identifies a source of data, without needing to know the specific source (be it a database, a machine, a file, the result of an algorithm, etcetera) or have access to it. The Namespace Provider is the link between keys and data sources. A request to the Namespace Provider should be _routed_ to the corresponding resource, and return the data.

Since data sources are scattered across an organization, we say that the Namespace is _distributed_. We need a Namespace Provider to have access to this distributed data ubiquitously.

## Read and write functions

The Namespace Provider allows users (people, systems, applications, etcetera) to perform two basic tasks:
- Read from the Namespace (with a given key)
- Write to the Namespace (to a given key)

It's important that both operations are consistent. This means, if I write some data to some key, then it's expected that I (or someone else) can read that same data back. 

Data will generally mean a collection of records . Each record will contain a timestamp `t`. Thus, we can talk about the data _since_ $t_0$ and _until_ $t_1$, meaning the collection of records with timestamps between those values. Generally, data in an organization is always "queried" by date. Any question that can be asked (for example, what are the current stock levels, how many items are we producing, what are the sales expectations for next month) are a function of time. This means, we could ask what the stock levels was yesterday, how many items we were producing seven days ago and what the sales expectations were a month ago (for this month). 

The Namespace Provider will thus fundamentally provide data as a function of time. This means that the `read` operation will take (at least) two parameters (apart from the key that we want to read from): _since_ and _until_. The `write` operation will only need the key and the data to write (if the data doesn't have a timestamp, it could automatically set the timestamp to the current time).
- `read(key, since, until)`
- `write(key, data)`

Read and write operations are handled as requests. The user requests to read or write data to the Namespace and the Namespace Provider responds (either with the data requested in the case of read or with a success/error message in the case of write, for example).

## Connections

The Connection-Service Model is a way to understand, represent and implement the different entities that interact with the Namespace. It's called that way because the fundamental entity is the Connection. 

Connections are defined as objects that have two functions, `read` and `write`. The most important connection is the Namespace Connection, that performs read and write requests to the Namespace Provider.

Other data sources are also connections. For example, we can read tags from a PLC or write some value to a tag. We can read from a database o insert new records. We can read and write to a file. An algorithm is something that we can "read" (i.e. perform the calculation and obtain the result), and which has no write method. All these connections are interfaces that allow us to interact with different sources. 

The important thing is that when we "read", we get some data (a single or multiple records) that has timestamps.

The read and write operations can generally have the same parameters as seen above:
- `read(key, since, until)`
- `write(key, data)`

Here, `key` is something more vague. In a PLC, it might mean the tag we're trying to read or write. It might be a table in a database or a sheet in an excel file. The key is not necessarily well defined, but we'll keep it in our read and write operations to make all Connections similar.

As we've seen on some examples, the `write` operation is optional. However, all connections must have a `read` operation.

### Default values for read parameters

Notice that some connections can effectively read data between two timestamps, while some can't. You might be able to read a database and return only the records that are between `since` and `until`, but this is not generally possible. A PLC will only allow you to read the present value of tags, not previous values. 

We will define the default value for the `since` parameter as the "last time the read function was called", and the `until` parameter as the "present moment". When called for the first time, the `read` function should only return the instant, present, value, if it's available to the connection. Subsequent reads should only return data from the last time the read function was executed.

This means that if we call the read function on a loop, using the default parameters, the connection will always return new values. This is equivalent to be constantly polling a server for new data.

### Connections and MQTT

An MQTT client is also a connection, since it can read and write to the broker. The `key` is, naturally, the topic, and the data is the payload. MQTT connections can return only present values. Subscribing to an MQTT topic is equivalent to calling the read function constantly (in any case, calling `read` is equivalent as calling `loop` for the client).

An MQTT connection that has clean session set to false will get all messages from the broker even if it was disconnected for some time. This is equivalent as remembering the "last time read" between sessions.

### Connections that allow extended reading

Only a few connections will allow you to call the `read` function with values for the _since_ and _until_ parameters different than the default ones. We say that these connections allow for *extended reading*, in contrast to connections that allow only for *simple reading*.

Connections that allow for extended reading are typically databases. Some files may be able to return data with timestamps, but you may want to treat them as simple-reading only (for example, if you have an excel file, you may want to read only new rows). Some algorithms can also allow extended reading, performing the calculation on different sets of data depending on the values of _since_ and _until_.

###  Chaining connections

Connections are usually chained to perform a certain result. For example, you can have a database system that, depending on the nature of the data, stores records on different databases. In this case the database system uses two connections to write to each database.

Many complex systems can be broken down to be a simple combination of connections.

###  Reusing connections

Notice that the same connection can be used in different ways in a complex system. For example, a database connection can be used to simply query data, or to perform a complex calculation based on a query on that database (these are two connections chained, first the one that queries the data and then the one doing the calculation). We say that there can be multiple *instances* of the same connection.

A connection can be as complex and have as many functions as desired. The only condition is that all connections must have a `read` method.

### Opening and closing a connection

Generally, connections should also provide a way to open (initiate) and close (terminate) the connection.
- `open()`
- `close()`

We need the connection open when we read, and may never close the connection.

### The Namespace Connection

The most important connection of all is the Namespace Connection. This is the connection that will interact will our Namespace Provider, performing `read` and `write` requests. We will see how this works later on.

## Services

Services are the second core concept of the Connection-Service Model. A service is simply a program that has the following characteristics:
1. It runs constantly (usually in the background)
2. It has (at least) two connections, one of them being an instance of the Namespace Connection, the other being a generic connection (can be any connection, for example, to a PLC) that we call the Main Connection (this is just a generic name, there can be more than one).
3. The two connections may or may not be constantly reading for new values (on a loop).
4. Based on (3), a service may do something based on two possible events:
    - New data from the Main Connection
    - New data from the Namespace Connection
1. A Service may accept *read requests* for a given Namespace key. These requests will execute a `read` on the Main Connection and return the result.

A script that reads PLC data and writes it to the Namespace is an example of a service. 

A program that reads data from the Namespace on certain keys and writes it to a database is also an example of a service. This service would also be able to accept requests for those same keys and return data with variable _since_ and _until_ parameters.

The PLC service described above should not be able to accept read requests, because it can only provide the present value and not historical values. We will understand how this works better when we learn about the inner workings of the Namespace Provider.

The key concept is that, just as the internet is composed of servers, **the Namespace is composed of services**. It is the collection of constantly running services, accepting read requests and writing data, what makes the Namespace exist. This is the reason why we say that the Namespace is _distributed_, because there are likely going to be services running on multiple different machines.

To get data from any Namespace key, we would simple need to identify the service that accepts read requests for that key and make a request. This would require having a routing table for each key in the Namespace. The Namespace Provider handles the routing for every client, much like a gateway does for computers in a network.

If a server can accept read requests, it tells the Namespace Provider on start up that it will be listening to requests on some key, so that the Namespace Provider can, when the case arrives, read the data from the service.

## Applications

Applications are programs that, like services, also use connections. However, applications are not constantly running, and thus cannot accept read requests. They are just stand-alone programs that connect to the Namespace with a Namespace Connection and let the user ultimately read and write to certain keys through a GUI or some other interface.

## The Role of the Namespace Provider

Remember that the Namespace Provider is simply a server that accepts read and write requests. 
- `read(key, since, until)`
- `write(key, data)`

Suppose that we need data from a given key. We make a read request to the Namespace Provider, and expect the data as response. How does the Namespace Provider get the data? When a service starts, it tells the Namespace Provider on which key it accepts read requests. The Namespace Provider then knows where to get the data from, so the read request is forwarded to that service and the service response is then returned to the original user that started the request.

With this logic, the Namespace Provider should also differentiate requests based on the values of _since_ and _until_. If the data that the user wants is fresh (the instant current value), then the Namespace Provider should get the data from the corresponding sensor-reading service, and if the data is historic, it should get it from some database-storage service. 

Fresh data is constantly being required by many users in the Namespace, so instead of requesting data from the plc-reading service, the plc-reading service should _write_ the data to Namespace Provider. The Namespace Provider holds this data on a buffer for as long as it's required (for some time, until the last potential end user is disconnected or reads the data), and uses this buffer to respond to requests. 

Then, when the user requests data from the Namespace Provider, it first checks if it has the data on its buffer (matching key, _since_ and _until_). If it doesn't, then it asks the service that can provide the data, passing the request to the service and then returning the data back to the client.

## What is the model good for?

The model is just a conceptual way of implementing a Unified Namespace. It provides a guideline for implementation, and allows us to understand how current systems could be made better.

Connections made point to point are exactly the opposite of the correct way to implement a Namespace. If the Namespace _is_ the collection of services that can respond to requests, then no services running means no namespace.

Data silos exist because the connections to these silos is not made available through a service to other clients, or the connections don't even exist (remember that all connections must have a `read` method). Much of the work required to create a Namespace consists of writing the required connections for each source present.

Many times we find that we lost or can't access historical data. We say that the Namespace is **complete** if, for every key being written to in the Namespace, there is a service that can accept read requests for that key (and return the required data). Incomplete namespaces are very common. For example, systems that use MQTT can only provide recent data and rely on a database for historical data, but more often than not the database is not connected through a service on the same Namespace (sometimes connections are made directly to the database).

Sometimes the structure for a Namespace exists, but there are so many technologies in place that communicate poorly or not at all with one another. We say that the namespace is **segmented** when there is more than one Namespace Provider, handling different sets of keys. It might be good to have segmented namespaces in some cases (for security and practicality), but in those cases, a **gateway service** must provide complete connection between the two namespaces. Segmented namespaces also mean a lot of unnecessary or redundant code and require quite some effort to make the system function properly.

## Final words

The previous paragraphs were just an outline of the Connection-Service Model for the Unified Namespace. Much more can be said about it, which I will do under another title.