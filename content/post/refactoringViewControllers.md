+++
author = "kandelvijaya"
date = "2017-06-11T20:17:50+02:00"
description = "Basic approach to refactoring large/legacy view controller"
tags = ["Refactoring", "SwiftLang", "Swift", "MVC"]
title = "Refactoring MassiveViewController"

+++

# When to refactor legacy code?
Last few releases, I was working on a fairly old code. I was required to implement a small new feature in an item list controller. The controller maintains list of items in horizontal collection view. 

We were using this controller to show remote items and last seen items (local). Now the new feature was to use another API URL and then transform the received data into appropriate items. Lets say its a recommended feed item list. 

# Current Situation
![ListController](/img/ListController.png)
The list item controller was a single UIViewController, that used CollectionView, a header view and empty state view. Empty state view 
 is used  when there are no items.

### This is how the controller looks like. Dealing with everything.
![ListController Current Architecture](/img/ListControllerCurrent.png)

The single largest problem I faced during the implementation of new feature is this:

1. *ItemListViewController* was UICollectionViewDelegate, UICollectionViewDataSource, ItemContainer and conformed to couple of other component related protocols.
2. *ItemListViewController* had ~10 mutable properties, including several boolean flag to switch between remote and local data fetching mechanism.
3.  *ItemListViewController* had quite a lot of `if remoteSource { .. }else { .. }` to toggle how and where to fetch content.
4. Size calculation was fairly complex logic that was dependent on the number of items, kind of items and parent specification. We calculate the size item size manually for reasons that I won't go over here.    
5.  The file was around **600 lines** of nested logic. 

In this current setup, If I were to implement this recommended feed items then I would need to pollute the code in several places with yet another `if else recoFeed {`. More importantly, I would have to untangle the pieces of code that are interacting/mutating each other and figure out exactly several places where I have to change the source. In all, my mind need to know exactly how all the 600 lines work beforehand, in every single situation. Source of lots of future bugs and  unknown side effects. Bear in mind, that the code didn't have good test coverage. 

# How I refactor legacy/large code?
How can I reduce complexity; this is the single approach I take during refactoring. Not specific architecture or how can I make things testable. However, trying to reduce complexity leads me naturally to have more testable architecture. On the other hand, dogmatically following a architecture for the sake of it, might make a simple thing complex or overly engineered. 

This specific controller transformed along with the changing requirements over the year and now it was getting harder to add features.

Steps that I took to refactor this controller:

1.  `UICollectionViewDataSource` and `UICollectionViewDelegate` are perfect candidate for separation. In our case, the `delegate` was fairly straight forward however, `datasource` was doing remote or local content fetch based on the model controller had. 
2.  Pulled out `delegate` into separate class. 
3.  Pulled out `datasource` into separate class. However, the `datasource` was still a bit complex due to the fact that it was doing `if else ` to choose how to load the data. 
4.  `dataSource` was then separated into 2 classes: `RemoteDataSource` and `LocalDataSource`. Based on the model, the correct one was instantiated using template pattern. 
5.  Sizing information was extracted to a separate class `ItemMeasurement`.
6.  Caching logic was extracted into a small class. 

That was almost all I did incrementally. Note that i didn't use any particular architecture. I just followed the good old OOP principle. Single Responsibility. 

One more point: refactoring old untested code should be done in incremental steps. Extract a portion, manually test, if it works write unit tests retroactively to make sure our next move wont break it. Follow this approach until everything is complete.

In the end, I could just add one more dataSource for recommended items and register to the factory and be done with the new feature. I wrote unit test for `datasource`, `delegate`, measurement, sizer and cache extensively in isolation; it was straightforward. More so because every part is dealing with only 1 concern. The ListItemViewController was dumb and thats how we want it to be, it contains animation and view life-cycle management. 

### This is how the controller looks in the end.
![ListController After Refactoring](/img/ListControllerAfterRefactoring.png)

## Conclusion
- Coupled `datasource` and `delegate` in the controller, is the first place to look for refactoring. 
- For a minimal use case, this coupling is fine. When the code gets larger and conditions start appearing like mushrooms in the wild, then its a good time to isolate delegate and `datasource`. 
- Other utilities like Caching, Sizing and Animation can be either extracted to protocol extensions or small utility classes. This will be obvious once the `datasource` and `delegate` are factored out. 
- Testing then becomes easy as each piece of code is doing only 1 thing in isolation. There should be nothing besides configuration to test for in the ViewController.
- One could then use MVP / MVVM and introduce presenter or view model and abstract away `datasource`. But, do it only when it makes things simple not the other way round. 
- Be consistent. I take this approach whenever I need to refactor large controllers. This ensures the upcoming developer will get the idea once they see the pattern once. 
