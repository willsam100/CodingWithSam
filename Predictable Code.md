#Predictable Code#

Code, code, code it's everywhere! As developers we have to read, understand and maintain it. Code that is predictable is code that can be understood just by reading it. On the contrast unpredictable code is code that is either ambiguous or misleading; in any case it will need to be executed somehow to see what it does. This post will show you how to write predictable code. 

**prerequisites**
- Assumed knowledge of an OOP language like C# or similar 
- A simple understanding of a functional programming language. (Code examples are in F#)

**Problem: mutating state**
An interface with some math operations for a list. I've modeled with an interface since the premise of OOP is about abstractions and encapsulation. 
<pre lang="csharp">
    public interface IMath
    {
        List<int> Double(List<int> input);
        List<int> Square(List<int> input);
    }
</pre>
    
First cut of the tests (for using the Math implementation of IMath)
<pre lang="csharp">
    public Main() 
    {
        IMath math = new Math();
        var inputDoubled = math.Double(new List<int> {1,2});
        var inputSquared = math.Square(new List<int> {1,2});

        Assert.Equal(2, inputMultiplied[0]);
        Assert.Equal(4, inputMultiplied[1]);

        Assert.Equal(1, inputSquared[0]);
        Assert.Equal(4, inputSquared[1]);
    }
</pre>

After code refactor
<pre lang="csharp">
    public void Main()
    {
        IMath math = new Math();
        var input = new List<int> {1,2};
        var inputDoubled = math.Double(input);
        var inputSquared = math.Square(input);

        Assert.Equal(2, inputMultiplied[0]);
        Assert.Equal(4, inputMultiplied[1]);

        // Fails
        Assert.Equal(1, inputSquared[0]);
        Assert.Equal(4, inputSquared[1]);
    }
</pre>

Clearly this is some unpredictable code; the refactor should have worked. At first glance it appears that the interface for IMath is the problem. It will return you a new list. Unfortunately, after the refactor, it is clear that it modifies the list. The problem is the List class. It is mutable in every way. The size can be changed, the elements can be changed. It's open for everyone. 

**Mutability leads to unpredictable code**
- Mutation/Mutability (definition): when a variable or object inside an instance can be changed/replaced 
- Mutation makes it hard to model time
- Mutation makes it hard to keep constraints with polymorphism (see wikipedia link below)
- Liskov substitution principle is followed by using immutable classes and is even stated on wikipeida 
> Mutability is a key issue here. If Square and Rectangle had only getter methods (i.e. they were immutable objects), then no violation of LSP could occur.
[Liskov substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle)]  

**What to do then**
- Avoid mutable classes. Copy the data and change the values during the copy, returning a new instance.
- To make a class immutable in C#: GetHashCode and Equals must be overriden
- Use a Functional language (F#), you get this all for free while still being on .NET

**How does F# do this**
Functional programming is about using functions + data to model the domain/behaviour. Encapsulation and abstractions are not goal. Common patterns are still factored out but these a for a another post. Our problem above written in F# looks like this (with the implementation):  
<pre lang="fsharp">
    module Math

    let doubleList: int List -> int List = // The type of the function. It takes a list of ints and returns a list of ints
        fun input -> List.map (fun x -> x * 2) input
    let squareList: int List -> int List // The type of the function. It takes a list of ints and returns a list of ints
        fun input -> List.map (fun x -> x * x) input

    let main() = 
        let input = [1;2]
        let inputDoubled = doubleList input
        let inputSquared = squareList input 

        Assert.Equal(2, inputDoubled.[0])
        Assert.Equal(4, inputDoubled.[1])

        Assert.Equal(1, inputSquared.[0])
        Assert.Equal(4, inputSquared.[1])
</pre>

**Why does this work**
- The list type in F# (which is different to C#'s list) is immutable so the result is copied and changed.
- It's harder to create mutable objects:
- F# creates constants by default (these are called bindings, and means the reference can't be changed. If the object is not immutable then elements inside the object can be change eg. an array)
- A variable in F# requires the ``mutable`` keyword (It's more work, and some IDEs highlight the keyword to be explicit)

**What's the point**
- immutability makes the code predictable. It can understood without executing it.  
- Mutability and unpredictable code requires you execute the code somehow to check it does what you think. 
~~Use F# if you don't like writing boilerplate code and/or want your code to just work~~ Use F#
  
**Immutability FTW**

NB: I claimed that FP does not have abstractions as goal. FP makes great leaps to reduce boilerplate code and encourage code reuse, but takes a mathematical approach to do this. Higher order functions and Monads (eg. State Monad) are examples of these.