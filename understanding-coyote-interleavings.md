
Coyote explores a large number of possible task interleavings in your programs and services, a number of which happen fairly rare and can be annoying to your users at best, or catastrophic in the worst case.

Interestingly and importantly though, Coyote does not explore _all_ interleavings in your program. Deeply understanding which interleavings Coyote explores and which it does not is key to using it effectively. While you can go a long distance without precisely knowing how Coyote picks out "scheduling points" around which it explores interleavings, you may end up missing important scheduling points in your programs thus preventing Coyote from finding critical concurrency bugs in your programs.

Let us explore this important topic through a series of code examples. Consider the following "sequential" code snippet.

```csharp

public void Greetings(string name)
{
  Console.WriteLine($"Hello {name}");
  Console.WriteLine($"See you later, {name}");
}

public void Test()
{
  Greetings("Foo");
  Greetings("Bar");
}

```

The above program does not contain any asnychrony and has only one output.

```
  Hello Foo
  See you later, Foo
  Hello Bar
  See you later, Bar
```

Let's introduce tasks in the program above.

```csharp

public void Greetings(string name)
{
  Console.WriteLine($"Hello {name}");
  Console.WriteLine($"See you later, {name}");
}

public async Task Test()
{
  var gt1 = Task.Run(() => Greetings("Foo"));
  var gt2 = Task.Run(() => Greetings("Bar"));

  await Task.WhenAll(gt1, gt2);
}

```

The above program has asynchrony as it spins up two tasks running concurrently with each other, and then waits for both of them to complete.

How many possible outputs do you expect to see if you run the above program with Coyote? Here is one output.

```
  Hello Bar
  See you later, Bar
  Hello Foo
  See you later, Foo
```

And here is another one.

```
  Hello Foo
  See you later, Foo
  Hello Bar
  See you later, Bar
```

What about the following?

```
  Hello Foo
  Hello Bar
  See you later, Bar
  See you later, Foo
```

The above output is clearly possilbe when two tasks are running concurrently with each other. Interestingly when you run your test with Coyote, the above interleaving will _never_ be explored. Hmmm - this may come as a surprise to you. And perhaps even make you doubt the effectiveness of using Coyote if it can't explore such simple interleavings.

We'll soon reason through why missing some interleavings is actually _okay_ in most cases, perhaps even advantageous as it can help Coyote scale to large programs. But first let's understand why Coyote behaves the way it does.

Paradoxically, while Coyote does explore the concurrency in your program, it never executes any two tasks in _parallel_ - in fact it goes to the other extreme and executes your entire program _sequentially_, always, even if your machine has enough cores to run the tasks in parallel.

It executes one of the tasks till it hits a "scheduling point". It then suspends that task (if it wasn't at the end of its execution already) and picks up one of the available set of task (including the same task, if it didn't already terminate) and runs it sequentially, without interference from any other task till it hits another scheduling point. It then picks another available task to run, and so on till either all the tasks terminate or Coyote decides to terminate this iteration and start a new iteration exploring a different set of interleavings.

Following are some of the scheduling points Coyote recognizes:

* `Task.Run`
* `Task.Delay`
* `Task.Yield`
* Entry and exit of C# `lock` statements
* `Microsoft.Coyote.Tasks.Task.ExploreContextSwitch` (this is an artificial scheduling point you can inject in your programs as required)

How many scheduling points did you see in the code example earlier? There were only two scheduling points: the two `Task.Run` calls in that program. Let's sprinkle some more debugging output in that program and take a look at it again.

```csharp

public void Greetings(string name)
{
  Console.WriteLine($"Hello {name}");
  Console.WriteLine($"See you later, {name}");
}

public async Task Test()
{
  Console.WriteLine("Beginning Test");

  var gt1 = Task.Run(() => Greetings("Foo"));
  Console.WriteLine("Spawned the first task");

  var gt2 = Task.Run(() => Greetings("Bar"));
  Console.WriteLine("Spawned the second task");

  await Task.WhenAll(gt1, gt2);

  Console.WriteLine("Ending Test");
}

```

Coyote starts the program with a single "thread of execution" in the `Test` method, let's call it `te-1` and run its till it hits a scheduling point, which is the first `Task.Run` call in the `Test` method above. `Task.Run` results in the creation of another thread of execution, let's call it `te-2`. We see the following output on the screen.

```
  Beginning Test
```

Coyote now has to decide whether to resume execution of `te-1` or `te-2`. In this iteration it chooses `te-1` and continues execution till it hits the next scheduling point, which is the second call to `Task.Run`. That call spawns another thread of execution, let's call it `te-3`. We see the following additional output on the screen.

```
  Spawned the first task
```

Coyote now has to decide whether to resume execution of `te-1`, `te-2` or `te-3`. It chooes `te-3` which beging execution of `Greetings("Bar")` and it runs till completion as it doesn't contain any scheudling point.

```
  Hello Bar
  See you later, Bar
```

Coyote now has to decide between choosing `te-1` and `te-2`. It chooses `te-1` which runs and hits an await statement. The `await` statement is not a scheduling point but it does block the task as `gt1` still hasn't finished yet. 

```
  Spawned the second task
```

Coyote resumes execution of the only thread of execution that it can which is `te-2` which runs `Greetings("Foo")` all the way to completion.

```
  Hello Foo
  See you later, Foo
```

Coyote only has one thread of execution to choose from (`te-1`) which it chooses and runs to completion.

```
  Ending Test  
```

Here's the complete output of the program above.

```
  Beginning of Test
  Spawned the first task
  Hello Bar
  See you later, Bar
  Spawned the second task
  Hello Foo
  See you later, Foo
  Ending Test  
```

You'll see various other interleavings in different Coyote iterations but the blocks of text which were emitted together in the example will _always_ be emitted together as they didn't contain any scheduling points to break their sequential execution.

Here's a challenge for you. Can you introduce an artificial scheduling in our program so Coyote can emit the following output in one of its iterations?

```
  Hello Foo
  Hello Bar
  See you later, Bar
  See you later, Foo
```

<details>
  <summary>Click to reveal the solution!</summary>

  ```csharp
  public void Greetings(string name)
  {
    Console.WriteLine($"Hello {name}");
    Microsoft.Coyote.Tasks.Task.ExploreContextSwitch();
    Console.WriteLine($"See you later, {name}");
  }
  ```

  The introduction of `ExploreContextSwitch` between the two `Console.WriteLine` statements causes Coyote to break the serial execution and explore other interleavings around them.
</details>

Now that you understand how Coyote explores the concurrency in your program through a simple example, you can continue your learning through the following articles.

* Exercise and refine your understanding through more illustrative examples
* Understand why Coyote is effective at finding bugs in real-world programs without requiring you to inject artificial scheduling points all over the place
* Develop an intuitive understanding of the state space explosion caused by concurrency and how Coyote's decision to not explore every possible interleaving helps it scale

