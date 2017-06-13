**Prerequisites**

- A simple understanding of F#
- A recap of F# Mailboxes: https://fsharpforfunandprofit.com/posts/concurrency-actor-model/
- A basic understanding of a sqlite database

This post aims to build on the discussion started in the prerequisites. Mailbox processors are a great alternative to use variables and locks. There are many blog other posts on MailboxProcessors, however, few of them mention how to retrieve the state they hold rather than just printing it to console. This post will explain the different approaches, while applying this to a typical Xamarin app problem; a database connection. 

**Background**

Before jumping in to the details, it is important to establish why this is important. State is a requirement of a program, but how that state is modelled is up to the programmer. Most main-stream programming languages (those OOP ones) model state with many variables. An alternative approach (and better in my opinion) is to eliminate most of the variables by modelling state through functions (how that is done is not the focus of this post). For those few variables that remain, a high quality can now taken to make sure everything is thread safe and protected. F# mailboxes are a nice way of doing that. As already stated, one of those variables is the database connection. With that out of the way, time for some code!

**A database connection: The standard approach**

    open SQLite
    open System
    open System.IO

    [<CLIMutable>]
    type UserName = {Id: Guid; FirstName: string; LastName:string }

    type DatabaseManager() = 

        let monitor = new Object()
        let dbName = "database.db3"
        let mutable connection: SQLiteConnection = null

        member this.Init() =
            lock monitor (fun () -> 
                connection <- new SQLiteConnection(Path.Combine(path, dbName), false) )
                connection.CreateTable<UserName>() |> ignore

        member this.GetAllUsers(): Seq<UserName> = 
            lock monitor (fun () -> 
                connection.Table<UserName>())

        //Rest of read write data

    // In app startup 
    let deviceDatabasePath = "" // real implementation would db path from IOC container
    let databaseManager = new DatabaseManager()
    databaseManager.Init(deviceDatabasePath)

As a staring point, this F# code is a very typical approach to creating a database connection that is thread safe. To explain the main points of the code, UserName is our DTO class that we're using for a single table. For the connection, a synchronous connection is being used. Normally an async connection is used, but in the context of this post, an async connection offers little benefits. I've also skipped any form of interfaces for testing and will leave adding that in as an exercise for the reader. 
 
** Step one: A mailbox**  

First, let's take out the lock and wrap the connection in a mailbox.

    open SQLite
    open System
    open System.IO

    [<CLIMutable>]
    type UserName = {Id: Guid; FirstName: string; LastName:string }
    type Message = Init of string

    type DatabaseManager() = 

        let mutable connection: SQLiteConnection = null
        let dbName = "database.db3"
        let mailbox = MailboxProcessor.Start(fun inbox -> 

            let rec loop () = async {
                let! msg = inbox.Receive()
                match msg with 
                | Init path -> 
                    connection <- new SQLiteConnection(Path.Combine(path, dbName), false)
                    connection.CreateTable<UserName>() |> ignore

                return! loop ()
            }
            loop ())

        member this.Init(path: string) =
            mailbox.Post(Init path)    

        // rest of methods to read/write data

    // In app startup 
    let deviceDatabasePath = "" // real implementation would db path from IOC container
    let databaseManager = new DatabaseManager()
    databaseManager.Init(deviceDatabasePath)

Now we have a ``MailboxProcessor`` holding our ``SQLiteConnection`` and we also have a message type to create the connection. Because the connection is stored inside the mailbox, we know that things are thread safe. We're not finished yet though, since there is no way to get data out of the ``MailboxProcessor``

**Getting data out: The many possibilities**

There are a few ways to get data out of the mailbox. They are:
- Reply synchronously
- Reply asynchronous
- use events 

lets use each approach to get a feel for what works best. Here are the code sections that need to be added/updated for replying synchronously:

**Reply to me synchronously**

    // Update the type to handle replying
    type Message = Init of string | GetUserNames of AsyncReplyChannel<seq<UserName>>

        // update mailbox to handle the message and reply with the data
        match msg with 
        | Init path -> 
            connection <- new SQLiteConnection(Path.Combine(path, dbName), false)
            connection.CreateTable<UserName>() |> ignore
        | GetUserNames replyChannel -> 
            replyChannel.Reply(connection.Table<UserName>()) 

    // Add a method on the class to use the mailbox and reply synchronously with the data
    member this.GetAllUsers () =
        mailbox.PostAndReply GetUserNames 

