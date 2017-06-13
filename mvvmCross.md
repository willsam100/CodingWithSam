#First view model with MvvmCross#

**prerequisites**

- Understanding of C#, Xamarin, Mvvm/MvvmCross

F# supports Object-Orientated programming. MvvmCross is a framework for building apps with MvvM design pattern. This blog post walks through combing them using a PCL. A new project will be created. A new F# core PCL will then be added. The C# core will then be translated to F#. A small problem awaits....

**Getting setup**

Follow the IDE template and get an MvvmCross project setup with a C# PCl. We will change the core once this has been added. If you haven't added a template you can add it to your IDE from [Xamarin Studio]("https://github.com/jimbobbennett/MVVMCross.XSAddIn") or [Visual Studio]("https://marketplace.visualstudio.com/items?itemName=JimBobBennett.MvvmCrossforVisualStudio"). Once the project is created and all the packages have been installed, run your app to double check everything installed correctly. I've named my app `MvvmCrossViewModelDemo`

**Adding F# core**

Next we will add an F# core to replace the C# Core. On the solution right click and add a Forms PCL, make sure to choose F# for the language drop down. For the name you can choose anything, this post will use 'central'. Once the F# PCL has been added, do the following:
- remove windows 8.1 support from the pcl, ie select '.NET Portable Subset (.NET Framework 4.5, Windows 8)' 
    -- This is showing as profile7 on my machine
- add the required packages via Nuget (or Paket if you know how)
    -- MvvmCross
    -- MvvmCross.Binding 
    -- MvvmCross.Core
    -- MvvmCross.Platform
    -- Xamarin.Forms

**Porting MvxApplication**

Rename the file `MyPage.fs` to `App.fs`

The `App.cs` is the first block of code to port. It appears as follows: 

    [lang=csharp]
    public class App : MvvmCross.Core.ViewModels.MvxApplication
	{
		public override void Initialize()
		{
			CreatableTypes()
				.EndingWith("Service")
				.AsInterfaces()
				.RegisterAsLazySingleton();

			RegisterAppStart<ViewModels.FirstViewModel>();
		}
	}

This is a direct port so the F# looks as follows:

    // F# port of MvvmCross MvxApplication
    type App() = 
        inherit MvvmCross.Core.ViewModels.MvxApplication() 
        
        override this.Initialize() = 
            this.CreatableTypes()
                .EndingWith("Service")
                .AsInterfaces()
                .RegisterAsLazySingleton()
            this.RegisterAppStart<ViewModels.FirstViewModel>()

Update the `App.fs` file to be as follows:

    namespace MvvmCrossViewModelDemo.Central
    open MvvmCross.Platform.IoC

    type App() = 
        inherit MvvmCross.Core.ViewModels.MvxApplication() 

        override this.Initialize() = 
        
            this.CreatableTypes()
                .EndingWith("Service")
                .AsInterfaces()
                .RegisterAsLazySingleton()

            this.RegisterAppStart<ViewModels.FirstViewModel>()


** Adding views**

