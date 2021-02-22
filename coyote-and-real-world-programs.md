
As explained in a number of other articles in this learning series, Coyote does not explore _all_ possible interleavings of concurrent programs but a subset of them. In particular, it only explores interleavings around "scheduling points", which include `Task.Run`, `Task.Delay`, `Task.Wait`, `Task.ExploreContextSwitch` and `lock` statements.

You might wonder whether the scheduling points which naturally arise due to the use of `Run`, `Delay`, `Wait` and `lock` statements are "good enough" or whether you should sprinkle your programs with additional `ExploreContextSwitch` statements for Coyote to find bugs.

This article will build your intuition on why the scheduling points which naturally occur in program are usually _good enough_ and how to determine when you should give additional hints to Coyote.

Concurrency by itself is not problematic if the various concurrrent tasks never read or write "shared state". Consider the following concurrent program.

```csharp

public async Task Greetings(string name)
{
  return Task.Run(() =>
  {
    Console.WriteLine($"Hello {name}");
    Console.WriteLine($"See you later, {name}");
  });
}

public async Task Test()
{
  var task1 = Greetings("Foo");
  var task2 = Greetings("Bar");

  await Task.WhenAll(task1, task2);
}

```

The two tasks above read and write _zero_ shared state so the interleavings between them do not matter at all. They do not impact each other's behavior in any material way. Concurrency only becomes interesting (and problematic) when the tasks read and/or write shared state.

# Shared state in Services 

In typical services, the most interesting shared state is maintained in _external_ storage systems, such as databases. Consider the following program which concurrently reads and writes rows from a database.

```csharp

private IDbCollection dbCollection;

public async Task CreateUser(string userName, string details)
{
  if (await dbCollection.DoesRowExist(userName))
  {
    return;
  }

  await dbCollection.CreateRow(userName, new User() { Details = details });
}

public async Task Test()
{
  var task1 = CreateUser("joe", "...");
  var task2 = CreateUser("joe", "...");

  await Task.WhenAll(task1, task2);
}

```

The `CreateUser` method in the above program checks whether a user with a given name exists in the database and creates one if it doesn't already exist. It contains a concurrency bug which gets triggered when two concurrent calls for the same user run at the same time, both determine the user doesn't exist (as neither has created the user so far), and both proceed to create the user. One of the requests succeeds while the other fails with an exception.

There are a number of very interesting points to note about the example above.

Firstly, the two concurrent calls read and write _shared state_ (in an external db in this instance) which is why it is interesting to explore the interleavings between them. It's also interesting to note that other than the database, they don't read/write and "local" shared state.

Secondly, there are no visible sources of asynchrony in the program above. While they contain a bunch of `async`/`await` statements, we can't see any `Run`, `Yield` or `Delay` statements which actually introduce asynchrony in a program. Most SDKs for talking to external systems like databases make network calls over the wire and spawn off new tasks when they do.

```csharp

public class ProductionDbCollection
{
  public async Task CreateRow(string key, object value)
  {
     return Task.Run(() =>
     {
        // make network REST call to database to write the key/value pair
     });
  }
}

```

Even though our program doesn't introduce any explicit asynchrony, the SDKs we use often end up introducing asynchrony so we can write performant highly concurrent programs. When writing simualtors and mock of those external dependencies in Coyote, we introduce asynchrony as well.

```csharp

public class TestDbCollection
{
  public async Task CreateRow(string key, object value)
  {
     return Task.Run(() =>
     {
        // simulate the behavior of database by writing to an in-mem dictionary
     });
  }
}

```

If you architect your programs and services to minimize or eliminate _local_ shared state (which is a good programming practice) then typically the only remaining shared state is the state maintained in external systems. And since most programs just have a bunch of async/await statements but typically don't introduce any explicit asynchrony themselves (other than the one introduced by the SDKs they consume), introducing the synchrony in the mocks/simulators of those external systems is usually all that is needed for Coyote to effectively find bugs caused by concurrent reads and writes to that shared external state.