``Message`` now includes a branch with ``GetUserNames`` that also carries some data with it; the reply channel. The reply channel is also typed with the response data, in this case ``seq<UserName>``. Once the message type has been updated, the pattern match inside the mailbox will give a warning till it has been updated to handle the ``GetUserNames`` case. The matching case is very simple. Just use the connection to pull out the data and pass it into the replyChannel's reply method. The last section of code is added to  ``DatabaseManager`` as a public method. In here, we specify that the request should be made synchronously, ie post the message and block on the same thread for the response. 
Here is the full output: 

    open SQLite
    open System
    open System.IO

    [<CLIMutable>]
    type UserName = {Id: Guid; FirstName: string; LastName:string }
    type Message = Init of string | GetUserNames of AsyncReplyChannel<seq<UserName>>

    type DatabaseManager() = 

        let mutable connection: SQLiteConnection = null
        let dbName = "database.db3"
        let mailbox = MailboxProcessor.Start(fun inbox -> 

            let rec loop () = async {
                let! msg = inbox.Receive()
                match msg with 
                | Init path -> 
                    connection <- new SQLiteConnection(Path.Combine(path, dbName), false)
                    connection.CreateTable<UserName>() |> ignore
                | GetUserNames replyChannel -> 
                    replyChannel.Reply(connection.Table<UserName>()) 
                
                return! loop ()
            }
            loop ())

        member this.Init(path: string) =
            mailbox.Post(Init path)   

        member this.GetAllUsers () =
            mailbox.PostAndReply GetUserNames 

        // rest of methods to read/write data

    // In app startup 
    let deviceDatabasePath = "" // real implementation would db path from IOC container
    let databaseManger = new DatabaseManager()
    databaseManger.Init(deviceDatabasePath)
    let usernames = databaseManger.GetAllUsers()

**Doing things asynchronously**

For the next variation we have replying asynchronously. To change the synchronously example to asynchronously, only one method needs to changed:

        member this.GetAllUsers (): Async<seq<UserName>> =
            mailbox.PostAndAsyncReply GetUserNames 

        let usernames = databaseManger.GetAllUsers() |> Async.RunSynchronously

As stated, the change is very simple, just use ``PostAndAsyncReply``, and the response will be asynchronous. For the example invoking the method, I have called it synchronously (``RunSynchronously``), but this should be avoided to get the benefits from asynchronous execution.

**An event to rule them all**

The final choice for getting data out of a ``MailboxProcessor`` is by using events. Don Syme wrote a great a post on this [here]("https://blogs.msdn.microsoft.com/dsyme/2010/02/15/async-and-parallel-design-patterns-in-f-agents/"). Let's apply that to our ``SQLiteConnection`` problem. First off let's update our ``Message`` to have the values we need. 

    type Message = Init of string | GetUserNames | Create of UserName

The ``GetUserNames`` is now used as an enum since we'll use an event to return the data. I've also taken the liberty of adding anther choice to the message that will allow a ``UserName`` to be added to the database. This choice can be added to any/all of the examples if you want to test them with some data. Now we need to add the event along with a few helper methods to the ``DatabaseManager``

        // Declared at the top, in DatabaseManager
        let event = new Event<seq<UserName>>()
        let context = 
            match SynchronizationContext.Current with 
            | null -> new SynchronizationContext()
            | ctx -> ctx

        let raiseEvent args = 
            context.Post((fun _ -> event.Trigger args), state=null)

As discussed earlier, the event is typed with our return type, in this case ``seq<UserName>``. We also capture a context that we can post back on, generally this will be the main thread of a GUI app. Finally a helper ``raiseEvent`` was added that raises the event on the captured context. Next up is to update the match block in our mailbox:

    // Inside the mailbox
    match msg with 
    | Init path -> 
        connection <- new SQLiteConnection(Path.Combine(path, dbName), false)
        connection.CreateTable<UserName>() |> ignore
    | GetUserNames -> 
        connection.Table<UserName>() |> raiseEvent
    | Create userName -> 
        connection.Insert(userName) |> ignore

The first match is the same as before. ``GetUserNames`` now loads all the data and calls the helper method we declared earlier. ``Create`` is also very simple, it inserts the userName into the database and ignores the return value of the insert. We're nearly done, we just need to expose the new functionality on the ``DatabaseManager`` via some methods. 

    // In DatabaseManager
    member this.Create(firstName: string, lastName: string) =
        mailbox.Post <| Create {Id = Guid.NewGuid(); FirstName = firstName; LastName = lastName}

    member this.ReceiveAllUsers = 
        event.Publish

    member this.RequestAllUsers () =
        mailbox.Post GetUserNames 

Adding data is a single method that takes in the required. It simple makes a ``Post`` call to mailbox with the ``Create`` tag and the data. Getting the data our now requires two methods. Because we're using events to get the data out, the clint will need to subscribe to the event. ``ReceiveAllUsers`` allows the clint to do this by exposing the ``Publish`` method on the event. Once the client has subscribed, the client can call ``RequestAllUsers`` and the data will be received on the event subscription. ``RequestAllUsers`` just makes a ``Post`` call to mailbox with the ``GetUserNames`` choice. 