Simply add the views to the F# PCL project as you would with a C# project. Be sure to select `Forms ContentPage Xaml`
When added, copy the xaml from below (or the C# project) and paste into the xaml for the F# project. Don't forget to update the two class references (`x:Class`) for each of the files at the top of the xaml. I've included just the FirstPage.xaml: 

FirstPage.xaml

    [lang=xml]
    <?xml version="1.0" encoding="utf-8"?>
    <ContentPage xmlns="http://xamarin.com/schemas/2014/forms" xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml" xmlns:forms="using:Xamarin.Forms" x:Class="MvvmCrossViewModelDemo.Central.FirstPage" Title="First Page">
        <!-- Jul 22 2015 Xamarin have not yet provided Device.OnPlatform property for W81. The below syntax works by putting "5, 0, 5, 95" into the default-->
        <ContentPage.Padding Thickness="5, 0, 5, 95">
            <OnPlatform x:TypeArguments="Thickness" iOS="5, 20, 5, 0" Android="5, 0, 5, 0" WinPhone="5, 0, 5, 0" />
        </ContentPage.Padding>
        <ContentPage.ToolbarItems>
            <ToolbarItem Name="Menu1" Text="About" ClassId="About" Order="Primary" Command="{Binding ShowAboutPageCommand}">
                <ToolbarItem.Icon>
                    <OnPlatform x:TypeArguments="FileImageSource" WinPhone="Toolkit.Content/ApplicationBar.Add.png" />
                </ToolbarItem.Icon>
            </ToolbarItem>
        </ContentPage.ToolbarItems>
        <StackLayout Spacing="10" Orientation="Vertical">
            <Label FontSize="24" Text="Enter your nickname in the box below" />
            <Entry Placeholder="Who are you?" TextColor="Red" Text="{Binding YourNickname}" />
            <Label FontSize="24" Text="{Binding Hello}" />
        </StackLayout>
    </ContentPage>


**Porting the view models**


A full version of the `App.fs` with the view models is at the bottom of this blog if you want to skip to the end. 
Let's start with the `AboutViewModel`, paste the following code below in `App.fs` below `open MvvmCross.Platform.IoC`:

    open MvvmCross.Core.ViewModels

    module ViewModels = 

        type AboutViewModel() =
            inherit MvxViewModel()

We need the import `MvvmCross.Core.ViewModels` in order to subclass `MvxViewModel`, just as we do in C#. (Note the project won't compile until everything is complete)

**Porting the FirstViewModel**

The `FirstViewModel` requires a bit more work to translate, so is covered bit by bit. `FirstViewModel` should be placed below `AboutViewModel` in the `ViewModels` modules. F# is indentation sensitive so this means it should be indented two tabs.

Create the class with subclass: 

    type FirstViewModel() =  
        inherit MvxViewModel()

Add the variable to hold the string with its associated property for `YourNickName` :

    let yourNickname = ref ""
    member this.YourNickname 
        with get() = !yourNickname 
        and set(value) =
            if (this.SetProperty(yourNickname, value)) then 
                this.RaisePropertyChanged("Hello")
            else 
                ()

My first attempt at doing this, (and I'm sure most people learning F# would be the same), was to use `let mutable yourNickname = ""` to hold the mutable string. However it doesn't work, (the code compiles but value/property is never updated). This is because of the implementation of `SetProperty` in MvvmCross. 

The implementation of `SetProperty` is: 

    [lang=csharp]
    protected bool SetProperty<T> (ref T storage, T value, [CallerMemberName] string propertyName = null)
    {
        if (object.Equals (storage, value)) {
            return false;
        }
        storage = value;
        this.RaisePropertyChanged (propertyName);
        return true;
    }

Notice that `ref` that is passed in. In F# if you take a `ref` of a mutable (ie `ref yourNickname`) you don't get the same result as taking a ref in C#. In F# it creates a wrapper that holds a mutable value. This is not what we want as we won't be able to compare it with the raw string; it wil always return false. For more details see this post [Pass by reference]("https://davefancher.com/2014/03/24/passing-arguments-by-reference-in-f/"). By using a ref value in the view model, this problem is avoided and all we need to remember is to dereference `yourNickname` when we want the string. 

As a result of using the ref we now have this `!` in the getter: 
    
    with get() = !yourNickname

As stated earlier, we need to return a string and we're using a reference type which is a wrapper type. The exclamation mark is a short hand to return the enclosed mutable field.

Next we can add the getter the for the string. Again we use `!` to get the string value out:

    member this.Hello = sprintf "Hello %s" !yourNickname

Finally we can add in the command to navigate to the `AboutViewModel`:

    member private this.ShowAboutViewModel() = 
        this.ShowViewModel<AboutViewModel>()

    member this.ShowAboutPageCommand 
        with get() = new MvxCommand(fun () -> this.ShowAboutViewModel() |> ignore)

You'll notice that we had to create a private method to make the call to base method `ShowViewModel`. The reason for this is that `MvxCommand` command takes in a function, and calling `ShowViewModel` must be called from a subclass ie it's not public method. The compiler gives an error if a function is passed in directly that calls `ShowViewModel` as it cannot prove that the call is made from a subclass ie it's enforcing the method is not public. The private method to `FirstViewModel` proves that the call is made from a subclass, so everything now compiles.

That's it for porting the C# to F#. Note that with F# we don't need lots of files. The view models are declared above the app class, and F# compiler enforces that, making the code easy to navigate. Here is how the entire `App.fs` should look:

    namespace MvvmCrossViewModelDemo.Central

    open MvvmCross.Platform.IoC
    open MvvmCross.Core.ViewModels

    module ViewModels = 

        type AboutViewModel() =
            inherit MvxViewModel()

        type FirstViewModel() =  
            inherit MvxViewModel()
        
            let yourNickname = ref ""

            member this.YourNickname 
                with get() = !yourNickname 
                and set(value) =
                    if (this.SetProperty(yourNickname, value)) then 
                        this.RaisePropertyChanged("Hello")
                    else 
                        ()

            member this.Hello = sprintf "Hello %s" !yourNickname

            member private this.ShowAboutViewModel() = 
                this.ShowViewModel<AboutViewModel>()

            member this.ShowAboutPageCommand 
                with get() = new MvxCommand(fun () -> this.ShowAboutViewModel() |> ignore)
            
    type App() = 
        inherit MvvmCross.Core.ViewModels.MvxApplication() 

        override this.Initialize() = 
        
            this.CreatableTypes()
                .EndingWith("Service")
                .AsInterfaces()
                .RegisterAsLazySingleton()

            this.RegisterAppStart<ViewModels.FirstViewModel>()
            
        

**The last wiring effort**

For both the iOS and droid project, remove the project reference to the C# core and change it to the F# core (named Central in the post). Alternativly you could delete the C# core if you want to be sure you are using F#. Compile these projects, and you will get a compile error in the `Setup.cs` files. Change `Core` to `Central` in `Setup.cs` and everything should compile, ie:

    [lang=csharp]
    return new Central.App();

**Wrapping up**

This post has shown how to create an F# core with mvvmCross. Remember the following:
- use ref for your variables if you need to call `SetProperty`, or any other method that takes in a ref value
- use private methods if need to pass a function that invokes a protected method on a base class

As always, leave a comment if you found this helpful or know a better way!













