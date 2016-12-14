Predictable Code

Code, code, code it's everywhere! As developers we have read, understand it and maintain. Code that is predictable is code that can be understood just be reading it. On the contrast unpredictable code is code that is either ambiguous or misleading; in any case you're going to have to execute it somehow. This post will show you how to write predictable code. 

*prerequisites*
- OOP language like C# or similar 
- A simple understanding of a functional programming language., 
- Code examples are in F#


*Problem: mutating state*

Some math actions that operation on a list
    public interface IMath
    {
        List<int> Double(List<int> input);
        List<int> Square(List<int> input);
    }
    
First cut of the tests
    public MathTest
    {
        public void Main() 
        {
            IMath math = new Math();
            var inputDoubled = math.Multiply(new List<int> {1,2})
            var inputSquared = math.Square(new List<int> {1,2})

            Assert.Equal(2, inputMultiplied[0])
            Assert.Equal(4, inputMultiplied[1])

            Assert.Equal(1, inputSquared[0])
            Assert.Equal(4, inputSquared[1])
        }
    }

After code refactor
    public MathTest
    {
        public void Main() 
        {
            IMath math = new Math();
            var input = new List<int> {1,2}
            var inputDoubled = math.Multiply(input)
            var inputSquared = math.Square(input)

            Assert.Equal(2, inputMultiplied[0])
            Assert.Equal(4, inputMultiplied[1])

            // Fails
            Assert.Equal(1, inputSquared[0])
            Assert.Equal(4, inputSquared[1])
        }
    }

Clearly this is some unpredictable code; the refactor should have worked. At first glance it appears that the interface for IMath is the problem. It will return you a new list. unfortunately, after the refactor, it is clear that it modifies the list. The problem is List. It is mutable in every way. The size can be changed, the elements can be changed. It's open for everyone. 

*Mutability leads to unpredictable code*
- Mutation/Mutability: when a variable or object inside an instance can be changed/replaced 
- Mutation makes it hard to model time
- Mutation makes it hard to keep constrainst with polymorphism 

*Solution*
- Avoid mutable classes. Copy the data and change the values during the copy, returning a new instance.
- To make a class immutable: GetHashCode and Equals must be overriden
- Use a Functional language (F#), you get this all for free while still being on .NET

*How does F# do this*
    
    module Math

    let doubleList: int list -> int List = 
        fun input -> List.map (fun x -> x * 2) input
    let squareList: int List -> int List 
        fun input = List.map (fun x -> x * x) input

    let main() = 
        let input = [1;2]
        let inputDoubled = doubleList input
        let squareDoubled = squareList input 

        Assert.Equal(2, inputMultiplied.[0])
        Assert.Equal(4, inputMultiplied,[1])

        Assert.Equal(1, inputSquared.[0])
        Assert.Equal(4, inputSquared.[1])


*Why does this work*
- the list type in F# (which is different to C#'s list) is immutable so the result is copied and changed.
- It's harder to create mutable objects:
- F# creates constants by default (these are called bindings, and means the reference can't be changed. If the object is not immutable then elements inside the object can be change eg. an array)
- A variable in F# required the ``mutable`` keyword (It's more work)

*What's the point*
- immutability makes the code predictable. It can understood without executing it.  
- Mutability and unpredictable code requires you 
- Liskov substitution principle is followed by being immutable and is even stated on wikipeida 
> Mutability is a key issue here. If Square and Rectangle had only getter methods (i.e. they were immutable objects), then no violation of LSP could occur.
["LSP"(https://en.wikipedia.org/wiki/Liskov_substitution_principle)]    


*immutability FTW*
