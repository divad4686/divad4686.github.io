---
layout: post
title: "CQS, CQRS, eventual consistency, what do this all mean?"
excerpt_separator: <!--more-->
hidden: false
---
With the rise and evolution of [microservices architectures](https://microservices.io/), there has been some new concepts being develop in the past few years, and also old concepts that are starting to get traction again.

But what do this concepts actually mean? how are they related to each other? 

Here I'm trying to give my vision of this concepts, from my research on how they are used, specially by whom I consider to be some of the leaders on the field. 

## What is CQS?

CQS is a principle that state that you should either have void methods that change state (commands) but don't return any value, or query methods that return a value but should not modify the state.

An example of CQS is IEnumerable interface in C#. IEnumerable have a MoveNext() command that modifies the state but returns nothing, and a query Current() method that returns the current element, but doesn't modify the state of the collection.

An example of something not doing CQS is the typical Queue implementation, where you have a Pop() method that remove the top element of the queue, modifying the state, and it also returns the value of this element. 

Check this great Gregg Young talk [CQRS and Event Sourcing](https://www.youtube.com/watch?v=JHGkaShoyNs), specifically minute 45:40 where I got this examples from.

## CQRS
[CQRS](https://martinfowler.com/bliki/CQRS.html) pattern main idea is that responsibility of commands and queries should be in different modules. 

Different modules can mean different assemblies in your project, it can also be different services in a microservices architecture or it can just be different classes inside your application, even if it is one singe monolith.

## Do you need 2 databases to apply CQRS?
![](https://drive.google.com/uc?export=view&id=17WETpbeYTGyd1rAIwCsuXmg43N5GhtkF)
You don't, the pattern can be applied using the same database, it can even be with the same data model. The pattern only dictates to separate the services where you do the commands and the queries. Now, it is very natural that since the queries and commands are separated, that you also use two different data models for them.


## CQRS and eventual consistency
![](https://drive.google.com/uc?export=view&id=19Z8VYL8PMNMiJH5spwlQ9DXUETRB3T6D)
In a microservices application, it is very common to have a service sending commands over a messaging system like RabbitMQ, and later another process picking this message up, writing to a user facing read database, and later have some user execute queries against this database. 

In this type of architecture you are already applying CQRS, your commands are being processed by a separate service over the network than the service handling your queries. Also, when using a messaging system, you are naturally using the eventual consistency distributed model.

![CQRS in a microservices system](https://drive.google.com/uc?export=view&id=19Kd2GbUKecfrO9qe90o3At6owYG51MoV)
This type of architecture has become so common that is very easy to associate CQRS with eventual consistency, but this concepts are completely different, and as we saw before, you can apply CQRS in a monolith application over a single, strongly consistent database engine.

## So, what is eventual consistency?
[It is a distributed system model](https://en.wikipedia.org/wiki/Eventual_consistency). In a strong consistent model,  after a transaction is executed against the system, and it response back with a success, you are guarantee that the entire system will immediately contain the changes made. This is what can be seen in a SQL database with ACID guarantees, with replicas across the network.
![](https://drive.google.com/uc?export=view&id=17_3ECWK-Ie5lTX71zuvWrR-HIRnmjIAY)

In eventual consistency, when a transaction is committed to the system and it respond back with a success, there is not guarantee that the data will be immediately available in the entire system, there is only a guarantee that the whole system will be consistent at some point in the future.
![Strong vs eventual consistency](https://drive.google.com/uc?export=view&id=1RSaSgQkFCXYxWwDIe09EKJP7B1Uxe7ni)

But this model does provide some great benefits, mainly higher availability and lower latency compared to strong consistent models. Additionally to this, strong consistency [techniques](http://thesecretlivesofdata.com/raft/) are a lot harder to implement, or near impossible in the case of microservices systems where you are dealing with different engines like RabbitMQ and a SQL database. Eventual consistent guarantees are usually good enough for most microservices systems.

But this model can cause issues in determined use cases. An example would be when a query needs to do done in the system immediately after executing a command, since the query may not return the updated state from the previous command.
![](https://drive.google.com/uc?export=view&id=1vTJECK-HxTkuxX5KAAm4xixVH9fROPdJ)

An example where eventual consistency is not an issue could be when you are importing data from an external system, and updating a read side database. Usually there is no need to have the data available to the user immediately after it is inserted in the system, a delay of a few seconds is perfectly fine for this kind of applications.
![](https://drive.google.com/uc?export=view&id=1JdyKYK2G3PvsKvd_QFi6f3pKldBwYjOR)

## And event sourcing?
[Event sourcing](https://www.youtube.com/watch?v=8JKjvY4etTY) consist on storing the state of your application as an immutable log of events, instead of (for example) using a relational model. This concept is typically associated with microservices, mostly because it is common to use [events to propagate changes](https://en.wikipedia.org/wiki/Event-driven_architecture) in your application across different microservices. 

But it is perfectly viable to have a single monolith with one database that use event sourcing to persist the application state.