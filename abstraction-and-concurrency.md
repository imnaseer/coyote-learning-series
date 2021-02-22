
Does the powerful tool of abstraction help us with taming the complexity of concurrency?

The answer is both a yes and a no.

Abstraction is a pervasive tool in software engineering. We use libraries and frameworks to abstract out interaction with external systems written in high level languages which compile down to assembly which compiles down to machine code. We're able to accomplish a surprisingly large amount of work with little effort using the tools of abstraction.

It's natural to apply the same tool when it comes to concurrency.

Abstraction is most effective when it can completely _hide_ and _encapsulate_ the underlying implementation details. Consider the following method.

```csharp

public bool IsLengthEven(string s)
{
  return s.Length % 2 == 0;
}

public void Test()
{
   Assert.IsTrue(IsLengthEven("ab"));
   Assert.IsFalse(IsLengthEven("abc"));
}

```

`IsLengthEven` is an exmple of a method which abstracts out a piece of computation, namely determining whether a string's length is even or not. It takes some input and returns an output. In the process, it does not modify any shared state at all and can thus be used fearlessly in any context.

Sometimes methods _have_ to read and write shared state. Consider the following class which extends our `IsLengthEven` method to a class which is given different sections of a stream over time and it calculates whether the stream seen so far is of an even length or not.

```csharp

public class StreamLengthEvenCalculator
{
  public bool IsStreamLengthEven { get; } = true;

  public void RecordSection(string section)
  {
    var isSectionEven = section.Length % 2 == 0;

    // if existing stream and new section are both even
    // or odd (i.e. their XOR is zero), then the combined
    // stream is of even length
    if (IsStreamLengthEven ^ isSectionEven == 0)
    {
      IsStreamLengthEven = true;
    }
    else
    {
      IsStreamLengthEven = false;
    }
  }
}

public async Task Test()
{
   var evenCalc = new StreamLengthEvenCalculator();

   var task1 = Task.Run(() => evenCalc.RecordSection("abc"));
   var task2 = Task.Run(() => evenCalc.RecordSection("xyz"));
 
   await Task.WhenAll(task1, task2);

   Assert.IsTrue(IsStreamLengthEven);
}

```

The above implementation has a bug where both the tasks start out by reading the existing length as even and both set as the stream length as odd. Even though we just had a single method above, it was reading and writing _shared state_ when run concurrently with itself. Abstraction can't really help us when we are racing with our own self!

In general, while layers of abstraction can help us deal with implementation complexity and construct reusable logic, they don't help us at all when it comes to shared state being read and written concurrently as they don't have the tools to _hide_ the shared state. 

This is why it's generally recommended that we minimize the shared state in our programs and this is also why paradigms like actor model of computation are helpful as they assign any given state to just a single actor all of whose 'methods' execute under a lock so that reads and writes to its local state proceed without interference.

It's interesting to realize that while programming paradigms which minimize shared state are helpful, they are far from a solution to the general problem as real world services read and write shared state from external systems, such as databases, storage buckets etc and no amount of programming language engineering can eliminate that shared state. So while developers should use the best practices to minimize and control local shared state, they should be aware of external shared state and reason through it and test it appropriately.

Is the problem of shared state hopelessly complex and is there anything we can do about it?

While abstraction doesn't by itself help hide the complexity of shared state and shared state _leaks through_ the layers of abstraction, the concept of modularity can still help us tame this complexity.

Consider the following diagram. In this diagram a lot of classes read and write from the same shared state. This is difficult to test and contains a lot of complexity.

```
  Diagram with multiple boxes reading and writing to the same or similar set of shared state.
```

We should instead strive to minimize the number of processes which have to read and write shared state. Consider this diagram instead.

```
  Diagram with different boxes reading shared state.
```

Of course we don't have islands in real systems and things interconnect. But things can interconnect at higher levels of abstraction.

```
  Diagram showing layers of methods interacting with shared state.
```

As long as each layer provides well-defined semantics on how it behaves, we can use the principles of modularity to construct systems with better understood semantics which can be tested with a focused set of concurrency tests.

