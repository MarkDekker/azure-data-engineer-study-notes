### Contents

- [Basic Concepts](#basic-concepts)
- [Billing](#billing)
- [Important Aspects - Nuances](#important-aspects---nuances)
  * [Partitioning data](#partitioning-data)
  * [Job States](#job-states)
  * [Sliding Windows](#sliding-windows)
  * [Choices for picking event time:](#choices-for-picking-event-time-)
  * [Watermark](#watermark)
- [Backups](#backups)
- [Security](#security)

# Basic Concepts

- Fully serverless - managed
- Use SQL=like query language
- Can be extended with JS / C# user defined functions (Useful for creating deserialisers if the data is not in a suported format) They return scalar values!
- Plug and play
- Good performance with in-memory processing and scaling to multiple nodes
- Process millions of events per second
- Integrated solutions for anomaly detection
- Only UTF-8
- Geospatial Queries and built=in functions

Basic design:
- Input
- Query
- Output

Can join the input stream data with a slowly changing or static reference data set (kind of like a dimension table)

# Billing
Pay in SU - Streaming units (up to 192 Streaming Units), these scale compute and memory. 

Make sure you have enough memory, because it will fail your job if you run out. Monitor this. 

Try to keep SU consumption below 80% for best perfomance.

# Important Aspects - Nuances

## Partitioning data
- If your input data is partitioned, it is good to provide this partition key. This can then be used to parallelise the job. (IoTHUb, Eventhub and even BLOB Storage)
- 8 partitions great for SQL Database because it has 8 writers

## Job States

running, stopped, degraded, failed

## Sliding Windows
- Tumbling = Tumbling window functions are used to segment a data stream into distinct time segments
- Hopping = It may be easy to think of them as Tumbling windows that can overlap
- Sliding = Produce an output only when an event occurs => every window will have at least one event
- Session = Group events that arrive at similar times. A session window begins when the first event occurs. If another event occurs within the specified timeout from the last ingested event, then the window extends to include the new event. Otherwise if no events occur within the timeout, then the window is closed at the timeout. The session window will keep extending until maximum duration is reached


## Choices for picking event time:

**Arrival time - default**

Arrival time is assigned at the input source when the event reaches the source.

**Application time (also named Event Time)**

## Watermark

What stream analytics has seen / processed.

1. On incoming event: Largest event time - out-of-order tolerance window
1. No incoming event: current estimated arrival time - the late arrival tolerance window.

# Backups

Each time a Stream Analytics job runs, state information is maintained internally. That state information is saved in a checkpoint periodically. 

# Security

Communication is TLS encrypted and checkpoints are also encrypted.

