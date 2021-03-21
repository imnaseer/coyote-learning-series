
Racing with Race Conditions: Building Robust ASP.NET Services

Modern ASP.NET services are often deployed across a fleet of machines, storing data in backend storage systems and coordinating with other internal services that are similarly deployed across multiple machines. This architecture has become a lot more widespread due to the popularity of container-orchestration systems like Kubernetes and leads to services which are horizontally scalable. Horizontal scalability increases the concurrency in our system and too much concurrency can induce race conditions, leading to bugs that can be hard to find and debug.

This blog shares the engineering and test practices adopted by our team to tackle concurency challenges head on as, taking the help of Microsoft Research's Coyote tool. Coyote enables something quite radical: "unit testing" your concurrency. Unit testing and concurrency typically aren't mentioned in the same sentence, so let's unpack this a bit further.

Unit tests exercise a focused set of functionality in our code bases, invoking a set of methods, usually sequentially, and asserting for correct responses at the end. Concurrency is discouraged in unit tests as it increases their flakiness leading to tests that typically pass but can sometimes fail. Such failures are generally very hard to reproduce as they may depend on sensitive timing conditions which is why developers don't mix concurrency with unit tests. They instead write stress tests to push the systems to its limit in order to expose concurrency bugs before the code hits production. Stress tests are never part of the dev inner loop however as they are too costly to run as part of regular feature development. They are usually not run as part of the regular build pipelines either but instead run before shipping the code to production, if that. Debugging failures found through stress tests can also be quite a challenge as there's a ton of activity and noise in the system, by design.

Coyote unlocks the ability to reliably find subtle concurrency bugs - the kind that only stress testing _sometimes_ finds - through simple, focused unit tests which developers can run as part of their inner loop (edit/test/debug cycle) during feature development.

The above might sound too good to be true. It is however both _too good_ and very much _true_.

Before we delve into the details of our testing strategy, it will help to understand the basics of Coyote through a simple, intentionally contrived, example. Consider the following code.

```csharp

public static Task Worker(int index, ConcurrentQueue<int> queue)
{
    return Task.Run(() =>
    {
        queue.Enqueue(index);
    });
}

public static async Task TestInterleavings()
{
    var queue = new ConcurrentQueue<int>();

    var tasks = new List<Task>();
    for (int i = 0; i < 3; i++)
    {
        tasks.Add(Worker(i, queue));
        await Task.Delay(1);
    }

    await Task.WhenAll(tasks);

    Assert.False(queue.ToList().SequenceEqual(new List<int>() { 1, 0, 2 }));
}

```

`TestInterleavings` spawns a few worker tasks each of which enqueues a number in a concurrent queue. It waits a small amount of time between each invocation to simulate natural delays in a more complex program. It then asserts that the sequence of numbers inserted in the queue should be anything but "1, 0, 2", that corresponds to an interlaving leading to a hypothetical bug.

If we invoke `TestInterleavings` multiple times, the probability of it hitting the interleavings which leads to "1, 0, 2" is fairly low given how the above program is written. Though technically we know such an interleaing _could_ happen. Coyote can help by taking control of the concurrency in the above program and exploring different interleavings, those likely to happen as well as those which are unlikely to happen but still possible. It does so by rewriting the assembly to use a variant of .NET tasks which allows it to precisely control when a particular task is scheduled to run. This allows it to _explore_ possible interleavings using a number of _scheduling_ algorithms. When it finds a bug, it can give developers the ability to reproduce the precise set of interleavings which lead to a bug as it can replay the same set of decisions again and again.

To find the bug above using Coyote, you should mark the test with an attribute so Coyote tooling knows about it. You can invoke the Coyote exploration engine programmatically as well but we'll keep things simple in this blog.

```csharp
[Microsoft.Coyote.SystematicTesting.Test]
public static async Task TestInterleavings()
{
  ...
}
```

Next you rewrite your assembly so Coyote can control the concurency during test and invoke the coyote tool to test it.

```
c:\coyote-samples\bin\net5.0>coyote rewrite .\TestInterleavings.dll

Microsoft (R) Coyote version 1.2.3.0 for .NET Core 5.0.4
Copyright (C) Microsoft Corporation. All rights reserved.

. Rewriting C:\Users\imnaseer\source\repos\coyote-samples\bin\net5.0\TestInterleavings.dll
... Rewriting the 'AccountManager.dll' assembly (AccountManager, Version=1.2.3.0, Culture=neutral, PublicKeyToken=null)
..... Skipping assembly with reason: already rewritten by Coyote v1.2.3.0
. Done rewriting in 0.7072057 sec

coyote test .\TestInterleavings.dll -m TestInterleavings -i 1000
...
```