Here is some sample client code I used in an F# repl to test this out:

    let deviceDatabasePath = "/Users/sam.williams/Desktop/" // real implementation would be db path from IOC container
    let databaseManger = new DatabaseManager()
    databaseManger.Init(deviceDatabasePath)
    databaseManger.Create("Sam", "Williams")
    databaseManger.ReceiveAllUsers.Add(fun users -> users |> Seq.iter (printfn "%A"))
    databaseManger.RequestAllUsers()

It's rather straight forward but I will go over it quickly. The first three lines are creating an instance and setting up the database as before. We then create a record so we can get something out. We then add our subscription the the ``databaseManager`` event. For this trivial example, the users will be printed to the console. Finally, the request is made to get all the users. The full listing for the mailbox with events is listed at the end of the post. 

To wrap-up, there are three possible methods to get state out of an F# mailbox, reply synchronously, reply asynchronous and via events. Hopefully one of these will be suitable the next time you need to protect a variable from thread bugs.

**tl;dr: full fsx script example with events**

Here is the full listing for the mailbox with events. I've included the required imports to run this as an fsx script on a Mac. For this script I've also included some very crude error handling, useful for getting the script running (but not production quality):

    #r "../packages/sqlite-net-pcl/lib/portable-net45+wp8+wpa81+win8+MonoAndroid10+MonoTouch10+Xamarin.iOS10/SQLite-net.dll" 
    #r "../packages/SQLitePCLRaw.core/lib/Xamarin.Mac20/SQLitePCLRaw.core.dll"
    #r "../packages/SQLitePCLRaw.provider.e_sqlite3.macos/lib/Xamarin.Mac20/SQLitePCLRaw.provider.e_sqlite3.dll"
    #r "../packages/SQLitePCLRaw.bundle_green/lib/Xamarin.Mac20/SQLitePCLRaw.batteries_v2.dll"
    #r "../packages/SQLitePCLRaw.provider.sqlite3.ios_unified/lib/Xamarin.iOS10/SQLitePCLRaw.provider.sqlite3.dll"
    #r "../packages/SQLitePCLRaw.provider.e_sqlite3.net45/lib/net45/SQLitePCLRaw.provider.e_sqlite3.dll"

    open SQLite
    open System
    open System.IO
    open System.Threading

    [<CLIMutable>]
    type UserName = {Id: Guid; FirstName: string; LastName:string }
    type Message = Init of string | GetUserNames | Create of UserName

    type DatabaseManager() = 

        let event = new Event<seq<UserName>>()
        let mainThread = 
            match SynchronizationContext.Current with 
            | null -> new SynchronizationContext()
            | ctx -> ctx

        let raiseEvent args = 
            mainThread.Post((fun _ -> event.Trigger args), state=null)

        let mutable connection: SQLiteConnection = null
        let dbName = "database.db3"
        let mailbox = MailboxProcessor.Start(fun inbox -> 
        
            // Helper to get out any exceptions that may occur
            let rec getException indent (e:exn) = 
                printfn "Finding exception message"
                if e.InnerException = null then 
                    sprintf "%s\n%s" e.Message e.StackTrace
                else 
                    sprintf "%s%s\n%s" indent (getException (sprintf "%s\t" indent) e.InnerException) e.StackTrace 

            // generalised catch for any exceptions on each branch
            let catch f = 
                try 
                    f () 
                with 
                | e -> printfn "Error: %s\n%s" e.Message (getException "" e)
                
            let rec loop () = async {
                let! msg = inbox.Receive()
                match msg with 
                | Init path -> 
                    catch <| fun () -> 
                        connection <- new SQLiteConnection(Path.Combine(path, dbName), false)
                        connection.CreateTable<UserName>() |> ignore
                        printfn "Connection created!"
                | GetUserNames -> 
                    catch <| fun () -> 
                        connection.Table<UserName>() |> raiseEvent
                | Create userName -> 
                    catch <| fun () -> 
                        connection.Insert(userName) |> ignore

                return! loop ()
            }
            loop ())

        member this.Init(path: string) =
            mailbox.Post(Init path)   

        member this.Create(firstName: string, lastName: string) =
            mailbox.Post <| Create {Id = Guid.NewGuid(); FirstName = firstName; LastName = lastName}

        member this.RequestAllUsers () =
            mailbox.Post GetUserNames 

        member this.ReceiveAllUsers = 
            event.Publish

        // rest of methods to read/write data

    // In app startup 
    let deviceDatabasePath = "/Users/sam.williams/Desktop/" // real implementation would db path from IOC container
    let databaseManger = new DatabaseManager()
    databaseManger.Init(deviceDatabasePath)
    databaseManger.Create("Sam", "Williams")
    databaseManger.ReceiveAllUsers.Add(fun users -> users |> Seq.iter (printfn "%A"))
    databaseManger.RequestAllUsers()
















