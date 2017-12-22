---
layout: post
title: Aggregates with Marten
---

We have been using [Marten](https://github.com/JasperFx/marten) for event sourcing

If you want to apply two aggregates to the same stream you will hit your database for every aggregate, in some case it may be better to load your stream of events once and them apply your aggregates.



# Using dynamics and foreach
A simple way to do this is calling the correct apply method for each event with the help of the dynamic keyword we can correctly select with method to use for each type of event.

{% highlight c# %}
quest.Apply((dynamic)@event);
{% endhighlight %}