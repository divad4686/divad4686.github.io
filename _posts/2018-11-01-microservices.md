---
layout: post
title: Unit testing functional
excerpt_separator: <!--more-->
hidden: true
---
A common mistake is thinking that creating a new API layer on top of your database is microservices. Microservices is not only about distributed systems in the network, the definition of [microservices](https://microservices.io/) is "a collection of loosely coupled services, which implement business capabilities"

Going from a monolith to a full microservices architecture can usually cause a [lot of problems](https://segment.com/blog/goodbye-microservices/), it is much better to it into smaller monolith little by little, starting with one domain to create a new service (micro-monolith?) with his own data storage. 

The hardest part about splitting is the data. The new service will have his own data store and should own the writes related to his domain, but some of this data will need to be shared back to the monolith, likewise the new service will also need data from the monolith. 

In this post I will focus on extracting data from the monolith that is required by the new service but not owned by it. 

For example, in my current project we are creating a new Notifications service. In this project we are eliminating all notification related tables from the main database because it will now be owned by the service. But there is data we need that it is not owned by the notifications domain. For example, we have the concept of favorite in our website and applications, this favorite help us decide who to send some of our notifications, but favorites is not a concept owned by the notifications domain. 

There are different ways to extract this type of data, they all have some pros and cons, some of them are generally better, but in the end it depends of your use case. 

## Connecting directly to the database
This is the most straightforward, just pass the connection string to your service and go directly to the source of the data you need. This pose some problems though.

First, any change in the database could break your service, you will be extremely coupled to the main database, taking away a of the benefits of creating a separate service, like independent deployments. 

Testing can also be very difficult with this approach. In modern developments is increasingly more common to deploy your dependencies locally with docker, including recreating your database anytime you want, for example, when running integration tests. 

Chances are this is will be impossible with a monolithic database. Running the migration scripts on a 5 year behemoth could take a really long time, and filling test data in a database with tables spamming foreign keys around multiple places could be a complete nightmare. 