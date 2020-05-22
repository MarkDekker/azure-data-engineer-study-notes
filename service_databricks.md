### Contents

- [Introduction](#introduction)
- [Workspace](#workspace)
- [More Benefits](#more-benefits)
- [How it Works](#how-it-works)
- [Security](#security)
- [Logs](#logs)

# Introduction

Databricks provides a notebook-oriented Apache Spark as-a-service workspace environment, making it easy to manage clusters and explore data interactively.

The Databricks Runtime adds several key capabilities to Apache Spark workloads that can increase performance and reduce costs by as much as 10-100x when running on Azure,

With the DataFrames API, the performance differences between languages are nearly nonexistence (especially for Scala, Java & Python)

To use your notebook to run a code, you must attach it to a cluster.

You can use Apache Spark notebooks to:
- Read and process huge files and data sets
- Query, explore, and visualize data sets
- Join disparate data sets found in data lakes
- Train and evaluate machine learning models
- Process live streams of data
- Perform analysis on large graph data sets and social networks

user management
git source code repository integration
user workspace folders

Optimised Spark runtime.

# Workspace

When talking about the Azure Databricks workspace, we refer to two different things. The first reference is the logical Azure Databricks environment in which clusters are created, data is stored (via DBFS), and in which the server resources are housed. The second reference is the more common one used within the context of Azure Databricks. That is the special root folder for all of your organization's Databricks assets, including notebooks, libraries, and dashboards,

# More Benefits

VM types: Many existing VMs can be used for clusters, including F-series for machine learning scenarios, M-series for massive memory scenarios, and D-series for general purpose.

Flexibility in network topology: Azure Databricks supports deployments into virtual networks (VNETs), which can control which sources and sinks can be accessed and how they are accessed.

Orchestration: ETL/ELT workflows (including analytics workloads in Azure Databricks) can be operationalized using Azure Data Factory pipelines.

Power BI: Power BI can be connected directly to Databricks clusters using JDBC in order to query data interactively at massive scale using familiar tools.

Azure Active Directory: Azure Databricks workspaces deploy into customer subscriptions, so naturally AAD can be used to control access to sources, results, and jobs.

Data stores: Azure Storage and Data Lake Store services are exposed to Databricks users via Databricks File System (DBFS) to provide caching and optimized analysis over existing data. Azure Databricks easily and efficiently uploads results into Azure Synapse Analytics, Azure SQL Database, and Azure Cosmos DB for further analysis and real-time serving, making it simple to build end-to-end data architectures on Azure.

Real-time analytics: Integration with IoT Hub, Azure Event Hubs, and Azure HDInsight Kafka clusters enables developers to build scalable streaming solutions for real-time analytics.

# How it Works

The "Databricks appliance" is deployed into Azure as a managed resource group within your subscription. This resource group contains the Driver and Worker VMs, along with other required resources, including a virtual network, a security group, and a storage account. All metadata for your cluster, such as scheduled jobs, is stored in an Azure Database with geo-replication for fault tolerance.

Internally, Azure Kubernetes Service (AKS) is used to run the Azure Databricks control-plane and data-planes via containers running on the latest generation of Azure hardware (Dv3 VMs), with NvMe SSDs capable of blazing 100us latency on IO. These make Databricks I/O performance even better. In addition, accelerated networking provides the fastest virtualized network infrastructure in the cloud. Azure Databricks utilizes this to further improve Spark performance.

Control (managed my Microsoft - you never see this) and Data Planes (only inside customer subscription - data is safe). Query results, logs and metadata do return to control plane for caching.

Information exchanged between this VNet and the Microsoft-managed Azure Databricks Control Plane VNet is sent over a secure TLS connection through ports (22 and 5557)

To clarify, you can write to the default DBFS file storage as needed, but you cannot change the Blob Storage account settings since the account is managed by the Microsoft-managed Control Plane. => only use the default storage for temporary files (the default file storage is tied to the lifecycle of your Azure Databricks account)

# Security

Encryption at-rest – Service Managed Keys, User Managed Keys
Encryption in-transit (Transport Layer Security - TLS is always on)
File/Folder Level access control lists (ACLs) for Azure Active Directory (AAD) Users, Groups, Service Principals
ACLs for Clusters, Folders, Notebooks, Tables, Jobs
Secrets with Azure Key Vault

Workspace is a fundamental isolation unit in Databricks. All workspaces are isolated from each other.


Access control - ADLS Passthrough
When enabled, authentication automatically takes place in Azure Data Lake Storage (ADLS) from Azure Databricks clusters using the same Azure Active Directory (Azure AD) identity that one uses to log into Azure Databricks. Commands running on a configured cluster will be able to read and write data in ADLS without needing to configure service principal credentials.

Only one user is allowed to run commands on this cluster when Credential Passthrough is enabled.

You can assign five permission levels to notebooks and folders: No Permissions, Read, Run, Edit, and Manage. All notebooks in a folder inherit all permissions settings of that folder.

As a best practice, instead of directly entering your credentials into a notebook, use Azure Databricks secrets to store your credentials and reference them in notebooks and jobs.

SCIM lets you use Azure Active Directory to create users in Azure Databricks and give them the proper level of access, as well as remove access for users (deprovision them) when they leave the organization or no longer need access to Azure Databricks.

Conditional access policies can restrict sign-in to your corporate network or can require multi-factor authentication.

The problem with using personal access tokens for the REST API is that your ability to create new tokens exists only after a workspace is created. Since the Azure Databricks workspace creation can only occur within the Azure portal, it is difficult to perform end-to-end automation without using Azure Active Directory tokens. 


Virtual network (VNet) peering allows the virtual network in which your Azure Databricks resource is running to peer with another Azure virtual network. Traffic between virtual machines in the peered virtual networks is routed through the Microsoft backbone infrastructure, much like traffic is routed between virtual machines in the same virtual network, through private IP addresses only.

If you're looking to do specific network customizations, you could deploy Azure Databricks data plane resources in your own VNet. In this scenario, instead of using the managed VNet, which restricts you from making changes, you "bring your own" VNet where you have full control. Azure Databricks will still create the managed VNet, but it will not use it.

Features enabled through VNet injection include:
- On-Premises Data Access
- Single-IP SNAT and Firewall-based filtering via custom routing
- Service Endpoint

# Logs

Databricks provides comprehensive end-to-end audit logs of activities performed by Databricks users, allowing your enterprise to monitor detailed Databricks usage patterns. Azure Monitor integration enables you to capture the audit logs and make then centrally available and fully searchable.




Runtimes

Pools

Autoscaling

Worker types

Driver? (Master)

Cluster modes