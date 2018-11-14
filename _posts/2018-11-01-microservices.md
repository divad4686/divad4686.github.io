---
layout: post
title: Data out of monolith
excerpt_separator: <!--more-->
hidden: true
---
A common mistake is thinking that creating a new API layer on top of your database is microservices. Microservices is not only about distributed systems in the network, the definition of [microservices](https://microservices.io/) is "a collection of loosely coupled services, which implement business capabilities"

Going from a monolith to a full microservices architecture can usually cause a [lot of problems](https://segment.com/blog/goodbye-microservices/), it is much better to it into smaller monolith little by little, starting with one domain to create a new service (micro-monolith?) with his own data storage. 

The hardest part about splitting is the data. The new service will have his own data store and should own the writes related to his domain, but some of this data will need to be shared back to the monolith, likewise the new service will also need data from the monolith. 

In this post I will focus on extracting data from the monolith that is required by the new service but not owned by it. 

For example, in my current project we are creating a new Notifications service. In this project we are eliminating all notification related tables from the main database because it will now be owned by the service. But there is data we need that it is not owned by the notifications domain. For example, we have the concept of favorite in our website and applications, this favorite help us decide who to send some of our notifications, but favorites is not a concept owned by the notifications domain. 

There are different ways to extract this type of data, they all have some pros and cons, some of them are generally better, but in the end it depends of your use case. 

There are two main ways to access the data needed from the monolith, you can synchronously request the data to the monolith when you need it, or you can store a copy of the data (a cache) you need from the monolith in your local database, in this case you will have to apply a strategy to synchronize your local db with the monolith DB.

# Accessing the data directly
Depending on the data you need to get, this may also put a heavy load in the main database, specially at peak time. The second problem is that you have a direct dependency on a system that is very prone to errors or going down (thats why you are splitting your monolith right?), this could take your microservice down very easily, return errors to your users, or worse, hang up until a timeout is approached.

There are two options for this solution, access the database directly from your microservice or create an api on top of the monolith.

## Connecting directly to the database
This is the most straightforward, just pass the connection string to your service and go directly to the source of the data you need. This pose some problems though.

First, any change in the database could break your service, you will be extremely coupled to the main database schema, taking away some of the benefits of creating a separate service, like independent deployments.

Testing can also be very difficult with this approach. In modern developments is increasingly more common to deploy your dependencies locally with docker, including recreating your database anytime you want, for example, when running integration tests. 

Chances are this is will be impossible with a monolithic database. Running the migration scripts on a 5 year behemoth could take a really long time, and filling test data in a database with tables spamming foreign keys around multiple places could be a complete nightmare. 

The advantage of this solution is that it is really fast to implement and fairly simple, making it less likely to fail, but also really hard to evolve.

## API on top of the database
This is a very common solution, it have the advantage over accessing directly the database that versioning at the API level is easier. Changes in the database schema can be handle in the API instead of breaking the new service accessing the data directly. 

It have the same problem of being a synchronous call, possible putting a heavy load on the database, and being prone network fails at critical moments. 

The testing can become a bit easier, by providing a 'sandbox' call where you return fake data. 

It is still a pretty fast and straightforward solution, but it adds a new point of failure for your application, meaning you will need more tests for the new API, and possible new deployments.

# Creating a local copy of the data
This solutions usually take more time to develop, and you will also have to deal with [Eventual consistency problems](https://en.wikipedia.org/wiki/Eventual_consistency). But they are also more resilience and less prone to failures.

This solutions usually 

## Enterprise solutions
Some databases vendors have solutions for this type of data extraction, but this solutions tend to be on their very expensive enterprise packages. For example, [Oracle GoldenGate](https://www.oracle.com/middleware/technologies/goldengate.html) allows you to stream data from a Oracle database to other systems. If you are using Azure SQL (there are solution)[https://docs.microsoft.com/en-us/azure/sql-database/sql-database-sync-data] to extract data out of it, but this could imply a tight vendor lock-in.

If you are using kafka, there is also a solution called (confluent)[https://www.confluent.io/]. They provide connectors to extract data out of a database to kafka, usually by streaming the event log of the engine to kafka, and also to unload data from kafka to another system. The main drawback of this solutions is that building and maintaining a kafka cluster can be hard and quite expensive in resources. Some of the connectors are also on early stage development, and they may not be reliable in a production environment. 

## ETL
## The outbox pattern 
This is a solution we are using in my current project. 



