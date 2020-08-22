---
layout: post
title: Welcome to Futures
category: patterns
tags: [unity, csharp]
---

C#'s [Task&lt;TResult&gt;](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task-1) is a fantastic tool for passing around a handler to a procedure. It does not come without it's baggage though. My issue is that it requires the procedure be executed with threads, and that's a no-go for my purposes. Being that Unity works best with single threaded applications, I needed something similar but simpler. So I implemented the [Future](https://en.wikipedia.org/wiki/Futures_and_promises) pattern in a way that is friendly to Unity.

# Quick Facts #
**When do I use this?** Your environment is single threaded and utilizes a coroutine structure. Object A starts a long-running procedure on Object B. Object A wants to know when Object B has completed.

**How does this improve existing patterns?** It takes responsibility for the managing of callbacks, providing more straightforward code.

**Where is the code?** I posted all of the types to my [unity-utilities](https://github.com/jamesreiati/unity-utilities/tree/master/Assets/Futures) repository.

# Deep Dive #
As its simplest, Future is an event emitter that will also provide its status. You have an object (you can have more than one, but for this example we'll just stick with one) which waits for a specific task to complete called the observer. You also have an object which performs the task called the performer.

As an observer, I just want to know what the current status is (in case I want to poll) and how to subscribe. That's why Future implements `IObservableFuture`.

```c#
public interface IObservableFuture
{
    bool HasFulfilled { get; }

    event FulfillmentHandler OnFulfilled;
}
```

> **The Unity Inspector can't show interfaces. Doesn't this limit my ability to use this pattern?**  
> The object we are creating here is really meant to only ever be constructed during runtime. It's a promise that some procedure will return. Things in the inspector are meant to represent persistent data. Procedures are not persistable, and therefore are not surfaced in the inspector. You might want to try surfacing a UnityEvent in your object, which could take this interface as an argument.

As an observer, I can ask for an `IObservableFuture`, and am provided with polling data (through `HasFulfilled`), or a notification (through `OnFulfilled`).

As a task runner, I just want to signal when my future has been fulfilled. For this, we can either return or expose a future, and then fulfill it later.

```c#
private Future easeTask = null;

public IObservableFuture StartEasing()
{
    this.easeTask = new Future();
    return this.easeTask;
}

private void Update()
{
    // Do easing, and set easingIsDone if done.

    if (easingIsDone)
    {
        this.easeTask.Fulfill();
        this.easeTask = null;
    }
}
```

You'll notice that in order to keep the state of the future safe, I have nulled out the value after fulfilling it. I found myself doing this enough, that Future comes with a static method called `ClearAndFulfillIfUnfulfilled`. If you utilize this method every time you need to clear or fulfill a Future, you will be able to guarantee that consumers of this future will get their events invoked at least once.

I'd consider providing this guarantee as a best practice for this type. I wouldn't even use this type if you can't or don't want to provide that guarantee. In fact, this guarantee is so powerful that I wanted to provide it even when tasks would get cancelled. That's why I also implemented `IObservableFuture<T>` and associated classes.

```c#
public interface IObservableFuture<T>
{
    bool HasFulfilled { get; }

    event FulfillmentHandler<T> OnFulfilled;

    T Value { get; }
}
```

This is basically the same interface, but with the an additional T value that can be retrieved. For a `Future<T>`, when you fulfill a Future, you have to provide a T. For cancellable tasks, I make this T a bool where `true` corresponds to a completed task, and `false` corresponds to a cancelled task. The success state lives inside of the future, so even polling observers will be able to get the status after the Future has been fulfilled.

## Pitfalls ##

I've found there to be two main places this tool can go wrong, and they both are around this assumption of invocation guarantee.
 - **The program may exit before the Future gets fulfilled.** This actually may be intentional behavior, but be sure to think about this case when implementing anything which observes a future.
 - **An event handler may throw an exception, and this will prevent the remaining handlers from being invoked.** Events don't provide great isolation, in this case. I've been thinking of implementing a "safe" event which just invokes every delegate inside of a try/catch. But I haven't gotten around to this.


And that's basically all there is to this type. I tried to keep it minimal because ultimately, the patterns in which you use this are the powerful parts of this tool. I'll go over those in some future blog posts.

