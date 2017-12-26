---
layout: post
title: Aggregates with Marten
excerpt_separator: <!--more-->
hidden: true
---

In my current project we have been using [Marten](https://github.com/JasperFx/marten) as an event sourcing database.

Marten is a great library that have saved us a lot of time and pain, but we encounter some small use case where we changed the way we create our aggregates. If you want to generate two or more aggregates on the same stream and same transaction using the [Aggregate function](http://jasperfx.github.io/marten/documentation/events/projections/#sec4) you will hit your database multiple times, making it in some case better to just load your stream of events once and them create your aggregates from there.

Lets see how we improved this by using a fantasy quests example, similar to the one in [marten documentation](http://jasperfx.github.io/marten/documentation/events/) for the Event Store.

Each stream represents a quest, and it can hold the [following events](https://github.com/divad4686/marten-example/tree/master/src/quest/Models): QuestStarted, QuestEnded, MembersJoined, MembersDeparted, ArrivedAtLocation, PigSlayed and BossSlayed.

For our reading side, we have two aggregates, [QuestParty](https://github.com/divad4686/marten-example/blob/master/src/quest/Aggregates/QuestParty.cs), with general information like the current members of the quest, and [MonstersSlayed](https://github.com/divad4686/marten-example/blob/master/src/quest/Aggregates/MonstersSlayed.cs), containing the pigs and bosses we have killed so far.

The typical Marten implementation for this aggregates would be something like this:
{% highlight c# %}
public class QuestParty
{
    private readonly IList<string> _members = new List<string>();

    public string[] Members
    {
        get
        {
            return _members.ToArray();
        }
        set
        {
            _members.Clear();
            _members.AddRange(value);
        }
    }

    public void Apply(MembersJoined joined)
    {
        _members.Fill(joined.Members);
    }

    public void Apply(MembersDeparted departed)
    {
        _members.RemoveAll(x => departed.Members.Contains(x));
    }

    public void Apply(QuestStarted started)
    {
        Console.WriteLine(started.Name);
        Name = started.Name;
    }


    public string Name { get; set; }

    public Guid Id { get; set; }

    public override string ToString()
    {
        return $"Quest party '{Name}' is {Members.Join(", ")}";
    }
}
{% endhighlight %}
{% highlight c# %}
public class MonstersSlayed
{
    public Guid Id { get; set; }
    public int PigsKilled { get; private set; }
    public List<string> BossNames { get; private set; } = new List<string>();

    public void Apply(PigSlayed @event) => PigsKilled += 1;

    public void Apply(BossSlayed @event) => BossNames.Add(@event.Name);
}
{% endhighlight %}

Now lets say we have a asp.net core [controller](https://github.com/divad4686/marten-example/blob/master/src/quest/Controllers/QuestContoller.cs), and we want to get both the general quest information and the monsters slayed:
{% highlight c# %}
[HttpGet("questInfo/{id}")]
public async Task<IActionResult> Get(Guid id)
{
    using (var session = _store.OpenSession())
    {
        var quest = await session.Events.AggregateStreamAsync<QuestParty>(id);
        var monsters = await session.Events.AggregateStreamAsync<MonstersSlayed>(id);
        return Ok(new { Quest = quest, Monsters = monsters });
    }
}
{% endhighlight %}
Here we encounter the possible issue we discussed early, we are doing two round trips to the network and database to get the data. 

To reduce the network trips, we could get the events only once from the database and then generate the aggregates in memory.

Getting our stream events is quite simple:

{% highlight c# %}
var events = (await session.Events.FetchStreamAsync(id)).Select(@event => @event.Data).ToList();
{% endhighlight%}

Now we need to construct the aggregates.

# Using dynamics and foreach
A simple way to do this is calling the correct apply method for each event, but all we have is a list of objects, and we don't know at compilation time which apply function to use. To solve this we can use the dynamic keyword to select the correct method.

{% highlight c# %}
var quest = new QuestParty { Id = questId };
events.ForEach(@event => quest.Apply((dynamic)@event);
{% endhighlight %}

This have a big problem though, the apply method will throw an exception if it can't find it for the concrete type we pass as parameter, a way to mitigate this would be using try-catch:
{% highlight c# %}
try
{
    quest.Apply((dynamic)@event);
}
catch { }
{% endhighlight %}

We can move this code to a static function inside the Aggregate class:
{% highlight c# %}
public static QuestParty Aggregate(Guid id, List<object> events)
{
    var quest = new QuestParty { Id = id };
    events.ForEach(@event =>
    {
        try
        {
            quest.Apply((dynamic)@event);
        }
        catch { }
    });
    return quest;
}
{% endhighlight %}

And our updated controller:
{% highlight c# %}
[HttpGet("questInfoDynamic/{id}")]
public async Task<IActionResult> GetWithDynamic(Guid id)
{
    using (var session = _store.OpenSession())
    {
        var events = (await session.Events.FetchStreamAsync(id)).Select(@event => @event.Data).ToList();
        return Ok(new
        {
            Quest = QuestParty.Aggregate(id, events),
            Monsters = MonstersSlayed.Aggregate(id, events)
        });
    }
}
{% endhighlight %}
# Try catch logic?
This code have some flaws, the use of try catch in this case is pretty inefficient, verbose, and hard to read and maintain. 

Lets use some functional and defensive programming techniques to protect our code from errors, and make it easier to test and maintain.
# Pattern matching
Instead of having different Apply functions, we can use the c# 7 feature for [pattern matching](https://docs.microsoft.com/en-us/dotnet/csharp/pattern-matching), using the type of the object to select the correct Apply function, and skipping in case there is no match for the event we are tying to apply:
{% highlight c# %}
switch (@event)
{
    case MembersJoined joined:
        state._members.Fill(joined.Members);
        break;
    case MembersDeparted departed:
        state._members.RemoveAll(x => departed.Members.Contains(x));
        break;
    case QuestStarted started:
        state.Name = started.Name;
        break;
}
{% endhighlight %}
We can put this function inside a static method, and use the Linq aggregate function to construct our own:
{% highlight c# %}
private static QuestParty Aggregator(QuestParty state, object @event)
{
    switch (@event)
    {
        case MembersJoined joined:
            state._members.Fill(joined.Members);
            break;
        case MembersDeparted departed:
            state._members.RemoveAll(x => departed.Members.Contains(x));
            break;
        case QuestStarted started:
            state.Name = started.Name;
            break;
    }
    return state;
}

public static QuestParty Aggregate(List<object> events) => events.Aggregate(new QuestParty(), Aggregator);
{% endhighlight %}
Notice that we are using an empty constructor for QuestParty. Since we are not using Marten Aggregate anymore, we can remove the id attribute from our class.

We can also make the constructor private, so we only allow creations of the aggregate with a List of Events through our static method.

As a side note, marten [allows](http://jasperfx.github.io/marten/documentation/events/projections/#sec6) to make the Apply methods private, but I'm not sure that you can do the same with the constructor or id attribute.

The final version of the aggregates:

{% highlight c# %}
public class QuestParty
{
    private QuestParty() { }
    private List<string> _members = new List<string>();

    public string[] Members
    {
        get
        {
            return _members.ToArray();
        }
    }

    public string Name { get; private set; }

    public override string ToString() => $"Quest party '{Name}' is {Members.Join(", ")}";

    private static QuestParty Aggregator(QuestParty state, object @event)
    {
        switch (@event)
        {
            case MembersJoined joined:
                state._members.Fill(joined.Members);
                break;
            case MembersDeparted departed:
                state._members.RemoveAll(x => departed.Members.Contains(x));
                break;
            case QuestStarted started:
                state.Name = started.Name;
                break;
        }
        return state;
    }

    public static QuestParty Aggregate(List<object> events) => events.Aggregate(new QuestParty(), Aggregator);
}
{% endhighlight %}
{% highlight c# %}
public class MonstersSlayed
{
    private MonstersSlayed() { }
    public int PigsKilled { get; private set; }
    public List<string> BossNames { get; private set; } = new List<string>();

    private static MonstersSlayed Aggregator(MonstersSlayed state, object @event)
    {
        switch (@event)
        {
            case PigSlayed pig:
                state.PigsKilled += 1;
                break;
            case BossSlayed boss:
                state.BossNames.Add(boss.Name);
                break;
        }
        return state;
    }

    public static MonstersSlayed Aggregate(List<object> events) => events.Aggregate(new MonstersSlayed(), Aggregator);
}
{% endhighlight %}
If you need to create an aggregate from a single event you can overload the aggregate function:
{% highlight c# %}
public static QuestParty Aggregate(object @event) => Aggregate(new List<object> { @event });
{% endhighlight %}
Our controller looks similar from our last version, we only remove the need to pass the id multiple times as a parameter:
{% highlight c# %}
[HttpGet("questInfoFunctional/{id}")]
public async Task<IActionResult> GetFunctional(Guid id)
{
    using (var session = _store.OpenSession())
    {
        var events = (await session.Events.FetchStreamAsync(id)).Select(@event => @event.Data).ToList();
        return Ok(new
        {
            Quest = QuestParty.Aggregate(events),
            Monsters = MonstersSlayed.Aggregate(events)
        });
    }
}
{% endhighlight %}
# Immutability

From outside our aggregates are immutables, we can not construct or modify them, only make new ones with our static aggregate function, but will the Haskell gods be happy with this implementation? I don't think so! we are still modifying the same object inside our Aggregator function. To make it really immutable we should create a new object in each case of the pattern matching, maybe using the [fluent builder pattern](http://blog.ploeh.dk/2017/08/21/generalised-test-data-builder/)

I don't like this option because it makes the code really verbose. It would be nice if C# had a more elegant 'copy and update' method like [F#](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/copy-and-update-record-expressions), [Haskell](https://stackoverflow.com/questions/14955627/shorthand-way-for-assigning-a-single-field-in-a-record-while-copying-the-rest-o), [Kotlin](https://kotlinlang.org/docs/reference/data-classes.html#copying), [Clojure](https://clojuredocs.org/clojure.core/update), [and](https://doc.rust-lang.org/book/first-edition/structs.html#update-syntax), [many](https://realworldocaml.org/v1/en/html/records.html#functional-updates), [others](https://docs.scala-lang.org/tour/case-classes.html#copying).

# When to use this?
The technique I show above have some advantages over using Marten aggregate feature, like immutability and making only a single call to our database. But it also have some drawbacks.

When using Marten aggregate function, the library does some query optimization, using reflection to check what events have the apply method, and only returning this types of events from the database. In our case, we always need to return all the events from our stream and filter them in memory. If you have really big streams with a lot of different events, is probably better to use Marten aggregate, but if not, you can use the option shown here. Reducing the number of network and database round trips you are doing is usually a better trade off than increasing the amount of data you are transferring and querying in this trips. The question becomes how many events is considered to be "a lot".

You can check the code used for this example in [this repository](https://github.com/divad4686/marten-example/).
Clone, execute run.sh and check it at http://localhost:82/swagger

Many thanks to [@jeremydmiller](https://twitter.com/jeremydmiller) and the rest of [contributors](https://github.com/JasperFx/marten/graphs/contributors) of marten for such an awesome library!
