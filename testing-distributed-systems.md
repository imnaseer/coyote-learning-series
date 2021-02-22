[[_TOC_]]

All distributed systems are concurrent but not all concurrent programs are distributed in nature. There are two broad classes of concurrent programs:

* In-process concurrency
* Inter-process concurrency

An example of in-process concurrency is a GUI application with multiple tasks reading and writing to shared data and UI components. In-process concurrency is challenging but it is simpler than inter-process concurrency. An example of inter-process concurrency is a distributed service running controllers on a bunch of machines for scalability, all reading and writing shared state in a common database.

We'll refer to applications exhibiting inter-process concurrency as 'distributed systems' from here on out.

# Challenges when building distributed systems.

In-process concurrency within a single machine vs. inter-process concurrency across a fleet of machines share a lot of similar challenges but inter-process concurrency shares a few challenges which are unique to it. Let's discuss some of these challenges.

## Duplicate and concurrent processing of messages

Your clients may be using a retry library, like Polly, for automated retries if they don't hear from you. Their original request may not have reached you due to a network delay causing them to retry the request. As luck would have it, the network glitch gets fixed and you suddenly get two requests with the same payload within a millisecond of each other, and both of them run concurrently with each other on the same or different machines.

Distributed services often employ messaging queues (such as Kafka, Azure Event Grid, Azure Service Hub etc) which typically provide at-least-once delivery semantics. While they guarantee the message will definitely be delivered at-least once, it might be delivered twice, or more. It is entirely possible that duplicate messages may be processed within a very short time period and thus race with each other, running concurrently on the same or different machines.

The above conditions are extremely hard to reproduce and test for. They happen rarely but they do happen and if your system can get in a corrupted state if you don't account for them.

## Network timeouts (and associated uncertainty)

Say you call a server and the request fails with a network timeout on your side. The server must be down, right? 

Well, not necessarily.

The following may also have happened:

* The network connectivity from your side to the server was broken and your request never reached the server
* Your request reached the server just fine, the server executed it but its response never reached out due to a network glitch between your machine and the server

Network timeouts are tricky as you just know the request timed out _from your perspective_. You don't really know why though. The last reason is a tricky one which most of us don't account for. It's also notoriously hard to test for as the network connection only breaks when the response is coming back but not when the request is going to the server.

## Prolonged outage of dependent services

One of your external dependencies may be down, for a long enough time that all your retries fail and you are forced to return an error message back to the user. How can you ensure your system is not left in a corrupt state due to such a failure? How can you ensure that everything will work just fine when the user retries and the dependency is up? What if you added some state to a database, then called another external service which was down. Can the interim state that you added to the database cause future retries to fail?

More importantly, how can you test for such situations? These situations are very hard to reproduce and happen rarely but can cause your service to be unreliable if you don't design and code for these edge cases.

## Crash failure

One of your controllers executing a user request can stop executing midway due to the machine crashing. If it was supposed to call two external services, it may have called one but failed before calling the second one. The state maintained in external services is thus partially updated but not fully updated.

How can you ensure your system behaves correctly in the presence of crash failures at arbitrary points in their execution? And more importantly, how can you test for such situations?

# How can Coyote help?

While Coyote only tests the concurrency inside a single process and it cannot control coordinate and across the concurrency across a set of processes and machines, counterintuitively, it is a _surprisingly_ effective tool to test the concurrency and correctness of distributed systems.

The trick is to design your service in a way that you can run it all inside a single process. Running a part of (or an entire distributed system) inside a single process for testing may seem like a pre-posterous idea but it's not too difficult to pull off and a very effective testing tool when you get the hang of it.

We'll discuss a few design and testing patterns in the next set of articles which will show you exactly how you can do that. Some of the testing patterns are enabled by Coyote and cannot be tested without Coyote, while others are made better with Coyote.

Here's a brief peek into how certain design and testing patterns, aided by Coyote, can help you address the challenges outlined above.

## Simulate your external dependencies to run them in-process

You should simulate your external dependencies, like Azure Cosmos DB, or Service Bus with in-memory simulators which model the semantics of these services. This is easier to do than might first appear and you can look at a few examples [here](https://github.com) and [here](https://github.com).

## Run different services in the same process

Your service may consist of multiple services, say a front-end server which does minimal processing and forwards request to back-end machines. You can structure your code so that you can instantiate the controllers and handlers of different services in the same process and invoke the relevant methods directly instead of going over the wire. A lot of the complexity in distributed systems comes due to networking though, so instead of directly invoking the methods you should have the calls mediated the a mock network layer which can simulate that complexity.

In addition to running different services in the same process, you can use the same technique to run multiple _instances_ of the same service in the same process. We'll talk through a few examples of this technique in the coming articles.

## Mock out network calls

Instead of making direct calls which go over the wire, you can mediate all calls to external services through a _service interface_, which makes the network calls to actual services in production but which makes calls to in-memory simulators in testing mode.

Consider the following interface.

```csharp

public interface IStorageServiceInterface
{
  public Task CreateStorageContainer(string containerName);

  public Task DeleteStorageContainer(string containerName);
}

```

The production version of the interface calls out to the actual storage service.

```csharp

public class StorageServiceInterface : IStorageServiceInterface
{
  private StorageClient storageClient;

  public Task CreateStorageContainer(string containerName, ContainerProperties properties)
  {
    return storageClient.CreateStorageContainer(containerName, properties);
  }

  public Task DeleteStorageContainer(string containerName)
  {
    return storageClient.DeleteStorageContainer(containerName);
  }
}

```

The test version of the this interface looks like the following.

```csharp

public class TestStorageServiceInterface : IStorageServiceInterface
{
  private SimulatedStorageSystem simulatedStorageSystem;

  public Task CreateStorageContainer(string containerName, ContainerProperties properties)
  {
    // clone parameters as bytes are serialized when sending over the wire
    // and deserialized on the receiving end (and we certainly don't want
    // a change in the properties object by the storage service code being
    // reflected in-memory on the client side)
    var clonedProperties = Json.Deserialize(Json.Serialize(properties));

    return Task.Run(() =>
    {
      simulatedStorageSystem.CreateStorageContainer(containerName, properties);
    });
  }

  public Task DeleteStorageContainer(string containerName)
  {
    return Task.Run(() =>
    {
      simulatedStorageSystem.DeleteStorageContainer(containerName);
    });
  }
}

```

The `TestStorageServiceInterface` calls the methods in the in-memory simulator wrapped in a `Task` which it spins up for making the call. Since Coyote systematically explores scheduling and interleaving of the tasks, spinning up a new task here makes Coyote indirectly explore the various orders in which concurrent networks calls are procesed.

It is fairly simple to simualte network timeouts, service outages and retries using the above design pattern and test our code against the challenges we talked about in the earlier part of this article.

We'll talk in detail about how to use the above testing and design patterns to test a representative 'image gallery' service in the next set of articles.

