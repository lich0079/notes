# Messaging System

|    | RabbitMQ | RocketMQ | Kafka | ActiveMQ | BES |
| -- | -------- | -------- | ----- | -------- | ------ |
| Push/pull | Push(prefetch limit),Pull(one message at a time)  | Pull  | Pull | 4  | Pull |
| Order | No in competing consumers, resolved by using the Consistent Hashing Exchange  | 2  | 3  | 4  | No  |
| Once | at most once, at least once  | 2  | 3  | 4  | At least Once |
| Transaction | 1  | 2  | 3  | 4  | 5  |
| Volume | 1  | 2  | 3  | 4  | 5  |
| Program language | 1  | 2  | 3  | 4  | 5  |
| Availability | 1  | 2  | 3  | 4  | 5  |
| Robust | 1  | 2  | 3  | 4  | 5  |
| Filter | Server side  | 2  | Client side  | 4  | Client side  |



pull has low latency




## RabbitMQ 


It is this routing capability that is its killer feature


### Scale

Using multi queues for parallel consume.

One queue can have multi consumers but the message processing will not have order guarantee.

If you need to have order guarantee also need scale, using Consistent Hashing. 

### Order between events

RabbitMQ allows you to maintain relative ordering across arbitrary sets of events

## Kafka

Commit Log because messages are stored in partitioned, append only logs which are called Topics. This concept of a log is the principal killer feature of Kafka.

### messages vs queues

Queue is another layer on top of message, provide routing feature for RabbitMQ, but also introduce complex on server side.


topic partition is the unit of parallelism in Kafka, partitions number decide the max parallelism

