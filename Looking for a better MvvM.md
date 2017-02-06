#Looking for A better [functional] MvvM#

This post explores some of the current problems I have experienced with MvvM frameworks, specifically MvvM and page navigation, and their current OOP approach. Most of the this is from the perspective of mobile apps (Xamairn Forms) but should also be transferrable to WPF. I'll cover some of the features that I would like from more of a functional approach.

##Overview of MvvM##
MvvM is a design pattern intended to allow the testing of the view, since current view controls were written without testing in mind. The primary piece of the pattern is the view model that aims to describe the view while still being testable (does not have any external dependencies that cant' be mocked). The view is self explanatory, and the model is everything else in the app. It's implied the view does not talk to the model since this would not be testable. 

##Page navigation with MvvM##
The above brief overview only describes a single page. Looking at a few of the most popular mvvm frameworks for mobile they have a few commonalities in how they handle page navigation. 
- Convention over configuration
- VM first or View first
- Some form of presenter

Let's look at each of these:

#####Convention over configuration#####
This means the framework does not require to be told how to link a view model to a view. Instead they use a naming conventional to figure this out. eg. given LandingViewModel, the view would be LandingPage. This does not really fit functional programming as it relies on magic strings (yes you can do it, but making things explicit (with types) is better). Also there is no link between view models. Any view model can call any view model so the framework must be told the root view model to start with. Given that view models can be tested, it is possible to assert the right view model is called given a command, but I still wanted to make it noted that their is not inherit structure imposed on view model interaction. 

#####VM first or View first#####
As stated briefly above, Vm first means that view models request page changes and 'talk' to each other. This means that views are not tightly coupled or referenced in any way. In order for the view model to remain testable it is important that it doesn't reference a view or that view (and navigation class) would need to mocked, and most likely can't be. 

View first means that the view creates the view model. Since the view creates the view model, the view model is still free from the view and thus testable. 

#####Presenter#####
Finally most of the frameworks have some kind of presenter that 'guesses' some defaults for how the page navigation is to be done, and allows for configuration/overriding of this. The defaults that it assumes and applies are correct for the simples cases ie a stack. Put the root page inside a navigation, and then push the next page on top. The presenters also have additionally defaults for master/detail views (a menu shown via a button) which doesn't fit the simple case, and also tab views. Tab views, are particularly interesting given the prospect of navigation within the tab or full page modal removing the tab views. Som apps (Twitter) do this differently iOS and Android ie iOS navigation per tab, while Android is modal navigation only. 

##So how do the FP guys do this? ##
Elm seams to be the place where GUIs and FP is considered the 'right' way to do it. Let's see what Elm does. If you're not familiar with the [Elm Architecture](https://guide.elm-lang.org/architecture/), then check it our first. For navigation, a current page is added to the model, the message du (Discriminated union) is updated to handle page changing and consequently the update function must handle this also. Finally an extra function is added before rendering the view so the right page is rendered. If that didn't make sense (It was very brief) then get the full read [here](https://www.elm-tutorial.org/en/07-routing/cover.html) 

Given Elm is on the web, every page has a unique url so can be looked up or navigated too and then rendered. Excellent! Unfortunately, mobile (and I suspect WPF) is not the same. For mobile the view that is rendered is typically some sort of xmal (which represents the actual view hierarchy on the device), and then the page binds to that. When the page changes, the view is changed and a new set of bindings are wired up. The bindings and current page must match or the view won't display correctly (no data, and buttons won't work). This means that the actual view controls and the framework must be kept in the same state. Mobile might change the view given the lower hardware requirements (Android docs state that for memory pressure a view could killed if view is not on screen).

##What's required from a functional mvvm framework for navigation#

So now that we have a few options, let's take a look at what to include and what to leave out. 

#####include#####
- presenter -> since mobile has different types of navigation it's essential to be able to describe this 
- Modelling the pages and view to rendered, tracked in the state model
- Keep the state and view [page] in sync
    - It would be great if the framework could handle this (I'm thinking of the back button)
    - Even better it would be great if the two (view and framework) could detect if they were out of sync. (Yes this should never happen, but probably will during development)
    - alternatively it would be better if the view could make the request, and the model serves up the binding. 

#####exclude#####
- magic strings for finding pages seams like a bad idea to me. This might save some typing but at the expense of testing. 
- crazy assumptions for how to do page navigation. 
    - Given explicit wiring of pages, I would think that it would make sense to have a simple API to describe the navigation between two pages
- classes: why use a class when a function will do

#####not sure (do both; evaluate#####
- VM first/ View first (as stated below, view first might be better)
<br>

##A possible implementation##
The Elm Architecture presents what would most likely be the best solution, with some tweaks. A du (discriminated union) could be used to describe all the pages. Each page, du page and vm function could then be mapped together to provide an explicit link between page and vm. Some types would be required to describe how navigation happens. The update function can handle the single case of page navigation and pass off to the module to handle the details of the navigation. This would allow for an implementation per platform (Xamarin Forms, WPF). This just leaves the issue of the state and views, and ensuring they stay in sync. The best approach, would be for the view to request data from the state, which could then return the correct binding. This suggests the view first approach would in fact be better. Using VM first could result in out of sync model. This is possible on Android where it can kill the view if it is not on screen. When the user returns to app/page it would be a different page then the model resulting in blank view or crash. Finally the framework would also handle the setup, holding the mapping of VMs to pages and also what type of navigation would be used to start with ie navigation page, tab page etc. 

So there it is, the feature set that I'm looking for in the next mvvm framework that is functional. If you have seen another approach don't shy, comment now. Also if I missed a case or think I'm mad, leave comment about it now :)
