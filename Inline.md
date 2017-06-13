#Overriding methods/functions in F##

Overloading methods is command practice in many OOP languages, but in functional languages (using a functional style) not only is this not common practice, sometimes it's not even possible. This short post highlights why, and shows the alternatives. 

**prerequisites**  

- Simple understanding of an OOP language (C# or Java)
- A simple understanding of a functional programming language. (Code examples are in F#)

**The Problem**  

As always we need some code to talk about. Here's some OOP style using F# 

    type LibraryClass() = 

        member this.PrintNumber (myNumber: int): unit = 
            printfn "%f" (float myNumber)
        member this.PrintNumber (myNumber: float): unit = 
            printfn "%f" (float myNumber)

    // Using class
    let myLibrary = new LibraryClass()
    myLibrary.PrintNumber 10
    myLibrary.PrintNumber 10.0

No surprises here, everything works as expected. So what happens if we want to write this in more of a functional style. Here's a first attempt:

    module Library = 
        // Does not compile
        let PrintNumber (myNumber: int) = 
            printfn "%f" (float myNumber)
        let PrintNumber (myNumber: float) = 
            printfn "%f" (float myNumber)

*This does not compile* and gives and the error *Duplicate definition of value 'PrintNumber'.* Argh, at first glance this would appear like F# is missing some basic machinery. 

**Discussion**  

The reason why this is happening is that the F# compiler can not determine the difference between the functions. Either the name needs to be changed or the number of parameters. If each function is rather different in nature then changing the name might be a better way to go. However, if your design is pretty good, then chances are these functions should have the same name.

**Last Effort**  

Here is the final effort and fortunately this code compiles:

    module Library = 

        let inline printNumber myNumber = 
            printfn "%f" (float myNumber)

    Library.printNumber 10
    Library.printNumber 10.0

The keyword `inline` comes to the rescue. `inline` can be used to relax the type inference on a function, despite the implementation not doing anything related to this. When the compiler reads `inline`, instead of creating a function, it replaces the caller with the function itself, ie it 'inlines' the function. `inline` can also be used for performance related tasks as a last effort. For a full read see [Inline Functions]("https://docs.microsoft.com/en-us/dotnet/articles/fsharp/language-reference/functions/inline-functions"). 

As a result, when a function is inlined, its type inference is the most general it can be. I think a quote is relevant here

> With great power comes great responsibility

Use `inline` sparingly, only to make the code more readable. As a guideline, the types of the inlined function should still have some restriction not obj -> obj. If you have this as problem, find a why to restrict the types. Perhaps by using a discriminated union (DU) and passing that through, but each case will differ.  


        




