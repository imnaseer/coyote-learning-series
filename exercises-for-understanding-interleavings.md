
[[_TOC_]]

# Example 1 (async/await)

```csharp

public async Task Greetings(string name)
{
  Console.WriteLine($"Hello {name}");
  Console.WriteLine($"See you later, {name}");
  return Task.Completed;
}

public async Task Test()
{
  var task1 = await Greetings("Foo");
  var task2 = await Greetings("Bar");
  
  await Task.WhenAll(task1, task2);
}

```

Which of the following outputs are possible interleavings explored by Coyote? Output 1, Output 2, both or none?

| Output 1                                                                           | Output 2                                                                           |
|------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| <code>Hello Foo<br/>See you later, Foo<br/>Hello Bar<br/>See you later, Bar</code> | <code>Hello Bar<br/>See you later, Bar<br/>Hello Foo<br/>See you later, Foo</code> |


<details>
<summary>Click to reveal the solution</summary>

Output 1 is the _only_ possible output in the program above, not just the only one explored through Coyote but in general as the program does not contain any asynchrony _at all_.  While it contains a series of `async` and `await` statements, the program doesn't spawn any Tasks, nor does it contain any `Task.Yield` statements. `async`/`await` statements allow you to write concise and readable code when dealing with asynchrony but don't introduce any asynchrony just by themselves.
</details>

# Example 2 (Task.Yield)

```csharp

public async Task Greetings(string name)
{
  Console.WriteLine($"Hello {name}");
  await Task.Yield();
  Console.WriteLine($"See you later, {name}");
}

public async Task Test()
{
  var task1 = Greetings("Foo");
  var task2 = Greetings("Bar");
  
  await Task.WhenAll(task1, task2);
}

```

Which of the following outputs are possible interleavings explored by Coyote?

| Output 1                                                                                | Output 2                                                                                | Output 3                                                                                |
|-----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| <code>Hello Foo<br/>Hello Bar<br/>See you later, Foo<br/>See you later, Bar<br/></code> | <code>Hello Bar<br/>See you later, Bar<br/>Hello Foo<br/>See you later, Foo<br/></code> | <code>Hello Foo<br/>Hello Bar<br/>See you later, Bar<br/>See you later, Foo<br/></code> |

<details>
<summary>Click to reveal the solution</summary>

Output 1 and 3 are both possible interleavings which Coyote can explore. Output 2 will never be explored by Coyote, and in fact can never happen even outside of Coyote as there is no `Task.Run`, `Task.Yield` or `Task.Delay` statement before the first `Hello Foo` line is printed. `Hello Foo` will _always_ be the first line which will printed, and after the first `Task.Yield` is hit, we can get all possible interleavings of the remaining statements.
</details>
 
# Exanple 3 (Task.Run)

```csharp

public Task Greetings(string name)
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

Which of the following outputs are possible interleavings explored by Coyote?

| Output 1                                                                                | Output 2                                                                                | Output 3                                                                                |
|-----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| <code>Hello Bar<br/>See you later, Bar<br/>Hello Foo<br/>See you later, Foo<br/></code> | <code>Hello Foo<br/>See you later, Foo<br/>Hello Bar<br/>See you later, Bar<br/></code> | <code>Hello Foo<br/>Hello Bar<br/>See you later, Bar<br/>See you later, Foo<br/></code> |

<details>
<summary>Click to reveal the solution</summary>

Output 1 and 2 are both possible interleavings. In fact they are the only two interleavings Coyote can explore, though there are more possible interleavings possible in production, such as Output 3. Output 3 will never be explored by Coyote as there is no scheduling point in the lambda passed to `Task.Run` and it always run as an atomic block by Coyote.

</details>

# Example 4 (Task.Delay)

```csharp

public async Task Greetings(string name, int delay)
{
  await Task.Delay(delay);
  Console.WriteLine($"Hello {name}");
  Console.WriteLine($"See you later, {name}");
}

public async Task Test()
{
  var task1 = Greetings("Foo", 100);
  var task2 = Greetings("Bar", 50);
  
  await Task.WhenAll(task1, task2);
}

```

Which of the following outputs are possible interleavings explored by Coyote?

| Output 1                                                                                | Output 2                                                                                | Output 3                                                                                |
|-----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| <code>Hello Bar<br/>See you later, Bar<br/>Hello Foo<br/>See you later, Foo<br/></code> | <code>Hello Foo<br/>See you later, Foo<br/>Hello Bar<br/>See you later, Bar<br/></code> | <code>Hello Foo<br/>Hello Bar<br/>See you later, Bar<br/>See you later, Foo<br/></code> |

<details>
<summary>Click to reveal the solution</summary>

Output 1 and 2 are both possible interleavings. In fact they are the only two interleavings Coyote can explore, though there are more possible interleavings possible in production, such as Output 3. Output 3 will never be explored by Coyote as there is no scheduling point after `Task.Delay` through to the end of the `Greetings` method. It's interesting to note that Coyote doesn't really respect the delay value passed to the `Delay` method - the behavior around `Delay` calls is the same as that with `Run` or `Yield` statements. The rationale is that .NET does not guarantee that a task waiting with a smaller delay interval will always resume execution before a task with a larger delay interval. It only guarantees that the task will wait _at least_ that amount but doesn't give any further guarantees therefore developers of concurrent programs shouldn't rely on such behavior and Coyote ensures that by not respecting the value passed to the `Delay` method when exploring the possible interleavings of concurrent calls wiht `Delay` statements.

</details>

# Example 5 (lock)

```csharp


