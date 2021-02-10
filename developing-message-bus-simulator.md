
Another very common pattern in applications and services is to use message passing, with at least once delivery semantics to perform some work. In Azure, teams often use services like Event Grid, Event Hub or Service Bus, for such use cases.

We'll explore how to create mocks for such services in this article.

Imagine a service which receives a request to provision a user account, but imagine that user account provisioning is a long running process where some background job is triggered to complete the account provisioning by firing off a message to service bus, which is picked up by aa worker running on a fleet of worker machines.

```csharp

public enum UserAccountState { Provisioning, Succeeded, Failed };

public class UserAccountManager
{
  private IDbCollection collection;
  private IMessageBus messageBus;

  public async Task<User> CreateUser(string userName, string details)
  {
    var user = new User() { Details = details, State = UserAccountState.Provisioning };

    try
    {
      await collection.CreateRow(user);
    }
    catch (RowAlreadyExists)
    {
      return null;
    }

    await messageBus.SendMessage(user);

    return user;
  }
}

public class BackgroundWorker
{
  public async Task ProvisionUser(User user)
  {
    // Provision resources for a user account, say dedicated Azure
    // Storage blobs etc in a background worker and update the
    // State of the User object to Succeeded (or Failed) at the
    // end.
  }
}

```

The above is a fairly common pattern in services where we provision a record in the database with a "Provisioning" state and send a message to, say service bus, which eventually triggers background work to provision the remaining resources which can take a long time (which is why its done in the background instead of inline with the request execution). The client can periodically poll for the status of the user to see if the provisioning succeeds or fails.

Production code typically uses SDKs provided by these messaging services, which periodically poll for the messages on the service bus and invokes a user registered lambda once a message is found.

How can we simulate the semantics and behavior of service bus in Coyote?

We'll start out by showing a solution which is _not_ a recommended way to do things. A lot of developers tend to write such code when starting out with Coyote so it is instructive to show case this solution before moving on to the recommended solution.

```csharp

public class MockMessageBus : IMessageBus
{
  private ConcurrentQueue<object> queue;

  public MockMessageBus(ConcurrentQueue<object> queue)
  {
    this.queue = queue;
  }

  public async Task SendMessage(object payload)
  {
    queue.Enqueue(payload);
  }
}

public class MockMessageListener
{
  private ConcurrentQueue<object> queue;

  public MessageListener(ConcurrentQueue<object> queue)
  {
    this.queue = queue;
  }

  public async Task StartListener()
  {
    var backgroundWorker = new Backgroundworker();

    while (true)
    {
      if (queue.TryDequeue(out object payload))
      {
        await backgroundWorker.ProvisionUser((User)payload);
      }

      await Task.Delay(100);
    }
  }
}

```

We model a `MockMessageBus` and `MockMessageListener` above, both of which share the same concurrent queue which simulates the service bus content. `MockMessageBus` adds a payload to the queue and `MockMessageListener` periodcally checks to see if there are messages available and invoeks the background worker method to provision the user if a message is there.

Here is how we can set up a test case using the above classes.

```csharp

public async Task Test()
{
  var queue = new ConcurrentQueue<object>();
  var messageBus = new MockMessageBus(queue);
  var messageListener = new MockMessageListener(queue);

  var collection = new MockDbCollection();

  var accountManager = new UserAccountManager(collection, messageBus);

  // start the event listener which will run in a while loop looking
  // for messages, forever
  _ = messageListener.StartListener();

  var success = await accountManager.CreateUser("joe", "...");
  Assert.IsTrue(success);

  User user;
  do
  {
    user = await accountManager.GetUser("joe");
    await Task.Delay(100);
  } while (user.State == UserAccountState.Provisioning)
}

```

The test sets up the classes, starts the message listener polling loop and then triggers the user creation request. It then waits in a loop till the provisioning is complete.

When you run the test, you'll notice that it runs for a _very long time_. It eventually does finish but runs for a long time. Your attention might go to the message listener loop which runs in a while loop, forever, looking for messages. That in fact is an anti-pattern in Coyote. Loops which run forever are generally a bad idea. The problem is not that Coyote gets hung because of an infinite loop - Coyote can in fact handle infinite programs just fine (see this [discussion](https://github.com) on liveness tests in Coyote). The problem is that infinite loops are generally a smell which you should avoid if possible. There are two distinct issues with such loops:

1. Coyote, by default, runs a program for a very large number of _steps_; it eventually terminates its exploration by runs for a longer time with infinite loops, usually doing nothing meaningful

2. Loops race with other tasks in the system. Most loops are just looking for some state to change (busy polling) and thus they increase the interleaving state space to explore without always adding much value

The second reason applies to all kinds of busy polling loops - we have two such loops in the test above. Busy polling in `MockMessageListener` and busy polling by the test while it waits for the user's provisioning to complete. The `MockMessageListener` loop starts at the very beginning of the test, and thus races with all the rest of the tasks in the test. The busy polling loop in the end of the test doesn't race with _as many_ concurrent tasks, and is bounded, so is less troublesome.

How can we improve our service bus mock above?

One possibility might be to signal `MockMessageListener` when some work is queued so its not busy polling all the time. That clearly works and is a perfectly fine thing to do as simulators can use such optimizations. 

There is a much easier solution however.

Since the work submitted to a service bus is _eventually_ executed by a worker, why don't we simulate that by just spawning off a concurrent task which invokes the background worker? Since our test runs under Coyote's control, Coyote will _eventually_ run that task. Our application code never peeks into the content of the queue in any case and just uses the queue to start work asynchronously so we don't really need to model the queue _at all_.

Here's a refined, much simplfiied (but equally effective) implementation of `MockMessageBus`.

```csharp

public class MockMessageBus : IMessageBus
{
  private BackgroundWorker worker = new BackgroundWorker();

  public async Task SendMessage(object payload)
  {
    // start work asynchronously
    _ = Task.Run(async () => await backgroundWorker.ProvisionUser((User)payload));
  }
}

```

The above is a much simpler implementation of the behavior we use service bus for. We were also able to eliminate `MockMessageListener` class completely. It's also fairly trivial to simulate "at least once" message delivery semantics, by generating a random number between 1 and 2, and creating one or two tasks to process the same message asynchronously.

The examples above show that we don't always have to simulate the exact machinery of an external service to model its behavior. Sometimes we can use much cheaper tricks and get the desired simulated semantics. We also learned why we should minimize the amount of loops in our code and tests and make them bounded wherever possible.

