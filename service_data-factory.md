### Contents

- [Introduction](#introduction)
- [Mapping Data Flows](#mapping-data-flows)
- [Connecting to DataBricks](#connecting-to-databricks)
- [Pricing](#pricing)


# Introduction

Data Flow allows you to specify ETL jobs for serverless execution. This can be done completely interactively without having to write any code. It integrates with most Azure data storage and transformation services.

You get a nice user interface that shows the logical data flow and transformation steps and allows you to monitor these pipelines for failures.

There are three types of data factory activities that can be grouped togerther into pipelines:
- Movement
- Transformation
- Control

Data Factory supports almost all data storage options in Azure as sources and sinks (except MariaDB). It also supports a wide variety of other cloud and open source data stores as sources.

For data transformation it also supports the usual suspects:
- Spark
- MapReduce
- Hive
- Pig
- Hadoop Streaming
- U-SQL
- Stored Procedures in Azure SQL
- Data Flow (which is DataBricks managed by Data Factory)
- Custom

Finally, the control flow activities can include:
- Append variable
- Execute pipeline
- Filter
- For Each
- Get Metadata
- If
- Lookup External resource
- Set variable
- Until
- Validation - Looking at reference dataset
- Wait
- Web (Call REST)
- Webhook

Pipelines and activities can be saved JSON files.

Pipelines are triggered at specified time intervals, manually or from other pipelines. Triggers are defined independently from pipelines, which means that multiple triggers can start a single pipeline or one trigger can start multiple pipelines.

Integration Runtime - Where is the operation performed:
- Azure DF (Data flow, data movement, activity dispatch - send to a different compute engine to execute)
- Azure SSIS (within SQL or Synapse)
- Self-hosted

Publish all in the factory top bar. While not covered in this lab, Azure Data Factory supports full git integration. Git integration allows for version control, iterative saving in a repository, and collaboration on a data factory. => Full branching and commits etc from UI!

# Mapping Data Flows

Uses managed Spark clusters to perform data transformations. All without writing code.

Debug mode spins up a Spark cluster for interactive debugging - stays up for 60min. 

Can run activities in parallel on individual clusters.

Join types:
- Inner (both match)
- Left Outer (All first with second match)
- Right Outer (All second with first match)
- Full outer (All regardless of match)

NULL added for failing matches.

Partitioning:
- Round robin
- Hash (specific columns)
- Dynamic Range (Spark feature)
- Fixed Range
- Key (without hashing - unique values like categorical data)



Mapping data flows allow users to develop transformation logic code-free and execute them on spark clusters managed by the ADF service.

# Connecting to DataBricks

Normally New job cluster is preferred for triggered pipelines as they use a lower cost engineering tier cluster.

Linked services define the connection information needed for Data Factory to connect to external resources.

You can configure parameters by using widgets on the Databricks notebook. You then pass in parameters with those names via a Databricks notebook activity in Data Factory.

# Pricing

Data Factory Operations
    Read/Write 
    Monitoring 
Pipeline Orchestration & Execution 
    Activity Runs
    Data Movement Activities
    Pipeline Activity
    External Pipeline Activity