Coyote clearly appears to be useful. The above was a small and intentionally contrived example though. How can we use Coyote to find concurrency bugs in our ASP.NET services? The kind which are deployed across a fleet of machines interacting with other services, instead of a couple of concurrent methods calling each other in a single process? While in-process concurrency and distributed service concurrency can appear to be very different beasts, both of them can be tested fairly effectively with Coyote if you follow a simple set of testing and design principles.

Before describing the details of our testing strategy, it will be helpful to look at the architecture of a simple ASP.NET service, which includes a number of controllers running across a fleet of machines, reading and writing data from a backend database, say Azure Cosmos DB.

<simplified diagram showing data plane service reading/writing to cosmos db>

Let's take the example of a simple controller, say an AccountManager which exposes a REST API to create, update and delete accounts by reading and writing to a backened database, say something like Azure Cosmos DB. We want to ensure that the our service continues to work properly despite race conditions which can lead to controllers on different machines trying to read and write the same set of rows at around the same time. This can happen due to a client of our service retrying the same request in a short span of time if he doesn't hear back on the first request, say due to an intermittent network glitch.

<diagram showing two requests going to two different controllers, both talking to the database>

We can simulate the above situation in a unit test setting by having the clients invoke the controller methods directly, and the controllers calling out to the mock of the database instead of the actual database.

```csharp

public interface IServiceClient
{
  public Task<Account> CreateAccount(Account account);
}

public class AccountManagerController : BaseController
{
  private readonly string CollectionName = "";

  private IDatabaseClient databaseClient;

  public AccountManagerController(IDatabaseClient databaseClient)
  {
    this.databaseClient = databaseClient;
  }

  [HttpPut]
  public async Task<ActionResult<Account>> PutAsync(string accountName, Account account)
  {
    if (await databaseClient.GetRow(CollectionName, accountName))
    {
      return this.NotFound();
    }

    var accountRow = await databaseClient.CreateRow(CollectionName, accountName, account);

    return this.Ok(account);
  }
}

public interface IDatabaseClient
{
  public Task<Row> CreateRow<T>(string collectionName, string key, T row);

  public Task<Row> GetRow<T>(string collectionName, string key);

  public Task<Row> ReplaceRow<T>(string collectionName, string key, T row);

  public Task DeleteRow<T>(string collectionName, string key);
}

```

We see the implementation of a simple controller which checks whether an account exists using a database client, and creates one if it doesn't. There is a simple concurrency bug in the above implementation where two concurrent calls to `PutAsync` can both conclude the row doesn't exist, both can try to create the row with only one succeeding and the other resulting in an uncaught exception which manifests as an internal server error in production. This bug can be caught by Coyote using the following test.

```csharp

public async Test AccountTest()
{
   var testDatabaseClient = new TestDatabaseClient();
   var testServiceClient = new TestServiceClient(testDatabaseClient);

   var account = new Account()
   {
     Name = "joe",
     Details "..."
   };

   var task1 = testServiceClient.CreateAccount(account.Name, account);
   var task2 = testServiceClient.CreateAccount(account.Name, account);

   await Task.WhenAll(task1, task2);
}

```

Here's the implementation of `TestServiceClient` and `ProductionServiceClient`.

```csharp

public class ProductionServiceClient : IServiceClient
{
  private string hostName;
  private HttpClient httpClient = new HttpClient();

  public ProductionServiceClient(string hostName)
  {
    this.hostName = hostName;
  }

  public async Task<Account> CreateAccount(Account account)
  {
    var response = await httpClient.PutAsJsonAsync(new Uri(hostName, $"/accounts/{account.Name}"), account);
    if (response.StatusCode == 200)
    {
      var content = await response.Content.ReadAsStringAsync();
      return JsonConvert.Deserialize<Account>(content);
    }
    else
    {
      throw new ServiceClientException(reponse.StatusCode);
    }
  }
}

public class TestServiceClient : IServiceClient
{
  private IDatabaseClient databaseClient;

  public TestServiceClient(IDatabaseClient databaseClient)
  {
    this.databaseClient = databaseClient;
  }

  public async Task<Account> CreateAccount(Account account)
  {
    var accountCopy = JsonConvert.Deserialize<Account>(JsonConvert.Serialize(account));

    return Task.Run(() =>
    {
      var controller = new AccountManagerController(databaseClient);

      var response = (ObjectResult)(await controller.PutAsync(accountCopy.Name, accountCopy));

      if (response is OkObjectResult)
      {
        return (Account)((OkObjectResult)actionResult).Value;
      }
      else
      {
        throw new ServiceClientException(reponse.StatusCode);
      }
    });
  }
}

```

