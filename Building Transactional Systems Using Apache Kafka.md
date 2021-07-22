# Building Transactional Systems Using Apache Kafka
## _The Last Markdown Editor, Ever_

[![N|Solid](https://cldup.com/dTxpPi9lDf.thumb.png)](https://nodesource.com/products/nsolid)

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

Traditional relational database systems are ubiquitous in software systems. They are surrounded by a strong ecosystem of tools, such as object-relational mappers and schema migration helpers. Relational databases also provide strong guarantees in the form of ACID transactions, which are loved by developers for their all-or-nothing semantics.

Today’s businesses, however, want to process ever-increasing amounts of data. Write-heavy loads in particular may run into scalability issues in traditional relational databases and therefore need alternative architectures that scale to their needs. This article presents an event-based architecture that retains most transactional properties as provided by an RDBMS, while leveraging Apache Kafka® as a scalable and highly available single source of truth.

## ACID? Say again?
ACID refers to Atomicity, Consistency, Isolation, and Durability. What do these mean, exactly?

- **A**tomicity in relational databases ensures that a transaction either succeeds or fails as a whole. This is especially relevant if the transaction consists of multiple SQL statements.
- **C**onsistency expresses the idea that the database is in a valid state. Martin Kleppmann argues in his book Designing Data-Intensive Applications that consistency is an application-specific notion. The database can only provide support by enforcing constraints, such as referential integrity. Defining the correct constraints and transactions is still up to the application.
- **I**solation refers to transactions that can be processed without interfering with other transactions, for example, during concurrent processing. The database system guarantees that multiple concurrent transactions will appear to the user to be executed one after the other. If two conflicting transactions modify the same entity, one of them will be successful and the other will fail.
- **D**urability guarantees that the results are persisted after a successful commit. All of these are enforced by relational databases. Upholding each property in a system based on Kafka is tricky but not impossible, as you are about to find out.
## A simple multi-tenant system
Let’s assume we want to build a multi-tenant system. Each tenant has a unique fixed identifier as well as other miscellaneous data, such as contact details or billing address. In order to add, modify, or delete a tenant, an administrator can interact with the system via an HTTP API. The event streaming model lends itself to an event-based architecture, so Kafka serves as a central event hub. All API calls are transformed into events and written to a Kafka topic using the tenant identifier as key and the remaining data as value.

If an event consumer encounters a new tenant identifier, it will create a new tenant. Subsequent events with the same key are considered modifications to the tenant, and tombstones represent tenant deletions. Since an event contains all tenant data, the newest event on the stream always represents the current state of a tenant.

![Multi-Tenant_System-e1566232715526](https://user-images.githubusercontent.com/87693294/126628216-3844c968-a495-4a31-a415-71f717562b00.png)

A naive implementation of this system will have several issues: Kafka consumers will automatically commit event offsets every five seconds by default. If the event processor goes down while creating a new tenant, the event still might have been committed although no tenant was created. In effect, the event is considered to be successfully processed when in reality it failed. Much in the same way, the producer can fail to submit an event to Kafka, although the user has already been notified that the operation was successful. So what are the nitty-gritty details required to make our example system transactional?

## Infusing transactional properties
In our system, a transaction is represented by a single event only. This means that an event is the most fine-grained unit we read from or write to Kafka. At the same time, events contain all the information necessary for manipulating a tenant. Reading and writing events is therefore inherently atomic, just like transactions in a relational database. That is straightforward, but what about consistency?

In an RDBMS, the database engine is responsible for enforcing constraints and referential integrity. However, Kafka does not know the exact data model. It concerns itself only with binary key-value pairs and cannot enforce any constraints on your data. In order to achieve consistency, the event producer must take care of everything before submitting an event to a Kafka topic. Verifying incoming data is something that needs to be performed anyway, as is the case with our event producer that receives input values via the HTTP API.

When events are sent from one or more producers to Kafka, Kafka will preserve the order of events. This is not a global ordering across partitions but rather a total ordering within each partition. Luckily, Kafka ensures that all of a partition’s events will be read by the same consumer so no event will be processed by two conflicting consumers. Each event is processed in isolation from other events, regardless of the number of partitions and consumers, as long as all processors of a specific event type are in the same consumer group.

Lastly, in order to achieve durability in our system, the event producer and processor must pay special attention to acknowledgements and commits. If a user requests the creation of a new tenant via the API, the producer needs to delay the response to the user until the tenant event is acknowledged by the Kafka cluster. Kafka’s durability guarantees now prevent the event from being lost if the API service goes down. Note that you can tune the level of durability based on the requested type of acknowledgement (ack): producers may request no ack at all, an ack by the leader, or an ack from the cluster as soon as the minimum number of replicas are in sync.

Once the event is persisted, it needs to be consumed by an appropriate event processor, though the processor may go down before it has finished. If auto-commit is enabled for the Kafka consumer, the event processor might already have commited the event offset and fail before finishing the event. The consumer must commit the event manually after all relevant sub-processes have completed. For example, if a new tenant was created and that tenant had to be registered with a third-party payment provider, the consumer delays committing the event until the tenant is successfully registered with the payment service. Only the synergy between the event producer, processor, and Kafka allows for proper durability.

## Prerequisites and limitations of the architecture
At this point, our system has most of the benefits that come with ACID transactions in relational databases: a user’s actions are atomic, consistency is handled by event producers, events are isolated by design, and durability is guaranteed by smart acks and commits. But to continue on this path, there are more considerations.

## Modeling events and transactions
The system was initially presented as “event based.” Yet, there are different ways to design events, and this term does not enforce any particular style. In our context, events should be self-contained. A tenant event, for example, should encapsulate all information about that tenant so that no dependencies on other events exist. This is referred to as event-carried state transfer. A transaction may in turn consist of one or more events. The following image depicts a producer that sends two events representing a single transaction to a Kafka topic and two event processors in the same consumer group that read events from the topic.
![Kafka_Processor_A-e1566232390379](https://user-images.githubusercontent.com/87693294/126628384-0d412020-bff6-4c79-a607-b0dda5369477.png)

There may be cases where the first part of a transaction is processed by the consumer and a rebalance kicks in. The partition to which both events were sent are now processed by a different consumer that might lack the context of the previous events.

![Kafka_Processor_B-e1566232420283](https://user-images.githubusercontent.com/87693294/126628420-483cd8ec-841c-475c-90d7-8271ab288300.png)

