### Contents

- [Introduction](#introduction)
- [Billing Method / SKUs](#billing-method---skus)
  * [vCores](#vcores)
  * [DTU](#dtu)
  * [Elastic Pools](#elastic-pools)
    + [DTU-based purchasing model example](#dtu-based-purchasing-model-example)
- [Important Aspects - Nuances (Performance)](#important-aspects---nuances--performance-)
  * [Service Tiers -  DTU](#service-tiers----dtu)
  * [Service Tiers -  eDTU](#service-tiers----edtu)
- [Backups and Failures](#backups-and-failures)
- [Security](#security)
- [Monitoring and Logging](#monitoring-and-logging)
- [Implementation](#implementation)
  * [Database Server](#database-server)
  * [Required](#required)
  * [Database](#database)
    + [Required](#required-1)
    + [Optional](#optional)

# Introduction

Fully managed PaaS SQL database based on Microsoft SQL Server. Upgrades, patches, backups and monitoring are handled by Azure. Can also handle JSON and XML ("NonRelational" structures).

There are two purchasing models: DTU and vCores. DTU are simple bundled performance options and vCores allow you to scale performance more granularly in the different dimensions (CPU, IO and Memory)

There are 3 operating models:
- Single Database - Single, fully managed DB (good choice for cloud first approach)
- Managed Instance - A full SQL server instance managed by Azure (good choice for migrations)
- Elastic Pool - Collection of single databases that can share resources (single instances can be moved in and out of these flexibly)

Scale the DB resources without downtime, automatic optimisation, global scalability and availability and advanced security options.

Single databases can be specced with 1-80 vCores (dedicated to the DB) and 32GB to 4TB (or even 100TB in Hyperscale).

The databases can be pooled into elastic pools to share the resources and save money.


# Billing Method / SKUs

The DTU pricing model is simpler and cheaper for lighter workloads, or those that more closely mimic the distribution of resources in the DTU scaling. Billed per hour.

vCores become cheaper when you have a larger resource consumption that is especially skewed on one or two dimensions. Billed per hour.

There is also a serverless model although this was not mentioned in the learning path... It allows you to set the scaling limits and how long to wait before pausing the DB completely. Then you only pay for storage. This does mean that there is some warmup delay after coming out of the paused state though. Billed per second.

## vCores

The option to choose between serverless and provisioned. The vCore unit price per unit of time is lower in the provisioned compute tier than it is in the serverless compute tier. Choose CPU and Storage independently vs. the preset options, linear scaling for DTU.

The managed instance deployment option in Azure SQL Database offers only the vCore-based purchasing model.

The Hyperscale service tier is available for single databases that are using the vCore-based purchasing model.

In the business critical tier 3 replicas are provisioned so costs are 2.7x higher compared to general purpose. Likewise, the higher storage price per GB in the Business Critical service tier reflects the higher IO limits and lower latency of the SSD storage.

You pay for the maximum storage you allocate for. (backups are dynamic)

Choose: n cores + memory, gen hardware, service tier (general and business critical), type and amount of data and log storage.

For single databases, you can also choose the Hyperscale service tier.

If you DB or Pool consumes more than 300 DTUs, converting to the vCore-based purchasing model might reduce your costs.

## DTU
A database transaction unit (DTU) represents a blended measure of CPU, memory, reads, and writes.

In the DTU-based purchasing model, you can choose between the basic, standard, and premium service tiers.

The DTU-based purchasing model isn't available for managed instances.

The ratio among these resources is designed to be typical of real-world OLTP workloads

To gain deeper insight into the resource (DTU) consumption of your workload, use query-performance insights to identify the top queries by CPU/duration/execution count that can potentially be tuned for improved performance.

> **Note** SQL Server database engine typically uses all available memory for its data cache to improve performance so don't use this as a scaling criteria.

In the DTU-based purchasing model, customers cannot choose the hardware generation used for their databases.


## Elastic Pools

When you have multiple databases with variable load, you can make use of elastic pools which allow you to share vCore or DTU resources between databases. This may be interesting to SaaS service providers that would like to provision one database per customer. There is no per-database charge! You only pay for the eDTU or vCores you provision. (max 100-500 DBs per pool depending on vCore/DTU model)

The user sets minimum and maximum resource limits for each database and the overall resources for the pool.

If the aggregate amount of resources for single databases is more than 1.5x the resources needed for the pool, then an elastic pool is more cost effective.

In general, not more than 2/3 (or 67%) of the databases in the pool should simultaneously peak to their resources limit.

Assess maximum storage size and resource requirements for all databases in the pool to decide on resources for the entire pool.

Elastic pools also give access to elastic jobs to make managing a large number of similar databases painless.

### DTU-based purchasing model example
At least two S3 databases or at least 15 S0 databases are needed for a 100 eDTU pool to be more cost-effective than using compute sizes for single databases.


# Important Aspects - Nuances (Performance)

There are three architectural models that are used in Azure SQL Database:

- General Purpose/Standard
- Hyperscale - Very large scale data 100TB, near instant backups and restores, higher log throughput, scale out with additional read nodes
- Business Critical/Premium


## Service Tiers -  DTU

- Basic - Max backup 7 days, 1-5 IOPS/DTU, CPU Low
- Standard - Backup 35 days, CPU Low Med High, Columnstore Indexing (S3)
- Premium - Backup 35 days, CPU Med High, Columnstore Indexing, 25 IOPS/DTU

Service 

## Service Tiers -  eDTU

- Basic - Max size 2GB DB, 156GB Pool
- Standard - Max size 1TB DB, 4TB Pool
- Premium - Max size 1TB DB, 4TB Pool

# Backups and Failures

Geo-Replication - Default (full every week, differential every 12hours, transaction logs every 5-10min)
Auto-Backup
Backup Retention 7(default)-35 Days
Long-Term Backup Retention up to 10 Years

A full restore takes less that 12 hours in most cases

# Security
Shares much in common with Synapse. [See more there](./store_synapse-analytics.md) for details on Masking, Transparent Data Encryption, Authentication, Advanced Data Security and Auditing.

TLS is enforced for all connections.

Always encrypted allows you to encrypt data in specific columns even when it is in use. So database administrators etc can never see it. (Passwords and credit cards) The encryption key is never exposed to SQL Server and stored in a different Key Vault or Certificate Store.



# Monitoring and Logging

You need auditing if you want to use threat detection. Not on by default.

As a best practice, avoid enabling both server blob auditing and database blob auditing together, unless:

You want to use a different storage account or retention period for a specific database.
You want to audit event types or categories for a specific database that differs from the rest of the databases on the server. For example, you might have table inserts that need to be audited but only for a specific database.

Otherwise, it's recommended you enable only server-level blob auditing and leave the database-level auditing disabled for all databases.



# Implementation
You need a database server to create an elastic pool or a single database.
## Database Server
## Required
--name
--resource-group
--location
--admin-user
--admin-password


## Database
For elastic pools it is possible to add exisiting databases after the fact.

### Required
--resource-group
--server
--name

### Optional
--sku-name
