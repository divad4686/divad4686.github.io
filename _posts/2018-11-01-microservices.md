---
layout: post
title: Data out of monolith
excerpt_separator: <!--more-->
hidden: true
---
Going from a monolith to a full microservices architecture can usually cause a [lot of problems](https://segment.com/blog/goodbye-microservices/), it is much better to it into smaller monolith little by little, starting with one domain to create a new service with his own data storage. But what data should you move to the new storage?

Usually the new service will own part o the data previously in the monolith, and it will need to be shared back to the monolith, likewise the new service will also have dependencies on data from the monolith. 
![](https://drive.google.com/uc?export=view&id=1U3rx1NGYSclNu6I3yP_oJsyHqdO74p1X)

In this post I will focus on extracting data from the monolith that is required by the new service but not owned by it.

For example, in my current project we are creating a new Notifications service. In this project we are moving all notification related tables from the main database to the new Notifications Domain database. But the new domain still have dependencies on data not owned by it. For example, we have the concept of favorite in our website and applications, this favorite help us decide who to send some of our notifications, but favorites data is not owned by the notifications domain. 
![](https://drive.google.com/uc?export=view&id=1B8B0fLVS3cCQl6-PyXFi7hDrD2h8Nra2)

There are different ways to extract this type of data, they all have some pros and cons, some of them are generally better, but in the end it depends of your use case. 

There are two main ways to access the data needed from the monolith, you can synchronously request the data to the monolith when you need it, or you can store a copy of the data (a cache) in your local database, in this case you will have to apply a strategy to synchronize your local DB with the monolith DB.

# Accessing the data directly
Depending on the data you need to get, this may also put a heavy load in the main database, specially at peak time. The second problem is that you have a direct dependency on a old complex system that could be very prone to errors (thats why you are splitting your monolith right?), this could take your microservice down very easily, return errors to your users, or worse, hang up until a timeout is triggered.

There are two options for this solution, access the database directly from your microservice or call an api on top of the monolith.

## Connecting directly to the database
This is the most straightforward solution, just pass the connection string to your service and go directly to the source of the data you need. This pose some problems though.

![](https://drive.google.com/uc?export=view&id=1wDNMrl7VpSiQtJXOTW2ekngv_pKn1uil)

First, any change in the database could break your service, you will be extremely coupled to the main database schema, taking away some of the benefits of creating a separate service, like independent deployments.

Testing can also be very difficult with this approach. In modern developments is increasingly more common to deploy your dependencies locally with docker, including recreating your database anytime you want, for example, when running integration tests. 

Chances are this is will be impossible with a monolithic database. Running the migration scripts on a 5 year behemoth could take a really long time, and filling test data in a database with tables spamming foreign keys around multiple places could be a complete nightmare. 

The advantage of this solution is that it is really fast to implement and fairly simple, making it less likely to fail, but also really hard to evolve.

## API on top of the database
This is a very common solution, it have the advantage over accessing directly the database that versioning at the API level is easier. Changes in the database schema can be handle in the API instead of breaking the new service accessing the data directly. 

![](https://drive.google.com/uc?export=view&id=1uPeZsVl06or0f6RQlP9jy0pKlC2Lpt0H)

It have the same problem of being a synchronous call, possible putting a heavy load on the database, and being prone network fails at critical moments. 

The testing can become a bit easier, by providing a 'sandbox' call where you return fake data. 

It is still a pretty fast and straightforward solution, but it adds a new point of failure for your application, meaning you will need more tests for the new API, and possible new deployments.

# Creating a local copy of the data
A very common solutions in the microservices world. Instead of synchronously accessing the monolith database, we create a copy of the data we need in a local database, and fill it asynchronously, either by loading directly from a database in a fixed time set, or by exporting the data from the monolith using a messaging or streaming system, i.e Kafka or RabbitMQ.

![](https://drive.google.com/uc?export=view&id=1RqotsXoWuBh9QhKrLOfSBZz_Qnmhw9Zr)

This solutions usually take more time to develop, and you will also have to deal with [Eventual consistency problems](https://en.wikipedia.org/wiki/Eventual_consistency). But they are also more resilience and less prone to failures.

## Syncing data at fixed times
It is a pretty straightforward solution, have a cron job or something equivalent (even a job triggered manually) query the database at fixed times, usually when the load on the main database is not so heavy, and dump the data in your local database. The main disadvantage of this method is that your data will be out of sync most of the time. This is a typical solution to extract data that doesn't change so much over time, like a countries table. 

![](https://drive.google.com/uc?export=view&id=1xCpamWEYmCbvm1S5_vUz9bsRTBL9i-RD)

## Streaming data out of the monolith
The most complex solution, but the best approach if you don't want to overload the monolith DB, and it can keep the consistency between the two system in a small time window. 

![](https://drive.google.com/uc?export=view&id=1xqP1bOPNLe8oBmL2BhlzE7NLTOZb2uIX)

Typically, the monolith exports the data as messages/events in some sort of streaming system, like kafka or rabbitMQ. Then have a consumer in the microservice reading the data and storing it in its local storage.

There are different techniques to publish messages from the monolith. You can publish your messages directly to the messaging system after storing the data in your database. Another solution is to connect directly to the database engine transaction log and publish this log to the messaging system. A third solution is to use the outbox pattern, storing the messages in another table in the monolith DB, and them publish this messages to the messaging system.



### Publishing the event from code
This is the easier to implement, but also the worst of the 3 solutions given. This is done in code by sending the message to the streaming service in the same block of code that you store in the database. 

``` 
UpdateDB(data);
SendToRabbit(data);
```
![](https://drive.google.com/uc?export=view&id=1DcJOWjxeo9Mx1IW4tVnMPb1nWlf6juLF)

The problem with this approach is that you can't guarantee the transaction between the two systems. Streaming services like RabbitMQ don't even support two phase commit. If you Send your message to rabbit first, but the database transaction fail, you have already send wrong data to the to the other systems. If you update the database first and publishing the message fails, you may never get the new data in your microservices, making the system inconsistent, and recovering from this type of consistency issues can be really hard.

### Using the database transaction log
Typically know as [transaction log tailing](https://microservices.io/patterns/data/transaction-log-tailing.html). In this solution you connect to to the transaction log of the database and publish the events that come from it (typically INSERT, UPDATE and DELETE events). 
This approach is usually very complex, DB engines don't tend to provide a easy way to read the transaction log (it is more of a internal implementation detail), so connecting to it can require low level code, making it a quite complex solution. The solution will also be tailored to the DB engine you are connecting to.

![](https://drive.google.com/uc?export=view&id=1WXkMp8ctoTqUX4fP6EvzjuwcJuxnOIHS)

Some databases vendors have solutions for this type of data extraction, but it tend to be on a very expensive enterprise packages. For example, [Oracle GoldenGate](https://www.oracle.com/middleware/technologies/goldengate.html) allows you to stream data from a Oracle database to other systems. If you are using Azure SQL (there are solution)[https://docs.microsoft.com/en-us/azure/sql-database/sql-database-sync-data] to extract data out of it, but this could imply a tight vendor lock-in.

If you are using kafka, there is also a solution called [confluent](https://www.confluent.io/). They provide connectors to extract data out of a database to kafka, usually by streaming the event log of the engine to kafka, and also to unload data from kafka to another system. The main drawback of this solutions is that building and maintaining a kafka cluster can be hard and quite expensive in resources. Some of the connectors are also on early stage development, and they may not be reliable in a production environment. 


### The outbox pattern 
This is a [solution](http://gistlabs.com/2014/05/the-outbox/) we are using in my current project. The idea is to have a outbox table in your DB where you will store the events you want to publish, and the storing of this event can be done in the same DB transaction where you are storing your actual data, giving a much better guarantee of keeping your data consistent, compared to publishing the event from code.

![](https://drive.google.com/uc?export=view&id=1Rt8TvbHOoNIDDVn1Sl42TYRQuE4zrCIa)

There are different ways to do this, if you are using a ORM you can insert in the outbox table while updating your data before committing the changes to the database. 

If you are using Stored Procedures you can insert in the outbox table there.

Another option is to capture the changes using a db trigger over the table that contains the data you want to take out, in this trigger you can store in the outbox table. This is a good solution when you have a big codebase, with multiple stored procedures, and don't really know for sure where is the data being updated.

After storing the outbox table, you will need a cron job to regularly read this table and publish the events to your streaming service, maybe extracting additional data when necessary. You have to be careful though, additional queries should only be done on indexed columns, so you don't overload the database.

