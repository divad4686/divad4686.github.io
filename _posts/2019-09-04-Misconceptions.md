---
layout: post
title: "Misconceptions"
excerpt_separator: <!--more-->
hidden: true
---

# What is CQS
I'm pretty much going to transcript from a Gregg Young talk [CQRS and Event Sourcing](https://www.youtube.com/watch?v=JHGkaShoyNs), skip to minute 45:40 if you want to check this description.

CQS state that you should either have void methods that change state (commands), or query methods that return a value but should not modify state.

An example of CQS is IEnumerable in C#. IEnumerable have a MoveNext() command that modifies the state but returns nothing, and a query Current() method that returns the current element, but doesn't modify the state of the collection.

An example of something not doing CQS is the typical Queue implementation, where you have a Pop() method that remove the top element of the queue, modifying the state, and it also returns the value of this element. 

# CQRS
[CQRS](https://martinfowler.com/bliki/CQRS.html) pattern main idea is that responsibility of commands and queries should be in different modules. 

Different modules can be different assemblies in your project, different services in a microservices architecture but it can also just mean different classes inside your application, even if it is one singe monolith.

### Do you need 2 databases to apply CQRS?
![](https://drive.google.com/uc?export=view&id=17WETpbeYTGyd1rAIwCsuXmg43N5GhtkF)
You don't, you can apply the pattern using the same database, you could even be using the same data model, the pattern only dictated to separate the services where you do the commands and the queries. Now, it is very natural that since you are separating the queries and commands, that you also use two different data models for then.


# CQRS and eventual consistency
![](https://drive.google.com/uc?export=view&id=19Z8VYL8PMNMiJH5spwlQ9DXUETRB3T6D)
In a microservices application, it is very common to have a service sending commands over a messaging system like RabbitMQ, and later another process picking this message up, writing to a user facing read database, and later the user executing queries against this database. In this type of architecture you are already applying CQRS, your commands are being processed by a separate service over the network that the service handling your queries. Also, when using a messaging system, you are naturally using the eventual consistency distributed model.

![CQRS in a microservices system](https://drive.google.com/uc?export=view&id=19Kd2GbUKecfrO9qe90o3At6owYG51MoV)
This type of architecture has become so common that is very easy to associate CQRS with eventual consistency, but this concepts are completely different, as we saw before, you can apply CQRS in a monolith application over a single, strong consistent database engine.

# So, what is Eventual consistency?
[It is a distributed system model](https://en.wikipedia.org/wiki/Eventual_consistency) 

that can cause issues in determined use cases. The typical example when it can be an issue is when you need to do a query in the system immediately after executing a command. In a eventual consistent system, the query may not return the updated state from the previous command.
![](https://drive.google.com/uc?export=view&id=1vTJECK-HxTkuxX5KAAm4xixVH9fROPdJ)

An example where eventual consistency is not an issue could be when you are importing data from an external system, and updating a read side database. Usually you don't need the data available to the user immediately after it is inserted in the system, a delay of 2,3 seconds is probably irrelevant in this situations.
![](https://drive.google.com/uc?export=view&id=1JdyKYK2G3PvsKvd_QFi6f3pKldBwYjOR)