public Task Greetings(string name, object syncObject)
{
  return Task.Run(() =>
  {
    Console.WriteLine($"Acquiring lock for {name}");
    lock (syncObject)
    {
      Console.WriteLine($"Hello {name}");
      Console.WriteLine($"See you later, {name}");
    }
  });
}

public async Task Test()
{
  var syncObject = new object();
  var task1 = Greetings("Foo", syncObject);
  var task2 = Greetings("Bar", syncObject);
  
  await Task.WhenAll(task1, task2);
}

```

Which of the following interleavings will be explored by Coyote?

| Output 1                                                                                                                                      | Output 2                                                                                                                                      | Output 3                                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| <code>Acquiring lock for Bar<br/>Hello Bar<br/>See you later, Bar<br/>Acquiring lock for Foo<br/>Hello Foo<br/>See you later, Foo<br/></code> | <code>Acquiring lock for Bar<br/>Acquiring lock for Foo<br/>Hello Foo<br/>See you later, Foo<br/>Hello Bar<br/>See you later, Bar<br/></code> | <code>Acquiring lock for Bar<br/>Acquiring lock for Foo<br/>Hello Foo<br/>Hello Bar<br/>See you later, Foo<br/>See you later, Bar<br/></code> |

<details>
<summary>Click to reveal the solution</summary>

Output 1 and 2 are both possible interleavings which can be explored by Coyote. Output 3 is impossible as it contains interleavings within the `lock` statement which will never happen in practice. It's interesting to note that Output 2 is explored by Coyote where there is a 'context switch' after the `Acquiring` line - this happens because Coyote treats the beginning and end of `lock` statements as scheduling points, just like it treats `Run`, `Delay` and `Yield` statements as scheduling points and explores interleavings around them.

</details>

# Example 6 (Task.ExploreContextSwitch)

```csharp

public Task Greetings(string name)
{
  return Task.Run(() =>
  {
    Console.WriteLine($"Hello {name}");
    Microsoft.Coyote.Tasks.Task.ExploreContextSwitch();
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

Which of the following interleavings will be explored by Coyote?

| Output 1                                                                                | Output 2                                                                                | Output 3                                                                                |
|-----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| <code>Hello Bar<br/>See you later, Bar<br/>Hello Foo<br/>See you later, Foo<br/></code> | <code>Hello Foo<br/>See you later, Foo<br/>Hello Bar<br/>See you later, Bar<br/></code> | <code>Hello Foo<br/>Hello Bar<br/>See you later, Bar<br/>See you later, Foo<br/></code> |

<details>
<summary>Click to reveal the solution</summary>

All three interleavings will be explored by Coyote. While Coyote normally executes statements atomically if it doesn't hit a scheduling point, `ExploreContextSwitch` forces Coyote to explore interleavings around `ExploreContextSwitch` that it otherwise would not. Note that `ExploreContextSwitch` is a no-op in production and is _only_ used to explore additional interleavings during Coyote's systematic testing.

</details>

# Example 7 (Task.ExploreContextSwitch - contd.)

```csharp

public Task Greetings(string name)
{
  Console.WriteLine($"Hello {name}");
  Microsoft.Coyote.Tasks.Task.ExploreContextSwitch();
  Console.WriteLine($"See you later, {name}");
  return Task.CompletedTask;
}

public async Task Test()
{
  var task1 = Greetings("Foo");
  var task2 = Greetings("Bar");
  
  await Task.WhenAll(task1, task2);
}

```

Which of the following interleavings will be explored by Coyote?

| Output 1                                                                                | Output 2                                                                                | Output 3                                                                                |
|-----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| <code>Hello Bar<br/>See you later, Bar<br/>Hello Foo<br/>See you later, Foo<br/></code> | <code>Hello Foo<br/>See you later, Foo<br/>Hello Bar<br/>See you later, Bar<br/></code> | <code>Hello Foo<br/>Hello Bar<br/>See you later, Bar<br/>See you later, Foo<br/></code> |

<details>
<summary>Click to reveal the solution</summary>

Output 2 is the only logical output in the above program and will be the only interleaving explored by Coyote. The above program does not contain any asynchrony which is why Output 2 is the only possible output. While `ExploreContextSwitch` explores interleavings that Coyote might otherwise miss, it does not _introduce_ any asynchrony like `Task.Run`, `Task.Yield` and `Task.Delay` statements do.

</details>
