- Learning F#
- Learning Xamarin Forms
- Learing Xamairn with F#  (And Android)

- PCLs and portable - this is a bit of a mess!
    - Xamarin Studio does not always handle this correctly with F# 
    - this should get better with .NetStandard
    - some type providers won't work in a PCL without .NetStandard 
        - eg. DB: Can't access file system. 
- Some F# bugs in Xamarin Android - Images currently don't work :(
- Many F# libraries are not setup for use in a portable PCL (259/78)
- F# is really easy for learning functional concepts
    - prefers a functional style, but allows you to break this (side effects without being in IO monad)
    - compared to: 
        - Scala (which prefers OOP approach or hybrid)
        - Haskell 
            - No IDE
            - Strictly functional, no logging without IO (This is great if you know what your're doing)
            - learning material leans towards mathmatics
            - Cabal is super confusing with libraries and dependencies (getting better with stack)
            - no support for mobile (ARM) but is coming

    - fsharpforfunandprofit.com - is a great resource for learning functional programing: credit to Scott Wlacshin for this!

- Xamarin Forms is fantastic
    - It is new, so there are some bugs
    - like all things, there is a learning curve. 

- Xamarin / Xamarin Forms 
    - MVVM is everywhere
        - Not particularly functional
    - exploring Gjallarhorn
        - extensive use of observables
        - encapsulates MailboxProcessor 

- learning .NET
    - INotifyPropertyChanged 
    - mono (since I'm on a Mac)




            


