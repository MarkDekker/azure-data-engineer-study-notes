### Contents

- [Introduction](#introduction)
- [Target Use Cases](#target-use-cases)
- [Billing Method / SKUs](#billing-method---skus)
- [Important Aspects - Nuances (Performance)](#important-aspects---nuances--performance-)
  * [Architecture](#architecture)
  * [Data Storage and Distribution](#data-storage-and-distribution)
    + [Hints](#hints)
  * [Indexing](#indexing)
    + [Hints](#hints-1)
  * [Workload Management](#workload-management)
  * [Other Nuances](#other-nuances)
- [Backups and Failures](#backups-and-failures)
- [Security](#security)
  * [Transparent Data Encryption](#transparent-data-encryption)
  * [Connection](#connection)
  * [Authentication](#authentication)
  * [Authorisation](#authorisation)
  * [Advanced Data Security](#advanced-data-security)
    + [Discovery and Classification](#discovery-and-classification)
    + [Vulnerability Assessment](#vulnerability-assessment)
    + [Advanced Threat Protection](#advanced-threat-protection)
- [Auditing](#auditing)
    + [Limitations](#limitations)
  * [Dynamic Data Masking](#dynamic-data-masking)
- [Monitoring and Logging](#monitoring-and-logging)
- [Workflow](#workflow)
- [More Details on Polybase](#more-details-on-polybase)
  * [Limitations](#limitations-1)
  * [Steps](#steps)
    + [Commands for linking Polybase](#commands-for-linking-polybase)

# Introduction

Synapse analytics bundles data warehousing and analytics solutions for "limitless analytics". 

The solution separates compute from storage, which means that you can also scale them independently from one another and pay for what you use. For compute you can either opt for serverless (preview) or provisioned resources.

Synapse Analytics comprises:
- Synapse SQL - Analytics using T-SQL (compute)
- Spark - Deeply integrated solution (preview)
- Synapse Pipelines (preview)
- Studio - User Interface (preview)

For the exam we only need to really understand the Provisioned Synapse SQL Pools (and a little of the Spark integration):

The compute engine for SQL Pools is the "Massively Parallel Processing" (MPP) architecture. This essentially divides the data to be processed amoung multiple (max 60) compute instances. These each host a local Relational Columnar Database and a data movement service (DMS). One master control node manages and orchestrates these compute nodes. The data is loaded directly from Azure Storage.

To get the data loaded from Azure Storage into the SQL Pool quickly and efficiently, the best-practice is to use [PolyBase](./service_polybase.md).

# Target Use Cases

It is a useful way to replace / migrate to a cloud-based data warehouse solution that can be used by traditional business data analysts. It enables bigger scale than traditional on-premise systems and more responsiveness than other big data analytics processes based on Hadoop.

So use it especially for more interactive queries, but note that it is on the more expensive side...

Synapse really starts shining for datasets > 1 TB, for anything under 250GB it is better to just use Azure SQL databases.

# Billing Method / SKUs

Azure Synapse is billed based on data warehouse units (DWUs) for compute and basic Azure Storage for persistence.

Data warehouse units are similar to Resource Units in CosmosDB. The administrator provisions the compute power for the data warehouse and any excess load is placed in a queue based on its priority. See workload management for more details.

There are two generations of compute nodes that can be used with Synapse. The newer generation has up to 5x more compute power and 4x more capacity for concurrent requests.

DWU (cDWU in Gen2) incorporate memory, CPU and IO in bundles. Increasing DWU linearly scales the capacity of the system. It also increases the number of compute nodes that are used in the cluster. Gen2 uses local disk-based cache to improve performance.

# Important Aspects - Nuances (Performance)

One of the core aspects of synapse analytics, is that it separates data storage from data processing. This also means that the storage and compute can be scaled independently from one another and one day it will even be possible to only pay for the compute you use (serverless - in preview).

## Architecture

The SQL Pool is accessed and managed through the control node. This is effectively the master node that controls the compute nodes based on the instructions it receives from a client or user.

Each of the control and compute nodes has a data movement service DMS module that takes care of internode communication and transfer of data.

Compute nodes each get assigned a shard of the data in storage and process these based on the instructions from the control node. There can be 1-60 compute nodes in a cluster. 

## Data Storage and Distribution

The data is stored in a columnar format in Azure Storage, which means that this also drives the pricing here. The data is sharded into distributions to make parallel processing more efficient.

There are always 60 data distributions regardless of the number of nodes. When you change the number of nodes, the data distributions are redistributed so that each distribution is assigned to a compute node (there can be many to one).

The sharding strategy can be chosen amoung the following three options:
- **Round Robin** default - Data is assigned to shards arbitrarily, that makes it a good option for staging tables where there are no clear query patterns and you want to write data as quickly as possible.
- **Hash** For big data, this sharding pattern should be the goal. Here you have to choose the column that will be used to distribute the data iin a given row (Similar to how CosmosDB does it). Excellent coice for fact and large dimension tables.
- **Replicated** This is by far the fastest strategy, however it is very resource intensive. That means that it is only feasible for __tables__ smaller than 2GB (5x Compression). This also means that there is a larger overhead when writing data. This also works well for small dimension tables inside of a star-schema.

### Hints

- Don't distribute on varchar!
- Make sure that common hash keys have the same data type ... Hashing a different data type gives a different outcome.
- Dimension tables with a common hash key in a fact table can also be hash distributed if there are common join operations.
- Check the partition statistics to check for skewness
- Check data movements, time broadcast and shuffle with queries to optimise distribution.

## Indexing

Indexing Strategies:
- **Heap** This is best for staging tables, not great for query performance. Also works well on small tables with small lookups - Replicated tables.
- **Clustered Index** This works well for tables < 100 Million rows or more than that if only 1-2 columns are used frequently for lookups.
- **Clustered Columnstore Index** Default - Good for large tables. Is not great when you make large updates on a table. Should have > 60 Million rows to really see a benefit. Do not create these under "memory pressure" => use account wnd compute setup with bigger nodes when creating the table.

### Hints

- If you are filtering a column a lot add a non-clustered index in addition to the clustered index.
- Carefully manage memory for CCI (the default). Avoid small row groups.
- Gen2 CCI tables are cached on compute node for better performance.
- Target 1 million rows per row-group. Less than 100'000 will result in poor performance.
- Rebuild indices automatically to keep performance good.

## Workload Management

Make full use of resource prioritisation and resource classes for queries and workloads to assign jobs the appropriate amount of compute power, memory and priority in the workload queue. It is also possible to reserve capacity for certain types of workflows so that these can start immediately when necessary.

Resource clases - the old way of doing things. Problematic to manage.

New approach:
- Workload Importance - Influences queue priority.
- Workload Isolation - Reserves (or limits) resources for workload groups. These can then be further split into a predefined amount per request in that group.
- Workload Classification - Assign requests to workload groups and importance.

This gives control over how workloads are run and prioritised.

## Other Nuances

Consider paritioning the fact table once you go > 1 Billion rows. Also, don't overpartition (esp. with CCI indices)

Maintain statistics to improve query performance and update these when significant changes take place. There is also an option to autocreate and maintain statistics.

Use materialised views to improve performance.

# Backups and Failures

The data in the SQL Pool can be backed up as restore points manually or use the automatically created snapshots that are enabled by default. (CLI, portal, powershell)

Note that the storage is based on Azure storage so much of the redundancy, snapshots and backup functionality can be used here too.

Active geo-replication leverages the Always On technology of SQL Server to asynchronously replicate committed transactions on the primary database to a secondary database using snapshot isolation.

Auto-failover groups is a SQL Database feature that allows you to manage replication and failover of a group of databases on a SQL Database server or all databases in a managed instance to another region. It is a declarative abstraction on top of the existing active geo-replication feature, designed to simplify deployment and management of geo-replicated databases at scale.


# Security

## Transparent Data Encryption

There is the option of using transparent data encryption (TDE) to protect data at rest. This data is then decrypted when it is loaded into memory. This is also available in Azure SQL Databases. [More Details](./store_sql.md) The data is encrypted (AES-256) with a symmetric key. Each Azure Database Server has a unique built-in certificate which is used to protect the encryption key. The certificates are automatically rotated every 90 days by Microsoft.

Enabling TDE also encrypts logs and backups.

Once a database is encrypted with TDE using a key from Key Vault, any newly generated backups are also encrypted with the same TDE protector. When the TDE protector is changed, old backups of the database are not updated to use the latest TDE protector.

If the server loses access to the customer-managed TDE protector in AKV, in up to 10 minutes a database will start denying all connections with the corresponding error message and change its state to Inaccessible.

If key access is restored within 8 hours, the database will auto-heal within next hour.

If key access is restored after more than 8 hours, auto-heal is not possible and bringing the database back requires additional steps on the portal and can take a significant amount of time depending on the size of the database. Once the database is back online, previously configured server-level settings such as failover group configuration, point-in-time-restore history, and tags will be lost.

it is highly recommended to configure the server to use two different key vaults in two different regions with the same key material

## Connection

Set the firewall rules to restrict access to the server to specific IP (ranges) or virtual subnets. Synapse does not allow IP firewall restrictions on the database level as is possible for Azure SQL.

For local access to the SQL Pool the local machine also needs outgoing access on port 1433 (TCP connections).

Connections to the SQL Pool are encrypted by default and cannot be disabled.

## Authentication

You can authenticate with Azure AD or with SQL Server Authentication (username and password). 

On creating the database server you have to specify a user and password for the database owner. This can be used for full access to the databases on that server. Do not let your consumers of these databases use these credentials, create additional users for each of them. This also greatly improves monitoring, resource management and troubleshooting.

## Authorisation

Same best practices as everywhere else: least is best. Assign users roles to grant access. There are also granular permissions that can limit access from the database down to the column (column-based), schemas and procedures.

## Advanced Data Security

A unified package that can be used for advanced SQL security. There is functionality to discover and classifying sensitive data, finding security vulnerabilities and detecting anomalous activity.

This service is not enabled by default and it has additional Azure Security Center costs associated with it. Each database server is counted as one node.

### Discovery and Classification
Discover what data you have, label it according to its sensitivity and who should have access and record the type of information. There are presets available already for labelling and information types, but you can also create your own naming and patterns. 

This then also allows administrators to monitor who accesses this sensitive data and to set permissions on an account level.  
### Vulnerability Assessment
This service can be configured to automatically identify, track and fix security vulnerabilities. It checks that data compliance requirements and privacy standards are met. It finds lax permissions, unprotected sensitive data and configuration mistakes.

### Advanced Threat Protection
This is the same as for all the other services. It does anomaly detection on access patterns and maintains a list of malicious IP addresses.


# Auditing

To meet regulatory compliance standards you may need database auditing enabled. It can be defined on a database or server level. Audit logs are written to Storage, EventHubs or Log Analytics. With this enabled you can maintain a record of selected events, create activity reports and analyse the audit log when required.

### Limitations
- Premium storage is not supported
- Hierarchical namespaces are not supported
- This only works on unpaused Synapse instances

## Dynamic Data Masking

To limit sensitive data access for less priviledged users, dynamic data masking can be used to hide part or all of the data returned from a query. This works both in Synapse and SQL Server. The data on in the database remains unchanged but the user sees only part of the data in a masked field.

This is very useful for payment details or personal data. The dynamic data masking engine automatically detects and reccommends fields that would be wise to mask.

Note that masking is not as safe as encrypting the data completely and there are ways for malicious actors to get around masking.

It can only be activated in the CLI or Powershell for Synapse. (Not the portal!)

Masking Funtions:
- Default - Mask according to the data type of the masked field.
    - XXXX or less for text data types
    - 0 for numeric data types
    - 01-01-1900 for dates and time
    - special data types are left blank
    - `<masked/>` for XML
- Credit Card - reveals last four digits
- Email - aXX@XXX.com
- Random number - in a preset range
- Custom Text - reveal x number of prefix characters and y number of suffix characters

# Monitoring and Logging

Logs can be emitted to Azure Storage, Stream Analytics and Log Analytics. Log Analytics is reccommended to take advantage of the Azure log querying, monitoring and alerting tools.

Use DMVs (Dynamic Management Views) to monitor workloads and query execution. This also allows you to monitor access to the data warehouse. It logs the last 10000 queries executed.

For each query you can see the query plan steps and their respective data movement.

It is also possible to monitor currently waiting queries.

There is a temp database that contains the intermediate results for any queries. This can be accessed and monitored.

# Workflow

To get data into synapse from your local SQL Data Warehouse:
1. Export data from your SQL tables with the "bluk copy program" BCP. This converts relational tables into flat files.
1. Load the flat files into BLOB storage with AzCopy - This goes over the public internet. If you need a faster connection there is the option of using ExpressRoute to create a dedicated private connection. You can also ship it physically.
1. Load the data into Synapse with PolyBase - Into a staging table
1. Transform the data into a useful star- or snowflake-schema
1. Load the semantic model in Azure Analysis services (basically define primary and foreign keys that allow you to join the data easily)
1. Load the data into a PowerBI model - do not perform queries directly (can get very expensive)

# [Read More](https://docs.microsoft.com/en-gb/azure/synapse-analytics/sql-data-warehouse/)


# More Details on Polybase

PolyBase enables your SQL Server instance to process Transact-SQL queries that read data from external data sources.

The same queries that access external data can also target relational tables in your SQL Server instance.

PolyBase pushes some computations to the Hadoop node to optimize the overall query. However, PolyBase external access is not limited to Hadoop. Other unstructured non-relational tables are also supported, such as delimited text files.

With the underlying help of PolyBase, T-SQL queries can also import and export data from Azure Blob Storage. Further, PolyBase enables Azure SQL Data Warehouse to import and export data from Azure Data Lake Store, and from Azure Blob Storage.

In the past it was more difficult to join your SQL Server data with external data. You had the two following unpleasant options:

- Transfer half your data so that all your data was in one format or the other.
- Query both sources of data, then write custom query logic to join and integrate the data at the client level.

Push computation to Hadoop. Pushing computation creates MapReduce jobs and leverages Hadoop's distributed computational resources.

Scale compute resources. To improve query performance, you can use SQL Server PolyBase scale-out groups.



## Limitations

Be aware of the following limitations:

PolyBase supports a maximum column size of varchar(8000), nvarchar(4000), or varbinary(8000). If you have data that exceeds these limits, one option is to break the data up into chunks when you export it, and then reassemble the chunks after import.

PolyBase uses a fixed row terminator of \n or newline. This can cause problems if newline characters appear in the source data.


# [Read More](https://docs.microsoft.com/en-us/sql/relational-databases/polybase/polybase-guide?toc=/azure/synapse-analytics/sql-data-warehouse/toc.json&bc=/azure/synapse-analytics/sql-data-warehouse/breadcrumb/toc.json&view=azure-sqldw-latest)



## Steps

1. Create a Scoped Credential
1. Create an External BLOB
1. Create an External Data Source
1. Create a Temp Table to Load the Data
1. Load the data into SQL Datawarehouse


### Commands for linking Polybase
1. Create Master Key and Scoped Credential

CREATE MASTER KEY; 
CREATE DATABASE SCOPED CREDENTIAL AzureStorageCredential
WITH
    IDENTITY = `value`,
    SECRET = `storage-access-key-value`
;

2. Create External Data Source (define it)
CREATE EXTERNAL DATA SOURCE AzureStorage
WITH (
    TYPE = HADOOP
    LOCATION = `resource-uri-to-blob`,
    CREDENTIAL = AzureStorageCredential     // The credential from before
);

3. Define import file format
CREATE EXTERNAL FILE FORMAT TextFile
WITH (
    FORMAT_TYPE = DelimitedText,
    FROMAT_OPTIONS = (FIELD_TERMINATOR = ',')
)

4. Create Temporary Table
CREATE EXTERNAL TABLE dbo.Temp (
    `add all of you column names, data types and default values`
)
WITH (
    LOCATION = '/',
    DATA_SOURCE = AzureStorage              // The data source from step 2
    FILE_FORMAT = TextFile                  // The file format from step 3
);

5. Create a Destination Table (Staging Table)
CREATE TABLE dbo.StageDate
WITH (
    CLUSTERED COLUMNSTORE INDEX,            // Index choice should probably be heap instead
    DISTRIBUTION = ROUND ROBIN
)
AS
SELECT * FROM dbo.Temp

6. Optional - Create statistics of important columns to improve query performance.