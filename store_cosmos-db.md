### Contents

- [Introduction](#introduction)
  * [Analogous Products](#analogous-products)
- [Target Use Cases](#target-use-cases)
- [Billing Method / SKUs](#billing-method---skus)
    + [Examples](#examples)
- [Important Aspects - Nuances (Performance)](#important-aspects---nuances--performance-)
  * [User Defined Functions](#user-defined-functions)
  * [Stored Procedures](#stored-procedures)
  * [Partitioning](#partitioning)
  * [Indexing](#indexing)
  * [Replication](#replication)
    + [Conflict resolution](#conflict-resolution)
    + [Failover](#failover)
  * [Consistency](#consistency)
  * [API Choices](#api-choices)
  * [Change feed](#change-feed)
- [Backups and Failures](#backups-and-failures)
- [Security](#security)
- [Monitoring and Logging](#monitoring-and-logging)
    + [Advanced Threat Protection](#advanced-threat-protection)
    + [Diagnostic Logging](#diagnostic-logging)
- [Emulator](#emulator)
    + [Limitations:](#limitations-)
- [Implementation](#implementation)
  * [Account](#account)
    + [Required](#required)
  * [Database](#database)
    + [Required](#required-1)
    + [Optional](#optional)
  * [Container](#container)
    + [Required](#required-2)
    + [Optional](#optional-1)

# Introduction

CosmosDB is a NoSQL database in Azure cloud, designed primarily to replace document store type databases. It can scale massively and globally, while also providing a wide variety of APIs so that it can be used as a "drop-in" replacement for other open source NoSQL sulutions.

The API alternatives include:
- MongoDB
- Cassandra
- Gremlin (Graph Queries)
- Azure SQL
- Table

The 3rd party APIs are emulated, so they do not support the latest features. The underlying architecture also remains the same, so it is unlikely that you get the full performance benefits (like for cassandra).

CosmosDB Account > Database > Container > Records

CosmosDB Guarantees 10ms response times and 99.999% read and write SLA (for 99% of queries and with a multi-region multi-master setup). For a single region this drops to an SLA of 99.99%.

Data in CosmosDB is partitioned horizontally to multiple nodes, and then replicated to each region. Each container is partitioned 

The database is designed to be replicated and as such does not give full consistency guarantees. Instead, the user needs to choose the appropriate consistency levels for their applications. See the "Important Aspects" section for more details.

## Analogous Products

- MongoDB
- CouchDB
- Firestore
- DynamoDB

# Target Use Cases

CosmosDB is specifically targeted for very large load an big data scenarios that also require some consistency and transactional guarantees. There are also great benefits to having a more flexible schema and being able to model relationships in different ways, such as through a graph data model.

Being able to adapt to changing data models easily is also particularly when the data is very hierarchical or if you don't have control over the source of the data.

The change feed and native JSON documents also make the database a very nice tool for microservice architectures, particularly those that use event sourcing.

Note that there are two data storage models, one is row-based and the other is column-based where the data is stored in parquet files. This can only be accessed by the Spark API, but is a great use-case for analytics workloads.

# Billing Method / SKUs

Based on the idea of RUs (Request Units) - Each operation is evaluated in terms of how many request units it takes to perform. The user pays for throughput, not operations. That means that they provision CosmosDB for the number of RUs that it should accommodate per second. If the number of requests to the database exceeds this, the requests are throttled. (When throttling occurs, CosmosDB returns HTTP 429 errors - too many requests)

The user can always scale the number of RUs/sec up to meet demand in increments of 100 (400 - 250'000). This can be achieved without downtime. These are then billed on an **hourly** basis.

RUs/sec are specced per container. That means that when you add an additional container, the total RUs for the "cluster" are increased.

Estimating how many RUs an operation will take is tricky, it depends on a wide variety of factors, but it essentially maps to compute and data transfer requirements. To get a better impression for the RUs consumed by a specific operation, you can get the amount by using the query engine in the portal or with the SDK.

The number of RUs for the exact same operation will always be the same.

### Examples

> Retrieving a 1kB document by its ID requires approximately 1 RU

> Modifications, Deletions and more complex lookups require more resources and therefore more RUs

The following table summarizes the results you've gotten in previous exercises. These results apply to read, query, and insert operations for 1 KB of data that's within a single partition.

| Operation                              | Indexing strategy             | Approximate consumption (RUs) |
|----------------------------------------|-------------------------------|:-----------------------------:|
| Read document directly                 | N/A                           | 1                             |
| Query document                         | All properties indexed        | 3                             |
| Query document by indexed property     | Partial                       | 3+                            |
| Query document by non-indexed property | Partial                       | 10+                           |
| Query document                         | No properties indexed         | 20+                           |
| Insert document                        | All properties indexed        | 13                            |
| Insert document                        | All properties indexed lazily | 7                             |
| Insert document                        | Some properties indexed       | 6                             |
| Insert document                        | No properties indexed         | 5                             |

# Important Aspects - Nuances (Performance)

## User Defined Functions
User defined functions allow you to store and perform more complex queries that may include aggregations and logic etc. These can be thought of as computed properties.

These are written in javascript and stored within CosmosDB.

## Stored Procedures
It is possible to have transactions on writes with CosmosDB, but for this the user needs to create stored procedures. These are custom functions that allow you to update multiple documents in the same transaction. The documentation calls them ACID, but it's difficult to tell how that works with the consistency accross regions. 

These are written in javascript and stored within CosmosDB.

## Partitioning
The data in a container is split into logical partitions by selecting a partition key. All data with the same parition key is colocated in the same logical parition and can be guaranteed to reside on the same physical partition too. This means that accessing related items and performing joins on this key will be fast.

The issue is that putting too much of the data in the same place can lead to partitions that become bottlenecks (all requests go to the same partition while others don't do anything). There is also a size limit of 20GB to partitions so make sure property cardinality (number of categories) is high enough.

It should also be noted that partition_keys cannot be changed after creating the container.

Choose your partitioning strategy based on whether you will be:
- **Predominantly Reading** Check common queries. Try to colocate results and use properties that are frequently used in joins.
- **Predominantly Writing** Performing transactions accross partitions is extremely expensive and should be avoided.

## Indexing

All properties are indexed automatically by CosmosDB. There are however various indexing strategies that can be set to optimise performance:

- **Consistent** This updates the indices on every write, which is fairly expensive, but also ensures that the indices are always up-to-date.
- **Lazy** This updates the indices after the writes, which means that they will be correct eventually. This strategy is a lot more resource efficient.
- **None** No idices. Most efficient writes, but queries require document scans.

## Replication
Sending a message around the world takes 100ms, so to reduce latency (but also resiliency) it makes sense to replicate data to other regions where the users are.

When adding a new region, the data will be available there within 30min for less than 100TB.

RUs are replicated, so each region adds another set of RUs.

Using a multi-master setup leads to better latency guarantees and also higher availability. It does require careful attention to the consistency setup though.

Data that is written is propagated automatically and conflicts are resolved automatically, but there are also ways to customise this. Especially by listening to the changelog.

### Conflict resolution

- **Last Writer Wins** Default
- **User Defined Functions** User defines how conflicts are handled. If these fail, the conflict is added to the conflict resolution log.
- **Custom Async** All conflicts are written to the conflict resolution log and not committed. That means that another write to the database is needed to update the document.

### Failover

 It requires multiple regions to be selected. Manual failovers are also possible and can be triggered by the user. 

If a read region fails, this is handled and brought back online automatically.

## Consistency

There are 5 levels of consistency that can be selected for a distributed CosmosDB ([for more details](https://docs.microsoft.com/en-gb/azure/cosmos-db/consistency-levels)):

1. **Strong** This guarantees linearisability, which is to say that the database acts like a single source of truth and looks the same to everyone who accesses it at any point in time. The computational and latency costs are the highest for this level.
1. **Bounded** This level guarantees strong consistency ouside of a specified time window and number of operations on the DB. It has similar resource requirements as strong consistency. With a multi-master setup this is the closest you can get to strong consistency.
1. **Session** Default - Within a client session reads and writes are guaranteed to be in order. In the same region the constant prefix consistency and outside of the region eventual consistency is guaranteed. (If you have a single-master setup, all regions have constant-prefix guarantees.)
1. **Constant Prefix** Reads will never see out of order writes. They may however see a different version of a document at a given time.
1. **Eventual** Reads may be out of order at any given time, but after a long time these are guaranteed to result in the same final state. This is a lot less resource hungry, but also difficult to work with in any kind of transactional capacity.

## API Choices

Choose the relevant Mongo, Gremlin, Cassandra or Table API when there is a preexisting codebase.

Otherwise choose either the Core or Gremlin API. The gremlin API is great for modelling relationships between data so it should be chosen in these situations => Otherwise always go for the Core SQL.

API choices can be made on the container level in the CLI

## Change feed
Uses for the change feed:
- Use it for triggering other functions / services when certain operations happen.
- Feed it into a stream processor for analytics, machine learning or whatever.
- Perform no downtime migrations
- High availability message queue. CosmosDB has a 99.999% uptime guarantee, much higher than most message queue services.

Limitations:
- The change feed does not capture deletes
- Intermediate events that happen to an item may get lost if they happen in close succession. Only the end state is guaranteed to end up in the change feed.
- The order of events are not guaranteed across partitions and may be completely messed up when eventual consistency is set.


# Backups and Failures

CosmosDB backups are performed automatically in regular intervals (every 4 hours) and stored in Geo-Redundant stores. The last 2 backups are kept. When you delete a container or database, the snapshot is backed up for 30 days.

Backups are stored in BLOB Storage, while the actual data is on CosmosDB SSD Drives for fast access. The snapshot is stored in BLOB storage in the local region and replicated to other regions.

If you have your CosmosDB set up to replicate regionally or globally any outages in one region automatically roll over requests to the next region. The order of priority of fallback data centers for each region can be set in the configuration.

Fhe concept of automatic and manual failover is applicable to single-region write (and multi-region read) account only. It is enabled automatically when adding another account.

Backups are restored by Azure Support!

The other option is to perform your own backups by using DataFactory to copy the data periodically or using the "change feed" that is like an oplog in mongo.


# Security

The data at rest and in transport is encrypted automatically as a full PaaS solution.  (AES-256). It is possible to use your own encryption keys, but Microsoft keys are default.

Access to CosmosDB can be limited by firewall rules limiting access to the DB to specific IP addresses and VPCs. 

It is also possible to allow access to the CosmosDB from Azure Services. This makes connecting easier, but note that this applies to any service that is running on Azure, even those you might not own!

It is important to configure RBAC access controls to manage user access to the data with a more fine-grained approach. There are already a range of default roles available. Authentication can take place via:
- Managed Identities and Exchanging Keys
- Certificates (Azure Active Directory)
- Azure Key Vault (Least reccommended) - This is a key server db so that you can avoid putting keys in your configuration files (especially git)


# Monitoring and Logging

Use [Azure Monitor](https://docs.microsoft.com/en-gb/azure/azure-monitor/overview) to monitor your CosmosDB instances. This allows you to monitor a wide variety of parameters such as queries access patterns, resource consumption, request statuses (http errors) and account administration.

Logs are stored in azure storage tables. There is a DSL that can then be used to query and search the logs (It's a bit like SQL but not quite).

### Advanced Threat Protection

There is a feature for CosmosDB, which is essentially anomaly detection and alerts the user to suspicious activity on the DB.

- Has to be enabled manually
- It only works for the SQL API
- Detects access from unusual locations and strange query patterns

### Diagnostic Logging

Logs all operations on the database and containers, which includes all CRUD operations. This is stored to Azure Storage containers and needs to be enabled manually. You can control what you want to enable such as requests from a specific API, RU requirements per partition_key, account administration events, partition_key statistics etc.


# Emulator

There is an emulator available for local development and it supports much of the functionality of the actual CosmosDB, but it allows you to develop locally without incurring any costs. You can create and query documents, provision and scale containers. There is also a data migration tool that allows you to upload the data from your local machine to CosmosDB.

The emulator is faithful, but the implementation is not the same. It uses the local file system for persistence and HTTPS for communication. So replication and the SLA obviously do not apply.

It runs in a windows docker container - it does not work on linux containers.

The emulator does not use encryption. You are responsible for the data that is on your development machine.

### Limitations:
- The DataExplorer window only works with the SQL API.
- It looks like there is only an image for Docker for windows.
- It only supports a fixed account with a single master key - no key swapping.
- It is not scalable and does not support a large number of containers (5 unlimited size or 25 fixed size containers), replication, or consistency levels.
- The RUs may not always be in line with what you see online. Check in the actual service for accurate numbers.


# Implementation

create account > create database > create container

## Account

### Required
--name
--resource-group

## Database

### Required
--name
--account-name
--resource-group

### Optional
--capabilities
--default-consistency-level
--disable-key-based-metadata-write-access
--enable-automatic-failover
--enable-multiple-write-locations
--kind (This is for the API type)
--locations
        Usage:          --locations KEY=VALUE [KEY=VALUE ...]
        Required Keys:  regionName, failoverPriority
        Optional Key:   isZoneRedundant
        Default:        single region account in the location of the specified resource group.
        Failover priority values are 0 for write regions and greater than 0 for read regions. A
        failover priority value must be unique and less than the total number of regions.
        Multiple locations can be specified by using more than one `--locations` argument.

## Container

### Required
--name
--account-name
--database-name
--resource-group
--partition-key

### Optional
--throughput
--idx
--conflict-resolution-policy
--ttl
