---
layout: post
title: Welcome to Futures
category: patterns
tags: [unity, csharp]
---

C#'s [Task&lt;TResult&gt;](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task-1) is a fantastic tool for passing around a signal. It does not come without it's baggage though. I avoid using it with Unity because it uses a lot of C# concepts built for threads, and that's a no-go for my purposes. Being that Unity works best with single threaded applications, I needed something similar but simpler. So I implemented the [Future](https://en.wikipedia.org/wiki/Futures_and_promises) pattern in a way that is friendly to Unity. In this post, I'm going to describe what this tool looks like, and some best practices I've discovered when using it. Look for a future post for information on how I use this in place of `System.Threading.Task<T>`.

# Quick Facts #
**When do I use this?** Your environment is single threaded and utilizes a coroutine structure. Object A wants to exposes a single use signal to Object B.

**How does this improve existing patterns?** It takes responsibility for the managing of event lifetime, providing more straightforward code.

**Where is the code?** I posted all of the types to my [unity-utilities](https://github.com/jamesreiati/unity-utilities/tree/master/Assets/Futures) repository.

# Deep Dive #
As its simplest, Future is an instance of an event emitter that will also provide its status. You have an object called the observer (you can have more than one, but for this example we'll just stick with one) which waits for a specific signal. You also have an object called the performer which provides access to, and invokes the signal.

As an observer, I just want to know what the current status is (in case I want to poll) and how to subscribe. That's why Future implements `IObservableFuture`.

```c#
public interface IObservableFuture
{
    /// True if the future has been fulfilled. False if it has not.
    bool HasFulfilled { get; }

    /// An event invoked when the value has been fulfilled.
    event FulfillmentHandler OnFulfilled;
}
```

> **The Unity Inspector can't show interfaces. Doesn't this limit my ability to use this pattern?**  
> The object we are creating here is really meant to only ever be constructed during runtime. It's a promise that some procedure will return. Things in the inspector are meant to represent persistent data. Procedures are not persistable, and therefore are not surfaced in the inspector. You might want to try surfacing a UnityEvent in your object, which could take this interface as an argument.

As an observer, I can ask for an `IObservableFuture`, and am provided with polling data (through `HasFulfilled`), or a notification (through `OnFulfilled`).

As a performer, I just want to signal when my future has been fulfilled. For this, we can either return or expose a future, and then fulfill it later.

```c#
private Future signal = null;

public IObservableFuture GetAlert()
{
    this.signal = new Future();
    return this.signal;
}

private void Update()
{
    if (readyToAlert)
    {
        this.signal.Fulfill();
        this.signal = null;
    }
}
```

You'll notice that in order to keep the state of the future safe, I have nulled out the value after fulfilling it. I found myself doing this often enough that Future comes with a static method called `ClearAndFulfillIfUnfulfilled`. If you utilize this method every time you need to clear or fulfill a Future, you will be able to guarantee that consumers of this future will get their events invoked at least once. I'd consider providing this guarantee as a best practice for this type. I wouldn't even use this type if you can't or don't want to provide that guarantee.

There are times where you want your signal to come with additional information. For this, I've implemented `IObservableFuture<T>`.

```c#
public interface IObservableFuture<T>
{
    bool HasFulfilled { get; }

    event FulfillmentHandler<T> OnFulfilled;

    /// The value of the future if it has fulfilled. Will throw if it has not fulfilled.
    T Value { get; }

    /// The value of the future if it has fulfilled, or the default value for type T if it has not.
    T GetValueOrDefault();
}
```

This is basically the same interface, but with the an additional T value that can be retrieved. For a `Future<T>`, when you fulfill a Future, you have to provide a T. Value will throw an exception if accessed before the Future has been fulfilled, so I've provided an interface similar to `Nullable<T>`. With `GetValueOrDefault()` you can avoid double accessing `HasFulfilled` if you've already checked for it. As another best practice, I would recommend you only use immutable types as your value. If you fulfill the Future with a mutable type, it's entirely possible some of your observers could receive a different Value than other observers.

So to summarize how to use this object:
 - Always Fulfill your Future, guaranteeing\* that the notification will be invoked.
 - Always null-out any performer references to the Future once it has been fulfilled.
 - Use immutable types when using `Future<T>`.

> \* Even if you do always fulfill the future, an event handler may throw an exception, and this may prevent remaining handlers from being invoked. This is a general problem with using C#'s events, and this type doesn't provide relief from that issue.

And that's basically all there is to this type. I tried to keep it minimal because ultimately, the patterns that come with this are more powerful than the object itself. I'll go over those patterns in future blog posts!

