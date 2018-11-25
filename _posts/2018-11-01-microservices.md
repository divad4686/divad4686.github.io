---
layout: post
title: "Microservices techniques: Taking data out of the monolith"
excerpt_separator: <!--more-->
hidden: true
---
Going from a monolith to a full microservices architecture is [not an easy task](https://segment.com/blog/goodbye-microservices/), and one of the hardest part is [breaking the database](https://www.youtube.com/watch?v=MrV0DqTqpFU), choosing which data should be moved out of the monolith, and what should stay in it.

The new service will own data that previously belong to the original DB, but it will need to share some of it back to the monolith. Likewise the new service will also have dependencies on data that still belongs to the monolith. 
![](https://drive.google.com/uc?export=view&id=1U3rx1NGYSclNu6I3yP_oJsyHqdO74p1X)

In this post I will focus on the later problem, accessing monolith data required by the new service, but not owned by it.

For example, in my current project we are creating a new Notifications service. Here we are moving all notification related tables to the new notifications domain database. But the new domain still have dependencies on data not owned by it. For example, we have the concept of favorite in our website and applications, this favorite help us decide who to send some of our notifications, but favorites data is not owned by the notifications domain. 
![](https://drive.google.com/uc?export=view&id=1B8B0fLVS3cCQl6-PyXFi7hDrD2h8Nra2)

There are different ways to extract this type of data, they all have some pros and cons, some of them are generally better, but in the end it depends of your use case. 

There are two main ways to access the data needed from the monolith, you can synchronously request the data when you need it, or you can store a copy of the data (a cache) in your local database, in this case you will have to apply a strategy to synchronize your local DB with the monolith DB.


# Accessing data directly
Depending on the data you need to get, this may put a heavy load on the main database, specially at peak time. The second problem is that you have a direct dependency on an old complex system that could be very prone to errors (thats why you are splitting your monolith right?), this could take your microservice down very easily, return errors to your users, or worse, hang up until a timeout is triggered.

A couple of options here are accessing the database directly from your microservice, or call an api on top of the monolith.


## Connecting directly to the database
This is the most straightforward solution, just pass the connection string to your service and go directly to the source of the data you need. This pose some problems though.

![](https://drive.google.com/uc?export=view&id=1wDNMrl7VpSiQtJXOTW2ekngv_pKn1uil)

First, any change in the database could break your service, you will be extremely coupled to the main database schema, taking away some of the benefits of creating a separate service, like independent deployments.

Testing can also be very difficult with this approach. In modern developments is increasingly more common to deploy your dependencies locally with docker, including recreating your database anytime you want, for example, when running integration tests. 

Chances are this will be impossible with a monolithic database. Running the migration scripts on a 5 year behemoth could take a very long time, and filling test data in a database with tables spamming foreign keys around multiple places could be a complete nightmare. 

The advantage of this approach is that it is really fast and fairly simple to implement, making it less likely to fail, but also really hard to evolve.


## API on top of the database
This solution is very common, it have the advantage over the previous approach in that versioning is easier at the API level.

![](https://drive.google.com/uc?export=view&id=1uPeZsVl06or0f6RQlP9jy0pKlC2Lpt0H)

It have the same problem of being a synchronous call, possible putting a heavy load on the database, and being prone to network failures at critical moments. 

The testing can become a bit easier, by providing a 'sandbox' call where you return fake data. 

It is still a pretty fast and straightforward option, but it adds a new point of failure to your application, meaning you will need more tests for the new API, and possible new deployments.


# Asynchronously creating a local copy of the data
A very [common solutions](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/architect-microservice-container-applications/asynchronous-message-based-communication) in the microservices world. Instead of synchronously accessing the monolith database, we create a copy of the data we need in a local database, fill it asynchronously either by loading directly from a database in a fixed time (a cron job), or by exporting the data from the monolith using a messaging or streaming system, i.e RabbitMQ or Kafka.

![](https://drive.google.com/uc?export=view&id=1RqotsXoWuBh9QhKrLOfSBZz_Qnmhw9Zr)

This approach usually take more time to develop, and you will also have to deal with [Eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency) problems. But they are also more resilience and less prone to failures.


## Syncing data at fixed times
It is a pretty straightforward option, have a cron job or something equivalent (even a job triggered manually) query the database at fixed times, usually when the load on the main database is not so heavy, and dump the data in your local database. The main disadvantage of this method is that your data will be out of sync most of the time. This is a typical solution to extract data that doesn't change so much over time, like a countries table. 

![](https://drive.google.com/uc?export=view&id=1xCpamWEYmCbvm1S5_vUz9bsRTBL9i-RD)


## Streaming data out of the monolith
The most complex solution, but the best approach if you don't want to overload the monolith DB, and it can keep the consistency between the two system in a small time window. 

![](https://drive.google.com/uc?export=view&id=1xqP1bOPNLe8oBmL2BhlzE7NLTOZb2uIX)

Typically, the monolith exports the data as messages/events in some sort of streaming system, like kafka or rabbitMQ. Then have a consumer in the microservice reading the data and storing it in its local DB.

There are different techniques for publishing messages from the monolith. You can publish them directly to the messaging system after storing in your database. Another option is connecting directly to the database engine transaction log and publish this log to the messaging system. A third technique, the outbox pattern, consist in storing the messages in another the monolith DB table, and later be published by another process. A four solution would be a bit of the opposite of the outbox pattern, the monolith first publish the event to the broker, and then listen to the same event, triggering the storing of the data in the DB.


### Publishing the event from code
This is the easiest to implement, but also [the worst](https://jimmybogard.com/refactoring-towards-resilience-evaluating-coupling#rabbitmqcoupling) of the 3 given solutions. This is done in code by sending the message to the streaming service in the same block you store in the database.

``` 
UpdateDB(data);
SendToRabbit(data);
```
![](https://drive.google.com/uc?export=view&id=1DcJOWjxeo9Mx1IW4tVnMPb1nWlf6juLF)

The problem with this approach is that you can't guarantee the transaction between the two systems. Messaging systems like RabbitMQ don't even [support](https://tech.labs.oliverwyman.com/blog/2016/10/25/rabbitmq-and-transactions/) two phase commit. 

Then you have the problem of which order the transaction will be done. If you first send your message to rabbit, and after the database transaction fail, you have already send wrong data to the the other systems. If you update the database first but publishing the message fails, you may never get the new data in your microservices, making the data inconsistent between the two systems. Recovering from this type of issues can be really hard, or worse, they may not even be noticed at all.

The biggest problem with this is how do you handle bugs introduced in the code or network errors. Usually when you insert in the database, you can rollback this transaction, and in general you will have a retry mechanism in case an exception occur. If you publish to both systems in the same code block, retry policies become almost impossible to implement. What happen if you published to Rabbit but then you rollback the database change?.

For example, you could have an hotel reservation system that save a reservation in the database and then publish a message to rabbit, so the payment system can charge the user. If there is an error at some point, you may rollback the DB transaction, but already emitted the event to the payment system, charging the users credit card even though he will not have any reservation in your database.  


### Using the database transaction log
Typically know as [transaction log tailing](https://microservices.io/patterns/data/transaction-log-tailing.html). In this technique you connect to the transaction log of the database, publishing the events that come from it (for example, INSERT, UPDATE and DELETE events). 
This approach is usually very difficult to implement. DB engines don't tend to provide an easy way for reading the transaction log (it is more of a internal implementation detail), so connecting to it can require low level code, making it a quite complex solution. It will also be tailored to the DB engine you are connecting to.

![](https://drive.google.com/uc?export=view&id=1WXkMp8ctoTqUX4fP6EvzjuwcJuxnOIHS)

Some databases vendors have products for this type of data extraction, but it tend to be on a very expensive enterprise packages. For example, [Oracle GoldenGate](https://www.oracle.com/middleware/technologies/goldengate.html) allows you to stream data from a Oracle database to other systems. 

If you are using Azure SQL (there are options)[https://docs.microsoft.com/en-us/azure/sql-database/sql-database-sync-data] to extract data out of it, but this could imply a tight vendor lock-in.

If you are using kafka, there is also [confluent](https://www.confluent.io/). They provide connectors to extract data out of a database to kafka, usually by streaming the event log of the engine to kafka, and also to unload data from kafka to another system. The main drawback of this solutions is that building and maintaining a kafka cluster can be hard and quite expensive in resources. Some of the connectors are also on early development stage, and they may not be reliable in a production environment. 

A very similar option to confluent is [debezium](https://debezium.io/), an open source project from redhat, also consisting on connectors from the database transaction log to kafka.


### The outbox pattern 
[This](https://docs.particular.net/nservicebus/outbox/) is a [pattern](http://gistlabs.com/2014/05/the-outbox/) we are using in my current project. The idea is to have an outbox table in your DB where you will store the events you want to publish in the same DB transaction where you are storing your actual data, giving a much better guarantee of keeping data consistency, compared to publishing the event from code.

![](https://drive.google.com/uc?export=view&id=1Rt8TvbHOoNIDDVn1Sl42TYRQuE4zrCIa)

There are different ways to do this, if you are using an ORM you can insert in the outbox table while updating your data before committing the changes to the database. 

If you are using Stored Procedures you can insert in the outbox table there.

Another option is to capture the changes using a db trigger over the table that contains the data you want to take out, in this trigger you can store in the outbox table. This is a good option when you have a big codebase, with multiple stored procedures, and don't really know for sure where is the data being updated, but it should be treated as a last resource, triggers are usually hard to maintain, 

After storing the outbox table, you will need a cron job to regularly read this table and publish the events to your streaming service, maybe extracting additional data when necessary. You have to be careful though, additional queries should only be done on indexed columns, so you don't overload the database.


### Listen to yourself
[This technique](https://medium.com/@odedia/listen-to-yourself-design-pattern-for-event-driven-microservices-16f97e3ed066) consist on first publishing the data in a single transaction, and make the monolith listen to this event and then store the data in the database.

![](https://drive.google.com/uc?export=view&id=1AQSa4QDMlcC1SE8oWnOSuYo_FH1beNez)

This is a great technique for sharing data in microservices, but it may not be the best to extract data out of the monolith. Your system is probably made to store data synchronously in the database, and make it immediately available to the client. The changes from this pattern may spam some undesired side effects.


# What is the best solution? #
I think all the solutions have their use case, and they all should be taken into account when designing your system.

Synchronous options have to be done with care, they make your microservice more tightly couple to the monolith, and problems like network errors, versioning and code bugs will we easier to propagate from one system to another.

Asynchronous techniques are usually better for making your services more resilience and loosely couple, but they introduce more architecture and infrastructure complexity. Out of all the solutions, the only one I don't think should ever be done is publishing the events from code, the potential risk and errors introduced with this technique far outweighs the benefits of implementing it.