# Functional Nodes for a Unified Namespace with MQTT

## The need for Functional Nodes

The Unified Namespace is a high-level architectural design pattern that has got a lot of traction in the last few years in the Industrial IoT community [1]. In essence, the UNS conceives all producers and consumers of data in a company structured in a hierarchical way and connected through a single hub. MQTT, a well known pub/sub protocol for IoT, is well suited for this application, as there is a unique broker (hub) connecting different clients, with messages organized in a flexible topic structure (hierarchy).

Being able to support an architectural design pattern, such as the UNS, on a popular, open-source and established protocol, such as MQTT, makes it accessible (read vendor-agnostic), flexible and easy to implement on a wide range of industrial applications. Consider as an analogy the RESTful design pattern for web APIs: it can be implemented on top of HTTP, using different web frameworks in practically any programming language, and it's thus accessible to every developer and can be used for an endless range of applications. The difference is perhaps that RESTful is a more mature and proven design pattern, whereas the UNS definition is still vague and has some practical limitations.

One of such limitations is that MQTT, as a pub/sub mechanism, is intended only for real-time communication. There is no built-in way to read past messages (although there is message retention, and maybe another protocol like Kafka could also cover this gap), or to have a request/response style communication between two nodes (other protocols like OPC-UA also face this issue). For example, if a terminal in a floor shop needs to display a list of all the jobs processed during the day, there is no direct way to relay this information through MQTT from the database or application that holds it. The UNS has sometimes been defined as the _Current State_ of the enterprise [2], meaning that perhaps the UNS is not intended to solve this problem. The definition of _Current State_ however is vague. For the example provided, the list of the jobs processed on the date can be interpreted that way: although the information is not being produced at the same moment a user looks at the screen (or loads the application), it does answer the question of "what's going in the plant at this moment".

This problem has been referred to as the need of _Functional Nodes_. They have been vaguely defined by members in the community in the following way [3]:

> A method or service exposed by a device or system that is mapped as a node or topic in the namespace, and that can be invoked via the UNS broker/hub which will route the request and the results.

A solution to this problem could be implementing a UNS on top of both MQTT and HTTP (maybe with a REST API), to allow for pub/sub and request/response type of communications. This however increases the complexity of any implementation. For starters, there is no ready-to-use HTTP server that provides this capability, as there is a lot of things that need to be manually set-up. There's also the complexity of handling authentication and authorization (access control policies) on both systems. The beauty of MQTT is that you can get started with a minimal set-up.

Another alternative is to constantly publish the data required with some (relatively high) frequency, so that it is always available if a client needs it. The downside is of course heavily increasing the network traffic with unsolicited messages, and that there is no way to specify how to query the data beforehand (different clients may need different query parameters).

Current projects implementing a UNS architecture that face this issue have found a work-around on the software used as _IIoT Platform_. This is, the packaged software product that handles data acquisition, human-machine interface (HMI) creation, historian integration and other aspects required in for an industrial automation project. Ignition [4] is an example of a well known _IIoT Platform_. The issue here is that the solution becomes dependent on how the vendor decides to solve the problem, making the UNS implementation not portable (to another platform) or accessible (to be able to start with a simple set-up) anymore.

## Solving the problem within MQTT

It is sensible to say that MQTT was not meant for any kind of historical or big data analysis. However, it is possible to extend the usage of MQTT to allow for request/response style communication. The size limit of the payload in MQTT is 256MB, which should be more than enough for most applications where a list of "past" data points is needed. Consider using the following idea, where `alv1` is just a prefix to identify the protocol:

- We have a node `N` in our UNS hierarchy-path, for example defined by the ISA95 standard `enterprise/site/area/line/my-node` (we will use `N` for short).
- When new data is produced for that node (say, by `Client Z`), it is published on the MQTT topic `alv1/w/N`. We call this a *Write Message*.
- If `Client A` wants to receive the produced data in real-time, it can subscribe to that same topic (`alv1/w/N`).
- Another `Client B` is connected to the database that holds this data (perhaps this client is also subscribed to `alv/w/N` and stored the data in the database itself). This client is listening to the MQTT topic `alv1/r/N`, and will receive any *Read Message* published on this topic.
- `Client A` can now also query past data available through `Client B`. To do so, it does the following:
    1. `Client A` generates a random number `x` to identify the request.
    2. `Client A` subscribes to the MQTT topic `alv1/x/N`.
    3. `Client A` publishes an MQTT message on `alv1/r/N`, with the number `x` on the payload. Other query parameters could also be included in the payload.
    4. `Client B` is subscribed to `alv1/r/N` and receives the read message. It queries the database with the parameters passed.
    5. `Client B` publishes the query results to `alv1/x/N`.
    6. `Client A` is subscribed to `alv1/x/N` and receives the message. We call this a *Response Message*.
    7. `Client A` unsubscribes from `alv1/x/N`.

