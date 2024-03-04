# Note on throwing exceptions in constructors in .NET

Tags: .NET
Publication Date: April 9, 2024
Status: In progress

[French version](https://www.notion.so/French-version-7a33049e12634fea9790876d4f2bd22c?pvs=21)

In this article, we will take a look into some details of throwing exceptions in the constructor of reference types in .NET, we will analyze it from the memory management point of view and also from code maintainability perspective.

You can find all the code written for this article in this GitHub repo. 

Let’s dive in.

## Motivation to throw exceptions in the constructor:

Throwing exception is a very common way to halt the execution when something unexpected happens. And when we instantiate new objects it’s possible to use exception to prevent the creation of an object in an invalid state. 

For instance, if we define a type that has X and Y properties to represent a position in a grid, the `Position` object can only be created for values of X and Y that are greater than 0 :

```csharp
public class Position
{
    public int X { get; private set; }
    public int Y { get; private set; }

    public Position(int x, int y)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(x, nameof(x));
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(y, nameof(y));

        X = x;
        Y = y;
    }
}
```

This example shows a case of preventing developer’s error - prevention from passing negative or nil values when creating the position instance. 

In this case we can just avoid passing the forbidden values with a prior validation but in other cases it’s not always that obvious to avoid exceptions being thrown, especially with external packages, code from `System.Collections` or `System.IO`. 

## Impact of exceptions on the object lifecycle :

Now that we have a type to work with, we can check if throwing exception in the constructor have any impact on memory management and the way the garbage collector handles it. 

In order to follow the lifecycle of the position object we can add a finalizer to print to the console when it has been finalized:

```csharp
~Position()
{
    Console.WriteLine("Position has been finalized.");
}
```

The finalizer is a special method that is called by the garbage collector before collecting an object. We use them here for a pedagogical reason, in real-life scenarios they should be left as a last resort as they add some memory overhead, their behavior is dependent on the GC, they might not be called at all, and they should never be called explicitly.

The proper way to add cleaning logic is to implement (you guessed it!) `IDisposable` and call it explicitly with the `using` keyword.

To test our code, we create a console app with a simple main method: 

```csharp
foreach (int i in Enumerable.Range(0, 5))
{
    try
    {
        var pos = new Position(1, -1);
        Console.WriteLine(pos.X);
        Console.WriteLine(pos.Y);
    }
    catch
    {
        Console.WriteLine("Exception has been thrown");
    }
}

Console.ReadLine();

while (Console.ReadKey().Key == ConsoleKey.C)
{
    GC.Collect();
}
```

The console app does mainly two things, the first part creates 5 Position objects with invalid parameters, and the second part is a loop to force garbage collection, we use a while loop because we don’t know when the finalizers are called so we will force the collection multiple times until we get what we want. 

Alongside the console app we will use the Memory Profiler to check the state of our heap. 

Running the console app until the `Console.ReadLine()` line will print : 

```
Exception has been thrown
Exception has been thrown
Exception has been thrown
Exception has been thrown
Exception has been thrown
```

And checking the heap, we see that the position object has been allocated 5 times: 

![Untitled](Note%20on%20throwing%20exceptions%20in%20constructors%20in%20NET%20360aaa9b1e6f431cab351f58b08c4858/Untitled.png)

If we inspect the content of one of the allocated objects:

![Untitled](Note%20on%20throwing%20exceptions%20in%20constructors%20in%20NET%20360aaa9b1e6f431cab351f58b08c4858/Untitled%201.png)

We see that it is initialized with default parameters. 

So, although the constructor didn’t finish running and completed with an exception, the position objects have been allocated. 

Continuing with the console app run and by forcing the garbage collection, we get in the console: 

```
Position has been finalized.
Position has been finalized.
Position has been finalized.
Position has been finalized.
```

and when we check the heap: 

![Untitled](Note%20on%20throwing%20exceptions%20in%20constructors%20in%20NET%20360aaa9b1e6f431cab351f58b08c4858/Untitled%202.png)

We can see that 4 instances have been freed and the size in bytes went from 120 to 24. The last object is still referenced so the GC won’t collect it (as we are still in the `Main()` method). 

To summarize this experiment, throwing exception in the constructor doesn’t have any impact on the lifecycle of the object who was allocated, initialized and then garbage collected normally. 

So, is it always ok to throw exceptions on the constructor? Not so fast. 

## Constructor exceptions and unmanaged resources:

The example we had was simple and had only managed resources, what happens when unmanaged resources come to play? Let’s figure it out by implementing a class that will allow us to dump the memory to a file `MemoryDumper`: 

```csharp
public class MemoryDumper : IDisposable
{
    private nint _unmanagedBuffer;
    private readonly int _size;
    private readonly Stream _destinationStream;

    public MemoryDumper(int size, string filePath)
    {
        this._size = size;
        _unmanagedBuffer = Marshal.AllocHGlobal(size);
        _destinationStream = new FileStream(filePath, FileMode.OpenOrCreate, FileAccess.Write);
    }

    public void DumpToFile()
    {
        using (var bw = new BinaryWriter(_destinationStream))
        {
            byte[] buffer = new byte[_size];
            Marshal.Copy(_unmanagedBuffer, buffer, 0, _size);
            bw.Write(buffer);
        }
    }

    public void Dispose()
    {
        if (_unmanagedBuffer != nint.Zero)
        {
            Marshal.FreeHGlobal(_unmanagedBuffer);
            _unmanagedBuffer = nint.Zero;
        }
    }
}
```

This class allocate an unmanaged memory of the given size and writes it’s content to a file, in the constructor we allocate the needed memory and we open the stream to the destination file, **the `FileStream` will throw an exception if the `filePath` is empty or null or invalid.**

To use this class and dump 1 GB of memory to a file we need to instantiate it and use the `using` keyword:

```csharp
using (var unmanagedMemoryHolder = new MemoryDumper(1024 * 1024 * 1024, string.Empty))
{
    unmanagedMemoryHolder.DumpToFile();
}
```

If you don’t see the issue with this code, let’s break it down to what it really is. This code when lowered is equivalent to this : 

```csharp
MemoryDumper unmanagedMemoryHolder = new MemoryDumper(1073741824, "wrontPath");
try
{
  unmanagedMemoryHolder.DumpToFile();
}
finally
{
  if (unmanagedMemoryHolder != null)
    unmanagedMemoryHolder.Dispose();
}
```

And in our case the exception is thrown at the constructor level, so to use this properly we need to wrap the using statement with a try catch: 

```csharp
try
{
		using (var unmanagedMemoryHolder = new MemoryDumper(1024 * 1024 * 1024, "wrontPath"))
		{
		    unmanagedMemoryHolder.DumpToFile();
		}
} 
catch 
{
		Console.WriteLine("Couldn't instantiate the memory dumper");
}
```

Even if we prevented our application from crashing we didn’t fix the real problem, we can’t call dispose on the object since we can’t reference it. That means that the 1 GB we allocated are not freed (since the exception is thrown after the allocation is done).

If our code is in production and the process is not restarted frequently that’s an issue that can lead to memory leaks.

In order to fix it, we have three options:

- Leave the unmanaged memory allocation as the last operation in the constructor.
- Make the constructor simple and with no exceptions and keep using `using` without the try-catch.
- Implement finalizers, which as we discussed should be avoided.

## Code Usability concerns

Other than the specific case that we might or might not encounter in our life as a .NET developer, exceptions in constructors can have other issues related to code usability and maintainability. 

When was the last time you have thought of the possibility of having an exception thrown when you used the constructor to instantiate an object from a library? The most probable answer is “never” and that’s for a valid reason. 

Not all constructors are implemented equal, most of them will never throw an exception, and for the ones who do, there is no way to know, since the constructor’s signature doesn’t hint to any possible issue and we are left with either reading documentation (if that exists), reading the code of the constructor, or with trial and error. 

In addition to that, having multiple constructors that throw exceptions can lead to having multiple try-catch statements all over the code base for every new instantiation. In addition to the extra level of nesting the code might turn difficult to read. 

## Alternatives to throwing exceptions in the constructor:

Whether we want to still throw exceptions in constructors or not, remains a personal or a team’s choice but it’s good to know the things to consider and the possible alternatives. 

### Throw exceptions early:

If we want to keep throwing exceptions in constructors, we need to consider the order of the operations, we should throw exceptions as early as possible and leave any allocations or unmanaged resources operations later in the constructor. 

This introduces temporal coupling and needs precaution to keep the order of execution that way and usually precautions in code should ring a bell and pushes us to rethink our design choices.

### 2-phase construction:

We can split the instance construction into two steps, a good example of this is the `SqlConnection` class in .NET: 

```csharp
using (SqlConnection connection = new SqlConnection(connectionString))
{
    command.Connection.Open();
    SqlCommand command = new SqlCommand(queryString, connection);
    command.ExecuteNonQuery();
}
```

We first instantiate the object then we call Open on it to fully have it ready. 

The constructor in this way should be lightweight and does the minimum required to get the object to a usable state. Exactly like the the constructor of `SqlConnection` , it has some validation exceptions but they are thrown first thing in the constructor and no unmanaged resources are used there, everything happens in the `Open()` method. 

This solution can be good for the example of the unmanaged memory allocation issue we had earlier.

### Factory Method

We can also have a factory method like some .NET types do. Methods like `Parse()` and `TryParse()`, unlike the constructor, are factory methods that clearly communicate the possibility of a failure or an exception when called:

```csharp
string? value = "10"
bool success = int.TryParse(value, out number);
if (success)
{
  Console.WriteLine($"Converted '{value}' to {number}.");
}
```

## Take-away points

To summarize, we have come to the conclusion that by themselves exceptions in constructors have no effects on memory management and how the GC does its work on the exception of some very specific cases that can cause memory issues when dealing with unmanaged resources.

Apart from those gray zones, throwing exceptions in constructors remains a design choice but if we don’t want to do it there are a couple of alternatives that we explored like 2-phase object construction or a factory method that c²learly communicate the intention of the possible failure and render the code more readable and self-explanatory. 

And if we decide to use exceptions It’s important to have good exception handling in our code and we should follow the  [Best Practices for exceptions](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions).