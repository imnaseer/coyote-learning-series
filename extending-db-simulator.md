

Coyote tests often involve simple simulators which simulate a subset of behavior of an external service instead of hard coding outputs to return on given inputs like typical mocks do. Simulators can be as simple or complex as we want and we can often write them in a "pay-as-you-go" fashino

We'll extend the database simulator we began implementing in the previous article to support ETags for optimistic concurrency control. While the actual implementation in CosmosDb can be quite complex, simualating the semantics in an in-memory simulator is fairly trivial.

Before implementing the ETag support, let's motivate the problem by adding a `version` property to the created users and only updating the usesrs if the version is greater than the existing version.

```csharp

public UserAccountManager
{
   private IDbCollection dbCollection;

   public async Task<bool> CreateUser(string username, string details, int version)
   {
      var userPayload = new User() { Details = details, Version = version };

      try
      {
        await dbCollection.CreateRow(username, Serialize(userPayload));
      }
      catch (RowAlreadyExists)
      {
        return false;
      }

      return true;
   }

   public async Task<bool> UpdateUser(string username, string details, int version)
   {
      string existingUserPayload;

      try
      {
        existingUserPayload = await dbCollection.ReadRow(username);
      }
      catch (RowNotFound)
      {
        return false;
      }

      var existingUser = Deserialize(existingUserPayload);
      if (version <= existingUser.Version)
      {
        return false;
      }

      var updatedUser = new User() { Details = details, version = version };

      try
      {
        await dbCollection.ReplaceRow(username);
      }
      catch (RowNotFound)
      {
        return false;
      }

      return true;
   }
}

```

That was a lot of code.

Let's first write a "sequential" test to check the above property.

```csharp

public async Task Test()
{
   var dbCollection = new MockDbCollection();
   var accountManager = new UserAccountManager(dbCollection);

   var result = await accountManager.CreateUser("joe", "...", 1);
   Assert.IsTrue(result);

   result = await accountManager.UpdateUser("joe", "secondVersion", 2);
   Assert.IsTrue(result);

   result = await accountManager.UpdateUser("joe", "secondVersionAlternate", 2);
   Assert.IsFalse(result);
}

```

The above test passes. Let's write a test where we update the user concurrently.

```csharp

public async Task Test()
{
   var dbCollection = new MockDbCollection();
   var accountManager = new UserAccountManager(dbCollection);

   var result = await accountManager.CreateUser("joe", "...", 1);
   Assert.IsTrue(result);

   var updateTask1 = accountManager.UpdateUser("joe", "secondVersion", 2);
   var updateTask2 = accountManager.UpdateUser("joe", "secondVersionAlternate", 2);
   await Task.WhenAll(updateTask1, updateTask2);

   // Assert that only one of the updates above succeed and not both
   Assert.IsTrue(updateTask1.Result ^ updateTask2.Result);
}

```

When you run the test above, you'll realize it will fail in one of the iterations as Coyote will find an interleaving in which both calls succeed. That race condition happens when both the calls read the first version of the row, both independently think their version is greater than what is currently in the database and update the entry.

The problem in fact is worse than that. Consider the following snippet.

```csharp

var updateTask1 = accountManager.UpdateUser("joe", "secondVersion", 2);
var updateTask2 = accountManager.UpdateUser("joe", "thirdVresion", 3);
await Task.WhenAll(updateTask1, updateTask2);

var user =  await accountManager.GetUser("joe");
Assert.IsTrue(user.Version == 3);

```

The above test will also fail in some iterations with version 2 overwriting version 3! So we find out that this not just a benign failure but our code doesn't respect the semantics at all in the presence of concurrency.

Cosmos DB provides ETags which we can use and only update the row if the ETags match. This ensures that we fail if another writer updates the row after we have read it, thus indicating that we made our decision operating on stale data.

Let's take a look at the correct implementation of `UpdateUser`.

```csharp

public async Task<bool> UpdateUser(string username, string details, int version)
{
   string existingUserPayload;

   ...

   var existingUser = Deserialize(existingUserPayload);
   if (version <= existingUser.Version)
   {
     return false;
   }

   var updatedUser = new User() { Details = details, version = version };

   try
   {
     // This call will fail if the ETags mismatch
     await dbCollection.ReplaceRow(username, existingUser.ETag);
   }
   catch (Exception ex) when (ex is RowNotFound || ex is ETagMismatch)
   {
     return false;
   }

   return true;
}

```

The above requires us to implement the ETag functionality in our simulator.

```csharp

public class Payload
{
  string Value;
  string ETag;
}

public class MockDbCollection : IDbCollection
{
   private ConcurrentDictionary<string, Payload> collection;

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
        var payload = new Payload() { Value = value, ETag = Guid.NewGuid().ToString() }
        var success = collection.TryAdd(key, payload);
        if (!success)
        {
          throw new RowAlreadyExists();
        }

        return true;
     });
   }

   public Task<bool> UpdateRow(string key, string value, string etag)
   {
     return Task.Run(() =>
     {
        lock (collection)
        {
          var getSuccess = collection.TryGet(key, out Payload existingPayload);
          if (getSuccess && etag != existingPayload.ETag)
          {
             throw new ETagMismatchException();
          }

          var success = collection.TryAdd(key, payload);
          if (!success)
          {
            throw new RowAlreadyExists();
          }

          return true;
        }
     });
   }
}

```

We take a `lock` during `UpateRow` to ensure no other task races with us while we checking the ETag for mismatch. We don't need to take a `lock` during operations which don't check the ETag as we're using a thread-safe concurrency dictionary.

If we run our test again, we'll see it passes! If we remove the ETag check, it will fail as expected.

You can find the complete implementation of the simulator using ETags over [here](https://github.com).

One interesting observation above is that we took a lock in our simulator when simulating the ETag functionality but we didn't (and couldn't) take a lock in the `UserAccountManager`. This is because `UserAccountManager` can run across different processes and different machines and locks clearly don't work in that setting. We run the concurrency test in one process however so its perfectly fine for our _simulator_ to take a lock to simplify the simulation of the ETag functionality.

As you can see above, it didn't take a lot of effort to simulate ETags in our simulator, as we are just simulating the semantics in-memory as opposed to implementing the _actual_ code which must function correctly when run in a distributed system context and has to worry about arbitrary faults and failures.