This logic allows for both real-time (pub/sub) communication but also request/response communication.

Access Control can also be set up the following way:
- `Client Z` can write to node `N`, so it has permission to publish on `alv1/w/N`.
- `Client A` can receive real-time data from node `N`, so it has permission to subscribe to `alv1/w/N`.
- `Client B` can receive Read Messages on node `N`, so it has permissions to subscribe to `alv1/r/N`.
- `Client B` should also be able to send back a Response Message for every Read Message, so it needs permissions to publish to `alv1/+/N`. Notice the `+` wildcard used. Notice this also covers `alv1/w/N`, which is useful if the same client also sends real-time updates. Otherwise, it needs to be restricted from writing to this topic.
- `Client A` needs to be able to receive the Response Message, so it needs permissions to subscribe to `alv1/+/N`. Notice that this also includes `alv1/w/N`, meaning by default a client with access to node `N` only needs subscription access to `alv1/+/N` for real-time and "historical" data.

The weak point with this structure is that any client can be listening to any Response Message. For example, if `Client A` sends a Read Message for node `N`, any other `Client C`  could intercept the Response Message. However, this only happens if `Client C` has permissions to subscribe to `alv1/+/N`, and in that case it could get the same data from `Client B` (the one responding to the Read Message) directly anyways.

## Extending the Message types

Notice that we use the word Read Message and not Request Message. This is to make it explicit that requests should be only to "GET" data (using the same verbs as in REST). Other kind of requests ("POST", "PUT", "DELETE", ...) means that the data is requested to change, something which cannot be necessarily handled by the same client. If we take a valve position as an example, the client that will receive the command to update the valve position (probably a client connected to the PLC that controls the valve) will be different from the client that can answer for the valve position in the past 24 hours (probably a client connected to the database).

For this kind of situations, we can use a *Command Message* (with a `c` in the topic structure, like `alv1/c/N`). Command Messages are different from Write Messages in the sense that the former are a request for data to change, whereas the latter is an actual event that contains the new data. A Command Message will usually be followed by a Write Message, if the command is successfully executed. A command can be anything requesting a change, like an operator wanting to update a valve position from an HMI, or a web application trying to delete a record on an ERP database on behalf of a user. It is not difficult to extend the Access Control policies to allow for Command Messages.

## Analogy with Event-Driven Design

In the context of Event-Driven Design [4], Write and Command Messages are analogue to Events and Commands:

> A command is an object that is sent to the domain for a state change which is handled by a command handler. [...] An event is a statement of fact about what change has been made to the domain state.

Read Messages allow then a client to know the full state, present and past, of a node in the UNS. If we follow the principle of Report by Exception ("producers publish data to the UNS only when they detect changes in the monitored value" [5]), then subscribing to the real-time updates of a node (`alv1/w/N` in the examples above) is not enough to know about the current state of that node. More explicitly, if a node value changes infrequently, and a client wants to know now the current value (for example to display in a web interface), then it should resort to a Read Message to get the last value.

## Conclusion

We can explicitly define a protocol on top of MQTT to handle both real-time events and a request/response style of communication. This would allow us to implement a Unified Namespace where clients can receive real-time data from a node, but also query the present and past state. 

Using different verbs in the MQTT topics, we can accommodate for different actions: `w` to send real-time changes, `r` to read past data and the whole state of a node, and `c` to send commands requesting changes to the state.

This pattern provides most of the functionality expected from a UNS, using features provided in most MQTT Servers available.

## References


[1] https://iiot-world.com/smart-manufacturing/uns-101-understanding-the-unified-namespace/

[2] https://www.hivemq.com/blog/what-is-unified-namespace-uns-iiot-industry-40/

[3] https://discord.com/channels/738470295056416930/740564843710382080/1248326011616231516

[4] https://inductiveautomation.com/ignition/

[5] https://www.hivemq.com/blog/implementing-unified-namespace-uns-mqtt-sparkplug/
