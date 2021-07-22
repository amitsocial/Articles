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