The above example illustrates why introducing `Task.Run` in the mocks/simulators to external stateful systems is typically all that's needed for Coyote to find a number of interesting bugs. It also points out an interesting technique to maximize state interference when writing concurrency tests. The test above triggered two `CreateUser` calls for the same user ("joe" in this case). If it had instead triggered two tasks, one for "joe" and another for creating "sally", the two tasks would not have read/written any shared state and our concurrency test wouldn't have been as effective in finding bugs.

# Shared state in local programs

Sometimes it is not possible to eliminate all shared local state and a program may contain multiple tasks reading and writing to shared local state.

Consider the following buggy implementation of a thumbnail manager which maintains a in-memory cache of thumbnails for images. It should reject storing a thumbnail if its last-modified-time is older than the existing entry.

```csharp

public class ThumbnailInfo
{
  public DateTime LastModifiedTime { get; set; }

  public byte[] Thumbnail { get; set; }
}

// ...

public class ThumbnailManager
{
  private ConcurrentDictionary<string, ThumbnailInfo> thumbnailCache;

  public void CreateOrUpdateThumbnailInCache(string filename, byte[] thumbnail, DateTime lastModifiedTime)
  {
    var shouldAdd = !thumbnailCache.ContainsKey(filename);
    var shouldUpdate = !shouldAdd && thumbnailCache[filename].LastModifiedTime < lastModifiedTime;

    if (!shouldAdd && !shouldUpdate)
    {
      return;
    }

    if (shouldAdd)
    {
      thumbnailCache.TryAdd(filename, thumbnail);
    }
    else
    {
      thumbnailCache.TryUpdate(filename, thumbnail);
    }
  }
}

public async Task Test()
{
  var thumbnailManager = new ThumbnailManager();

  var lastModifiedTime = DateTime.Now;

  var thumbnail1 = new byte[] { 0, 1 };
  var thumbnail2 = new byte[] { 2, 3 };

  var task1 = Task.Run(() => CreateOrUpdateThumbnailInCache("test.jpg", thumbnail1, lastModifiedTime.AddMinutes(-10));
  var task2 = Task.Run(() => CreateOrUpdateThumbnailInCache("test.jpg", thumbnail2, lastModifiedTime));

  await Task.WhenAll(task1, task2);

  // Assert that we always get thumbnail2 which was modified at a later timestamp
  Assert.IsTrue(Enumerable.SequenceEquals(thumbnailManager.GetThumbnail("test.jpg"), thumbnail2));
}

```

The above program has a bug where a race between the two tasks can result in thumbnail1 being stored instead of the later thumbnail2. This can happen when both calls start out, both find no entry in the cache, thumbnail2 is written and then thumbnail1 is written overwriting thumbnail1.

When you run the above program with Coyote however, it will never find this bug. This is because `CreateOrUpdateThumbnailInCache` doesn't contain any scheduling point and Coyote will always execute it _atomically_ and thus never find this bug.

In order to help Coyote find this bug, we'll have to inject `Task.ExploreContextSwitch` statements in the right places in `CreateOrUpdateThumbnailInCache`.

```csharp

  if (!shouldAdd && !shouldUpdate)
  {
    return;
  }

+ Microsoft.Coyote.Tasks.Task.ExploreContextSwitch();

  if (shouldAdd)
  {
    thumbnailCache.TryAdd(filename, thumbnail);
  }
  else
  {
    thumbnailCache.TryUpdate(filename, thumbnail);
  }
```

With the introduction of the above statement, Coyote will explore context switches around the time when the check is done and when the dictionary is updated and it will find the bug.

The bug fix will be to include all the statements in `CreateOrUpdateThumbnailInCache` in a `lock` statement. Coyote does explore context switches around beginning and ends of `lock` statements and since the statements inside a `lock` block are always executed atomically, we'll also be able to remove the `ExploreContextSwitch` statement in the method after the bug fix.

# Lessons

The above discussion showed how concurrency bugs nearly always arise when different tasks are reading from and writing to _shared_ state. In the cases of external state, introducing asynchrony through `Task.Run` statements in mocks/simulators to external services is all we need to do. In case of local state, we might have to inject synthetic scheduling points to aid Coyote if the reads/writes to shared state aren't naturally surrounded by scheduling points around which explores Coyote explores various interleavings.

