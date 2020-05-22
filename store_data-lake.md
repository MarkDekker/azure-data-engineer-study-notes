### Contents

- [Introduction](#introduction)
  * [Analogous Products](#analogous-products)
- [Target Use Cases](#target-use-cases)
- [Billing Method / SKUs](#billing-method---skus)
- [Important Aspects - Nuances (Performance)](#important-aspects---nuances--performance-)
  * [Hierarchical Namespace](#hierarchical-namespace)
- [Security](#security)

# Introduction

Azure Data Lake extends the functionality of Azure Storage to meet the demands of big data analytics and machine learning workloads. Gen2 merges the functionality of Azure Storage and Azure Data Lake Gen1.

You get the benefits of cheap, high availability and resilient storage with multiple perfomance tiers as well as the file system semantics and more granular security controls.

In the documentation they talk about storing petabytes and throughput of hundreds of gigabits. So big, big data...

Because the Data Lake storage emulates a hierarchical file system and POSIX access control lists, it works well as a storage option for a Hadoop cluster.

## Analogous Products

- Hadoop Storage

# Target Use Cases

- Big data
- As a hadoop file system

# Billing Method / SKUs

Storage costs the same, although transactions get ~20% more expensive. Note that if chosen for the correct application the savings on compute will far outweigh this...

# Important Aspects - Nuances (Performance)

## Hierarchical Namespace
The core addition of Data Lake storage is the hierarchical namespace. This gives file system performance with blob storage scale.

This works by organising blobs into a hierarchy of nested forders (same as a file system). 

The convention prescribed for Azure storage to deal with the fact that it is not possible to have nested containers, is to use slashes in file names to make the data appear as if it were nested in a file system. This works for most applications except when you need to reorganise the file hierarchy frequently. This is extremely expensive with this convention since all files need to be renamed. A hierarchical file system by contrast manages the folder "location" separately from the file name, making these operations cheap and easy.

Traditional hierarchical filesystems degrade in performance as they scale. Data Lake does not...

Hierarchical namespaces are not supported with the full set of functionality and cannot be changed back to a flat namespace. Use it for workloads that are designed to work on filesystems.


# Security

 Most of the security features are in line with all of the other Azure Storage options. However, Azure DataLake also adds support for access control lists (ACLs) that are POSIX (UNIX) compliant. This allows administrators to set very fine-grained control restrictions on what can be accessed by each user. An important feature for company-wide analytics applications.

 Access control lists grant access only to authorised users, groups and service principals on a file level.

 This also means that we can require multi-factor authentication and advanced Azure Active Directory features.

 > Reccommendation: Use security groups instead of administrating individual users. Basically what you do in UNIX. 

