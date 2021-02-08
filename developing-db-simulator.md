
Mocks are quite important when writing traditional unit tests and they play an even more important role when writing Coyote based concurrency tests. Since Coyote explores different interleavings on each iteration, we have to write "intelligent" mocks (perhaps better called 'simulators') which simulate the behavior of the external dependencies so they return the correct responses no matter which interleaving is explored by Coyote.

Consider a typical mock written during unit tests.

```csharp

var mockCollection = new Mock<IDbCollection>();
mockCollection.Setup(c => c.DoesRowExist("joe")).Returns(true);
mockCollection.Setup(c => c.CreateUser("joe", ...)).Returns(true);

var userAccountManager = new UserAccountManager(mockCollection);

var result = await userAccountManager.CreateUser("joe", payload);
Assert.IsTrue(result);

```

While the above test works perfectly for sequential tests, it doesn't work when wr write concurrent tests. Consider the following concurrent test.

```csharp

var mockCollection = new Mock<IDbCollection>();
mockCollection.Setup(c => c.DoesRowExist("joe")).Returns(false);
mockCollection.Setup(c => c.CreateUser("joe", ...)).Returns(true);

var userAccountManager = new UserAccountManager(mockCollection);

var task1 = userAccountManager.CreateUser("joe", payload);
var task2 = userAccountManager.CreateUser("joe", payload);

Assert.IsTrue(task1.Result ^ task2.Result);

```

The above mock will fail as both the calls to `CreateUser` succeed as `DoesRowExist` returns false no matter what. `DoesRowExist` should only return false if the user doesn't exist. Let's fix it.

```csharp

bool userExists = false;
var mockCollection = new Mock<IDbCollection>();
mockCollection.Setup(c => c.DoesRowExist("joe")).Returns(userExists);
mockCollection
  .Setup(c => c.CreateUser("joe", ...))
  .Returns(() =>
  {
     if (userExists)
     {
       throw new RowAlreadyExists();
     }

     userExists = true;
     return true;
  });

```

The above mock is a bit more complicated but models the `DoesRowExist` behavior properly.

If you run the test above, while the assertion will pass, it will also pass for a buggy implementation of `CreateUser`.

Why is that?

While we invoke the two tasks at the same time, there is no _actual_ concurrency in the code path which means the two calls effectively execute sequentially.

In order to test the true asynchrony in the system, we'll have to write a mock which will spawn a new task whenever `DoesRowExist` or `CreateRow` methods are called.

```csharp

public class MockDbCollection : IDbCollection
{
   private bool doesUserExist = false;

   public Task<bool> DoesRowExist(string key)
   {
     return Task.Run(() =>
     {
       return doesUserExist;
     });
   }

   public Task<bool> CreateRow(string key, string value)
   {
     return Task.Run(() =>
     {
        if (doesUserExist)
        {
          throw new RowAlreadyExists();
        }

        doesUserExist = true;
        return true;
     });
   }
}

```

If you run the test using the above mocked out implementation of IDbCollection, you'll find that the test case catches the bug in the faulty implementation of `CreateUser`. The introduction of `Task.Run` in the above methods introduces true asynchrony in the system, similar to how the production implementation of IDbCollection would spawn new tasks when it makes network calls so as not to block the rest of the system.

Can we make the above mock a bit more generally applicable, so we don't have to write custom mocks for each test case? What if we turned it into a simple `simulator` which simulated the behavior of actual IDbCollection?

Most external dependencies tend to have very simple simulators and a little effort in writing the simulator can yield large productivity gains.

Let's write out an IDbCollection simulator which we can use in our tests.

```csharp

public class MockDbCollection : IDbCollection
{
   private ConcurrentDictionary<string, string> collection;

   public Task<bool> DoesRowExist(string key)
   {
     return Task.Run(() =>
     {
       return collection.ContainsKey(key);
     });
   }

   public Task<bool> CreateRow(string key, string value)
   {
     return Task.Run(() =>
     {
        var success = collection.TryAdd(key, value);
        if (!success)
        {
          throw new RowAlreadyExists();
        }

        return true;
     });
   }
}

```

Through a very simple change, we now have a simple simulator which can model the behavior of adding rows and checking for their existence which we can use in our concurrency tests. Simulators are often surprisingly easy to write and have the benefit that they can be reused in multiple tests. Simulators can also be written in a "pay-as-you-go" fashion where the initial implementation can be simple and more functionality can be added as we go along. We'll see an example of this later in this article.

Let's make our simulator complete by implementing the `GetRow` and `DeleteRow` methods.

```csharp

public Task<string> GetRow(string key)
{
  return Task.Run(() =>
  {
     var success = collection.TryAdd(key, out string value);
     if (!success)
     {
       throw new RowNotFound();
     }

     return value;
  });
}

public Task<bool> DeleteRow(string key)
{
  return Task.Run(() =>
  {
     var success = collection.TryRemove(key);
     if (!success)
     {
       throw new RowNotFound();
     }

     return true;
  });
}
```

With the above simulator, we will be able to write a bunch of concurrent test cases without having to write custom mocks for each of them. You can find the implementation over [here](https://github.com)

We'll explore how to add support for ETags for optimistic concurrency control in the next article. Adding support for ETags combined with Coyote's concurrency testing will allow us to test a scenario which is fairly hard to hit in production but can lead to data loss.
