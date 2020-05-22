### Contents

- [Introduction](#introduction)
  * [Analogous Products](#analogous-products)
- [Target Use Cases](#target-use-cases)
- [Billing Method / SKUs](#billing-method---skus)
  * [Storage](#storage)
  * [Transactions](#transactions)
    + [Reads](#reads)
    + [Writes](#writes)
  * [Optional](#optional)
- [Important Aspects - Nuances (Performance)](#important-aspects---nuances--performance-)
  * [BLOB types](#blob-types)
  * [Access Tiers](#access-tiers)
  * [Performance Tiers](#performance-tiers)
- [Backups and Failures](#backups-and-failures)
  * [Data Redundancy](#data-redundancy)
  * [Preview features - Don't learn](#preview-features---don-t-learn)
- [Security](#security)
  * [Network Access](#network-access)
  * [Identity Access Control](#identity-access-control)
    + [Shared Access Signatures](#shared-access-signatures)
- [Implementation](#implementation)
  * [Required Fields](#required-fields)
  * [Optional](#optional-1)

# Introduction
Azure Storage is the cheapest and simplest way of storing data in the cloud. It is also a very versatile solution catering to many needs. 

The service was recently revamped, which means that there is a Gen1 and a Gen2 version of the service. In practice, unless maintaining a legacy service, Gen2 should be chosen for the richer feature set and better performance.

Accessing the data stored in Azure Storage can be done via a REST API (HTTP/HTTPS), the CLI or via client libraries in a variety of languages:

- Go
- PHP
- Python
- Node.js
- .NET
- Ruby
- Java

To start using storage, you need a storage account. Generally you create a new storage account for each new project and use-case, since the storage account provides an umbrella to set access policies and performance tiers for all the data stored under a given account.

There are different storage account types that give the user access to different kinds of features and performance SLAs:
- **General Purpose** v1 & v2
- **BlobBlock Storage** - Premium performance where you have small objects but you need good latency and read performance.
- **File Storage** - High performance for files only.
- **Blob Storage** - Legacy, try to use the general accounts instead.

Data stored in storage can be classified as one of the following object types:
- Table
- Queue
- BLOB (Append, Block, Page)
- Files
- Disks
- Data Lake

In addition to this, there are performance tiers (standard and premium) and access tiers (hot, cool, archive) to satisfy your access requirements.

Data is replicated with multiple copies within the same primary region automatically in case of hardware failure, but there are also a variety of options to replicate the data to other locations and to provide read access here too. For more information see the **Backups and Failures** section.

DataLake Gen2 is built on top of Storage and shares many commonalities with the service. [For more information see this document.](./store_data-lake.md) service is built on top of the

## Analogous Products

- S3
- Cloud Storage

# Target Use Cases

- Big Data (tables, data lake)
- Storing binary data that is referenced by entries in a database
- Hosting static web assets such as images, videos and static websites
- Storing logs
- As a cheap and simple event log (queue)
- File system backups for cloud-based VMs
- Backups of databases etc.
- File shares

# Billing Method / SKUs

You pay for storage, transactions and metadata related updates, which are cherged per 1000s of operations.

## Storage

HOT = 2x COOL 
COOL = 10x ARCHIVE

## Transactions
Writes cost 10x what reads cost, except for archive, where a read costs 100x more than for HOT storage...

### Reads

COOL = 3x HOT
ARCHIVE = 100x HOT

### Writes

COOL = 2x HOT
ARCHIVE = COOL

## Optional

Reserving capacity and support

# Important Aspects - Nuances (Performance)

The data hierarchy in Azure Storage consists of:
> Storage Account > Containers > Blobs

Note that containers cannot be nested inside of one another. Instead, it is recommended to create a "folder hierarchy" by naming convention. Please note the character and length restrictions for this. YOu can use forward slashes in names though.

## BLOB types

-**Block** Blobs are composed of blocks and are optimal for text or binary files.
-**Append** Also made of blocks but optimised for append operations.
-**Page** Made of 512-byte pages up to a total size of 8GB. Designed for random read-write access. They are the foundation for VM disks and file systems. This also makes them interesting as a base for databases.

## Access Tiers
The hot and cool access tiers can be set at the BLOB level.

- **HOT**
- **COLD** Store data for at least 30 days. Lower availabilty and higher access costs (lower storage costs). Latency and throughput are similar to hot.
- **ARCHIVE** Store for at least 180 days, variable latency (can be hours). Cannot be set at the account level. Highest access costs and lowest storage costs. This data needs to be "rehydrated" before it can be used again.

## Performance Tiers
- **Standard** High capacity, high throughput
- **Premium** Consistent single-digit latency, high throughput and high transaction rates

The premium tier is for when you are closer to database style loads such as for analytics and processing, interactive access, data transformation, AI. The standard tier is more appropriate for serving media, hosting backups and for bulk data processing.

For premium performance the storage type needs to be set to BlockBlobStorage.

# Backups and Failures

There is the option of enabling soft deletes for BLOBs, which means that there is a possibility to recover deleted data. The user specifies a retention period for this deleted data. This works for BLOB data only (page, append or block).

A further option is to create snapshots of your container. This happens at a preset interval and creates read-only copies of your data. A snapshot cannot outlive its base-blob (it is not a backup!). Repeated snapshots of unchanged data do not incur any charges (probably hash based), the storage costs increase only when there is new data.

## Data Redundancy

There are a variety of redundancy and distribution modes that can be chosen for azure storage:
- **LRS Locally Redundant Storage** Default - Storage is backed up in a singe region. (all in a datacenter)
- **ZRS Zone Redundant Storage** Replicate data to 3 other zones in the primary region. Higher availability. Writes are synchronous, so it is only complete once all zones have been written to... The availability is limited to some regions.
- **GRS Geo-Redundant Storage** Uses LRS in the primary region and then replicates the data to other regions.
- **GRZS Geo-Zone-Redundant  Storage** Uses ZRS in the primary region and then replicates the data to other regions.

If the data is not stored in LRS, it will always provide read and write access even if there is a failure in one of the data centers. => Automatic Failover

Note that data in secondary regions for geo-redundant storge is only available after a failure. If you want to read from a secondary region during normal operation you have to enable this manually.

When this happens there may be a small amount of downtime / inconsistency between the data. You as the application developer are responsible for accommodating this situation.

Azure automatically checks and repairs for any data corruption.

## Preview features - Don't learn

**Preview** Another possibility is versioning BLOBs, which allows you to track changes on the data over time. This also works in combination with the soft deleting option. BLOB versioning creates a new instance of the object for each version, it is not diff-based like Git...

**Preview** There is an immutable, guaranteed, durable, read-only change feed associated with a storage container, which allows you to track chages to objects over time. This can be processed in a stream or in batch mode for any number of applications. (Does not work for data lake) You pay for the storage of this feed and can customise the retention period.

# Security

All data stored is encrypted at rest (with 256-bit AES encryption). There is also the possibility to enforce encrypted data transfer, but this is disabled by default. You can use the default encryption keys from Azure or provide your own.

CORS is supported.


## Network Access

Secure network access to the storage account with the usual suspects in the firewall rules:
 - Limit IP Ranges
 - Confine it to a VPC
 - Limit to specific Azure Resources

There is also the possibility to provide private endpoints within a private network. With this you can rest assured that all traffic between your service and the storage account does not leave the Microsoft network infrastructure - it does not pass through the public internet.

Enable logging of network events to monitor access patterns.

## Identity Access Control

Even when you allow network access from a limited set of IP ranges, access to the storage resources (containers and objects) can/should be restricted to authenticated users. Their access rights can be customised with access keys or special RBAC roles.

Given the special access requirements for storage, there are options to provide anonymous public access (for BLOBs) or restrict access with Active Directory (RBAC for BLOBs and Queues), Shared Access Signatures (not for Files) or shared key authentication.

With RBAC it is a good practice to restrict access to the absolute necessary containers or objects only.

### Shared Access Signatures 
Shared access signatures provide signed URIs to containers and BLOBs that give limited access to these resources for agents that may not have full access rights. Like users of a service that need read or write access to a stored object. This access can be limited to a container level, an object level and it can also be limited in time.

There are two basic architectures to create these signatures:
- A front end proxy can be placed between the client and the storage which can also validate some business rules. This could become a bottleneck when there is a lot of data though.
- There could also be some form of authentication provider that provides shared access signatures back to the client, which it can then use to access the storage directly.

Often both can be used at the same time for different access patterns.

Advice for using Shared Access Signatures (SAS):
- Always use HTTPS
- Use short-term expiry for the access even if the client has longer term rights (just renew automatically)
- Be careful with times. There may be some differences in time between machines.
- Be specific with which resources can be accessed - rather return multiple SAS for different resources.

Important data should be stored in write once BLOBs that are immutable to prevent modification.

There is also the option to enable Azure Advanced Threat detection. ([See the summary for CosmosDB for a more detailed description](./store_cosmos-db.md))

Use the Azure resource manager deployment model to get access to all of the security features. 


# Implementation

## Required Fields
- Subscription / Resource group
- account name

## Optional
- location
- kind = v1 & v2
- sku (performance + replication) = standard / premium + LRS, ZRS, (RA-)GRS, (RA-)GZRS
- access-tier = Cool, Hot