It's very instructive to compare `TestServiceClient` with `ProductionServiceClient`. The latter makes an actual HTTP call to our service while the former just invokes the controller method directly but after a round of serializing/deserializing the input and wrapping the method invocation inside a new task. `TestServiceClient` is effectively following a very simple pattern we can use in Coyote testing to "mock out the network".

Network messages have the following two important properties that we are replicating in our setup:

1. The bits are serialized over the wire and deserialized on the receiving end
2. Network messages can be delivered out-of-order; if a network message A starts out before message B, B can be received by a machine before A

The second property is simulated by performing the operatoin in a newly spawned task. Just like two network messages may be delivered out of order, two spawned tasks can also execute out of order. Just because a task is created before another task is not a guarantee that it will start (or finish) its execution before the one spawned later. This sort of _equivalence_ between tasks and network messages allows us to leverage Coyote to simulate network message delivery semantics. It's a simple but very powerful trick.

You'll also notice that `TestServiceClient` creates a new instance of a controller each time, passing it an `IDatabaseClient` which it uses to read and write to the database. As long as the controller instances don't have any shared in-memory state, and don't use concurrency control mechanisms like locks (which don't work when the code is physically running on different processes), we can even get away with just having a single instance of our controller. Simulating controllers running on multiple machines just requires us to ensure our instances don't accidentally use shared in-memory state which doesn't work across processes.

Let's now look at the database client implemnetations, focusing just on the `CreateRow` method.

```csharp

public interface IDatabaseClient
{
  public Task<Row> CreateRow<T>(string collectionName, string key, T row);
}

public class ProductionDatabaseClient : IDatabaseClient
{
  private CosmosDbClient dbClient = new CosmosDbClient();

  public Task<Row> CreateRow<T>(string collectionName, string key, T row)
  {
    return dbClient.CreateRow<T>(collectionName, key, row);
  }
}

public class TestDatabaseClient : IDatabaseClient
{
  private string expectedCollectionName;
  private ConcurrentDictionary<srting, Row> collection = new ConcurrentDictionary<srting, Row>();

  public TestDatabaseClient(string collectionName)
  {
    this.expectedCollectionName = collectionName;
  }

  public Task<Row> CreateRow<T>(string collectionName, string key, T row)
  {
    if (collectionName != expectedCollectionName)
    {
        // throw the same exception as thrown by the ProductionDatabaseClient
       throw new DatabaseCollectionDoesNotExist(collectionName);
    }

    var rowCopy = JsonConvert.Deserialize<T>(JsonConvert.Serialize(row));

    return Task.Run(() =>
    {
      bool success = collection.TryAdd(key, rowCopy);

      if (!success)
      {
        // throw the same exception as thrown by the ProductionDatabaseClient
        throw new DatabaseRowAlreadyExistsException(key);
      }

      return rowCopy;
  }
}

```

`TestDatabaseClient` follows a similar pattern to mock out network calls where it serializes/deserializes its input and spawns a task to simulate sending a network message. Unlike `TestServiceClient` which called out controller, `TestDatabaseClient` simulates the database semantics by writing the row to an in-memory collection. Since the two controllers end up calling the same instance of `TestDatabaseClient`, this in-memory collection serves as the "persistent" store for our mock database. Sometimes it's not possible to share the same instance between controllers in which case we can simulate such _persistent_ state using `AsyncLocal` variables.

Mocks for concurrent unit tests are similar to the ones used in traditional unit tests in that we only need to simulate/mock the functionality we need and can adopt as a "pay-as-you-go" model where we can incrementally make them more complete as our testing needs evolve. Furthermore, it's perfectly fine to use lock statements and other in-process concurrency control constructs if it makes our mocking simpler. As an example, this Coyote tutorial shows how you can easily simulate CosmosDB ETag semantics in the database mock with the help of lock statements. Production Cosmos DB code of course has to implemenet ETags on a distributed cluster of machines, while taking arbitrary failures into account. Mocking the same functionality in a single process for use in concurrency unit tests is a lot more simpler however.

The following diagram depicts how we were able our concurrency unit test is able to explore the concurrency in our distributed system by replacing all external dependencies and clients with in-memory equivalents, communicating with each other through clients which mock out the network calls.

<diagram>

Our team was able to leverage these simple testing and design patterns to great effect.
