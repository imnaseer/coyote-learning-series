

We explored how to write our first concurrency test and reliably reproduce a bug in the `CreateUser` implementation using Coyote. Let's continue writing more tests for our `UserAccountManager` implementation.

Our first test set up a race between two concurrent `CreateUser` calls. A logical next test can set up a race between two concurrent `DeleteUser` calls. Let's write one.

```csharp
[Test]
public void TestConcurrentUserAccountDeletions()
{
  var dbCollection = new InMemoryDbCollection();
  var userAccountManager = new UserAccountManager(dbCollection);

  var userName = "joe";
  var userPayload = "...";

  // create the user, without any concurrent call
  await userAccountManager.CreateUser(userName, userPayload);

  // call DeleteUser twice, but do not await thus making both of them run concurrently
  var task1 = userAccountManager.DeleteUser(userName);
  var task2 = userAccountManager.DeleteUser(userName);

  await Task.WhenAll(task1, task2);

  var result1 = task1.Result;
  var result2 = task2.Result;

  // assert that one of the calls succeeds and the other one fails
  Assert.IsTrue(result1 ^ result2);
}
```

When you run this test, you'll find that it fails due to a very similar reason which made the first test failed.

```csharp
  // return true if user is deleted, false otherwise
  public async Task<bool> DeleteUser(string userName)
  {
     if (!await userCollection.DoesRowExist(userName))
     {
       return false;
     }

     return userCollection.DeleteRow(userName);
  }
```

Both the tasks conclude the user exists (before either of them gets a chance to delete the row), then both of them proceed to delete the row with one succeeding and the other failing.

Here's the fixed method.

```csharp
  // return true if user is deleted, false otherwise
  public async Task<bool> DeleteUser(string userName)
  {
     try
     {
       await userCollection.DeleteRow(userName);
       return true;
     }
     catch (RowNotFound)
     {
       return false;
     }
  }
```

The implementation above does not use the `DoesRowExist` method at all as we can never be guaranteed that a concurrent call didn't already delete the row from when we called `DoesRowExist` and when we call the `DeleteRow` method. There would be no such races _most_ of the times but there clearly can be such races _some_ of the times which is exactly what makes such bugs notoriously hard to catch and debug.

The fixed method calls directly calls the `DeleteRow` method of the database and relies on the database returning an error response which manifests in the `RowNotFound` exception to deal with such races. It effectively offloads the problem of preventing such races to the database collection itself as doing it in the application logic clearly didn't work.

You can find the updated `UserAccountManager` following the above pattern [here](https://github.com).

So far we wrote two concurrent tests, one racing two `CreateUser` calls and the other one racing two `DeleteUser` calls. Can you think of other interesting test cases? Any set of methods which may read and write shared state (and thus potentially interfere) are a great candidate for writing concurrent tests.

What about a race between a `CreateUser` and `DeleteUser` call?

At first glance, this may seem like a weird test write as we can't control the order in wihch the two methods will race with each other which makes it difficult to write meaningful assertions at the end. But upon some deeper thinking, you'll realize that there are only a few distinct cases of such a race and we can assert for each such outcome. Tests like these are useful for ensuring that upon such a race, the user is either fully created or fully delete and is not left in a half-broken state. This particualr example is simple but you can easily imagine more complex examples where such a race can in fact lead the object to be in a half-broken state, partially created and partially deleted in the presence of such a race.

Let's write this test.

```csharp

[Test]
public void TestConcurrentUserAccountCreateAndDelete()
{
  var dbCollection = new InMemoryDbCollection();
  var userAccountManager = new UserAccountManager(dbCollection);

  var userName = "joe";
  var userPayload = "...";

  // call CreateUser and DeleteUser concurrently, but do not await thus making both of them run concurrently
  var createTask = userAccountManager.CreateUser(userName, userPayload);
  var deleteTask = userAccountManager.DeleteUser(userName);

  await Task.WhenAll(createTask, deleteTask);

  var createResult = createTask.Result;
  var deleteResult = deleteTask.Result;

  // the create call will always succeed, no matter what.
  // the delete  may or may not succeed depending on if it finds
  // the user already created or not created
  Assert.IsTrue(createdResult);

  if (!deleteResult)
  {
     // the delete method didn't find the user which was created after
     var fetchedUserPayload = await userAccountManager.GetUser(userName);
     Assert.IsTrue(fetchedUserPayload, userPayload);
  }
  else
  {
    // if create and delete both returned true, then the user _must_
    // be deleted the user didn't exist before create was called
    // and since delete also returned true, it must not exist anymore

    var fetchedUserPayload = await userAccountManager.GetUser(userName);
    Assert.IsTrue(fetchedUserPayload == null);
  }
}

```

Let's run our test with Coyote controlled concurrency and see if it passes.

```
... Task 0 is using 'random' strategy (seed:2992598643).
... Telemetry is enabled, see http://aka.ms/coyote-telemetry.
..... Iteration #1
..... Iteration #2
...
..... Iteration #100
```

The test passed without any bugs which gives us confidence in the correctness of the code.

Often times, even if writing a Coyote doesn't find bugs, it makes you deeply think about possible outcomes in the presence of races and reason about the correct behavior. This clarity in thinking alone is extremely valuable. Developers often reason about edge cass when writing sequential tests but sometimes miss bugs in the presence of concurrency as they don't write focused concurrency tests as they are impractical to write without tools like Coyote.

This was a simple example but you can imagine how it's possible for the system to get into an inconsistent state if the `CreateUser` and `DeleteUser` calls involved multiple steps, such as modifying a couple of collections instead of just one, emitting events and create entries in other non-database systems, say, provisioning a dedicated Azure Storage container for the user. You get the idea.

You can find the complete source code for this example over at this [repo](https://github.com) and play around more with the code and the tests.