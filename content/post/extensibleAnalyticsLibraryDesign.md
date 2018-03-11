---
title: "Extensible Analytics Library Design with final tag-less concept"
date: 2018-03-11
author: "kandelvijaya"
description: "Testing new concept with analytics system library"
tags: ["swift", "iOS", "architecture", "analytics", "expression-problem"]
draft: false
---


# Table of Contents

1.  [Keywords](#orgac401b9)
    1.  [Expression Problem](#org782b390)
2.  [Introduction](#orge44e061)
    1.  [Following the ritual](#orgfb4805d)
    2.  [Not following the ritual](#org2f2d1a2)
    3.  [Preface](#org5b5460a)
3.  [2 classical ways to implement analytics library](#orgc6e418e)
    1.  [Enum based (Sum type based)](#org60bf77b)
    2.  [Class, struct+protocol based (product type based)](#orgc0c2b41)
4.  [Client problem: Add ForceTapEvent & support another analytics service](#org3fa5d7d)
    1.  [The limitation of enum approach](#org7ade7c5)
    2.  [The limitation of class/(struct+protocol) based approach](#orgcb8e292)
    3.  [Summarizing](#org767e732)
5.  [A radical way to solve the same issue aka Tagless final](#orga3f5a8e)
    1.  [Lets get both: ability to add cases and new interpretation.](#org65533bb)
    2.  [Client: Adding force tap event](#orgb65dbdd)
    3.  [Client: Adding support for FirebaseTracking](#orgb0c20cb)
    4.  [Summarizing:](#org989494f)
6.  [Getting a step ahead](#org99eeb59)
    1.  [Adding new event which can aggregate other event](#orgcf8442d)
    2.  [Adding a new drawing log backend](#org6cc0eb7)
    3.  [Using the new backend](#orgbbc7e8c)
    4.  [the outcome](#orgca14703)
7.  [Conclusion](#orgf3f0b1d)



<a id="orgac401b9"></a>

# Keywords

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">Keywords</th>
<th scope="col" class="org-left">Description</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left">Tagless final</td>
<td class="org-left">An approach to solve expression problem. [Read more here](http://okmij.org/ftp/tagless-final/index.html)</td>
</tr>


<tr>
<td class="org-left">Open Closed Principle</td>
<td class="org-left">Open for extension, closed for modification.</td>
</tr>


<tr>
<td class="org-left">Library</td>
<td class="org-left">A module, framework or static library (in context of iOS development)</td>
</tr>


<tr>
<td class="org-left">Analytics Event</td>
<td class="org-left">A type representing a state that needs to be tracked.</td>
</tr>
</tbody>
</table>


<a id="org782b390"></a>

## Expression Problem

The idea is that your program is a combination of a datatype and operations over it. 
The problem asks for an implementation that allows to add new cases of the type and 
new operations without the need for recompilation of the old modules and keeping static 
type safety(no casts or runtime type checks).  [View the full discussion on SO](https://stackoverflow.com/questions/3596366/what-is-the-expression-problem)

<a id="orge44e061"></a>

# Introduction

I can start this blog in 2 different ways. And I will. 


<a id="orgfb4805d"></a>

## Following the ritual

Few months back, articles emerged trying to implement analytics in various ways. 
This is a response to [Soroush Khanlou's article](http://khanlou.com/2017/12/misusing-subclassing/) which is itself a response to [Dave DeLong's article](https://davedelong.com/blog/2017/12/07/misusing-enums/), 
which is itself a response to [Matt Diephouse's article](http://matt.diephouse.com/2017/12/when-not-to-use-an-enum/), which is itself a response to [John Sundell's article](https://www.swiftbysundell.com/posts/building-an-enum-based-analytics-system-in-swift). 


<a id="org2f2d1a2"></a>

## Not following the ritual

Today we are going to look at how can we write a analytics library that can be extended but not modified. Remember, the 
open close principle. Our library must be open for extension but closed for modification. We will briefly walk 
through different approaches iOS devs have put before me and evaluate those under 2 points from outside the 
library. I will use analytics as an example. There is a wonderful talk by [obj.io guys and Brandon Kase](https://talk.objc.io/episodes/S01E89-extensible-libraries-2-protocol-composition) which uses 
drawing instead of analytics. That talk was the inspiration for me to tackle analytics problem in similar realm.  

1.  ****Can I add new type?****
2.  ****Can I add new operation?**** (Change what it means to be that type.)

<span class="underline">Both FireAnalyticsService and GoogleAnalyticsService are example types not the real library interface.</span>


<a id="org5b5460a"></a>

## Preface

The ground is how can we write a extensible and type safe api/library for analytics tracking system. Although 
I will present yet another way of looking into the same problem with different idea, I, in no means, conclude 
the idea that I present is the best one. Lots of smart people have come forth and wrote, debated and 
talked on this matter. I will like to muster my thoughts here too. One thing that I can promise is if you stick 
with me; you will be rewarded with aha moment towards the end of this post. We will use analytics events to draw 
(yeah draw, you got it) events diagram.


<a id="orgc6e418e"></a>

# 2 classical ways to implement analytics library


<a id="org60bf77b"></a>

## Enum based (Sum type based)
```swift
    // library code
    enum Event {
        case tap(buttonId: String, onScreen: String)
        case opened(page: String, user: User)
    }
    
    extension Event {
    
        func send(usingService service: GoogleAnalyticsService) {
            switch  self {
            case let .tap(buttonId: id, onScreen: screenName):
                service.sendEvent(named: "click", 
                                  userInfo: [:], 
                                  metadata: ["screenname": screenName, 
                                             "buttonId": id])
            case let .opened(page: page, user: user):
                service.sendEvent(named: "view", 
                                  userInfo: [user.id: user.info], 
                                  metadata: ["screenname": page])
            }
        }
    
    }
```

<a id="orgc0c2b41"></a>

## Class, struct+protocol based (product type based)

Although, below code shows class subclassing approach, one can emulate similar result with protocol and struct. 
```swift
    // library code
    open class EventC {
    
        public init() {}
    
        func send(usingService service: GoogleAnalyticsService) {
            guard let payload = payload() else { return }
            service.sendEvent(payload: payload)
        }
    
        open func payload() -> EventPayload? {
            // meant for subclassing
            return nil
        }
    
    }
    
    
    class TapEvent: EventC {
    
        private let buttonId: String
        private let screenName: String
    
        public init(buttonId: String, screenName: String) {
           self.buttonId = buttonId
        self.screenName = screenName
        }
    
        override func payload() -> EventPayload? {
            return EventPayload(name: "click", 
                                userInfo: [:], 
                                metadata: ["screenname": screenName])
        }
    
    }
    
    
    final class ViewEvent: EventC {
    
        private let page: String
        private let user: User
    
        public init(page: String, user: User) {
            self.page = page
            self.user = user
        }
    
        override func payload() -> EventPayload? {
            return EventPayload(name: "view", 
                                userInfo: [user.id: user.info], 
                                metadata: ["screenname": page])
        }
    
    }
```

<a id="org3fa5d7d"></a>

# Client problem: Add ForceTapEvent & support another analytics service


<a id="org7ade7c5"></a>

## The limitation of enum approach

-   Its easy to add new extension from outside the library. Say we want to track the events with
    firebase analytics system. It differs from GoogleAnalytics is info key and some properties.

```swift
    final class FireAnalayticsService {
        func send(event: String, metadata: [String: String]) {}
    }
    
    
    extension Event{
    
        func send(usingService service: FireAnalayticsService) {
            switch self {
            case let .opened(page: screen, user: user):
                service.send(event: "fire-event-view", 
                             metadata: ["customer-name": user.name, 
                                       "fire-screen-name": screen])
            case let .tap(buttonId: id, onScreen: screen):
                service.send(event: "fire-event-click", 
                             metadata: ["fire-screen-name": screen, 
                             "fire-button-id": id])
            }
        }
    
    }
```

-   Its not possible to add new case (new event type) from outside of the library/module.


<a id="orgcb8e292"></a>

## The limitation of class/(struct+protocol) based approach

-   Its easy to add new case/event type wither by subclassing or new struct conforming to the type. We will stick 
    with subclassing here.

```swift
    final class ForceTapEvent: TapEvent {
    
        private let buttonId: String
        private let screenName: String
    
        override init(buttonId: String, screenName: String) {
            self.buttonId = buttonId
            self.screenName = screenName
            super.init(buttonId: buttonId, screenName: screenName)
        }
    
        override func payload() -> EventPayload? {
            return EventPayload(name: "click-force", 
                                userInfo: [:], 
                                metadata: ["screenname": screenName])
        }
    
    }
```

-   Its hard to use change what it means to be a tracking event. In this case, its hard to change to a 
    firebase analytics system. At first, it seems like its doable. It might be in this case. But this is a contrived example. 
    In this scenario, you can put a extension on super class with \`send(usingService: FireAnalyticsService)\` and
    add a method to return \`FirePayload\`. Then for each subtype, without the compiler helping, override the 
    2 methods. Lots of work. What if the the overridden method is a customization hook of a internal method on 
    superclass. What if the super class maintains specific state for GoogleAlanlytis purpose which makes no 
    sense for other analytics system.


<a id="org767e732"></a>

## Summarizing

1.  Enums are great if the cases/types are fixed and will never have to change. Great examples are Optional, 
    Result. You can easily use enums to intrepret different things.
2.  Class/struct are great if all types/cases cannot be defined upfront. They are however don't allow to change the 
    intrepretation of what it means to be a subtype.


<a id="orga3f5a8e"></a>

# A radical way to solve the same issue aka Tagless final

I suggest reading the keyword section for what a tagless final is before proceeding. 


<a id="org65533bb"></a>

## Lets get both: ability to add cases and new interpretation.

As the app grows, you will need to track different cases. As time goes by, you might want to track things 
differently or use different backend services. Above mentioned enum vs class based library allowed you to pick one. We 
now will allow the client to choose both benefits. 

```swift
    // library code
    protocol EventProtocol {
        static func tap(buttonId: String, onScreen: String) -> Self
        static func opened(page: String, user: User) -> Self
    }
    
    public struct GoogleEvent {
        public var send: (GoogleAnalyticsService) -> Void
    }
    
    
    extension GoogleEvent: EventProtocol {
        public static func tap(buttonId: String, onScreen: String) -> GoogleEvent {
            let payload = EventPayload(name: "click", 
                                       userInfo: [:], 
                                       metadata: ["screenname": onScreen, 
                                                   "buttonId": buttonId])
            return GoogleEvent { transporter in
                transporter.sendEvent(payload: payload)
            }
        }
    
        public static func opened(page: String, user: User) -> GoogleEvent {
            let payload = EventPayload(name: "view", 
                                       userInfo: [user.id: user.info], 
                                       metadata: ["screenname": page])
            return GoogleEvent { transporter in
                transporter.sendEvent(payload: payload)
            }
        }
    }
    
    // library code ends here
```

By default, we are using GoogleAnalytics specific interpretation. And we have provided client with only two types 
of tracking; event and view. Pretty limited. We now have given responsibility of extension to the client. Every code 
below from here is a client side code. 


<a id="orgb65dbdd"></a>

## Client: Adding force tap event
```swift
    protocol ForceTapEventProtocol {
        static func forceTap(buttonId: String, onScreen: String) -> Self
    }
    
    extension GoogleEvent: ForceTapEventProtocol {
        static func forceTap(buttonId: String, onScreen: String) -> GoogleEvent {
            let payload = EventPayload(name: "force-click",
                                       userInfo: [:], 
                                       metadata: ["screenname": onScreen, 
                                                  "buttonId": buttonId])
            return GoogleEvent { transporter in
                transporter.sendEvent(payload: payload)
            }
        }
    }
```

We have effectively extended the library from outside to support new type/case. This is what enums lacked. 


<a id="orgb0c20cb"></a>

## Client: Adding support for FirebaseTracking
```swift
    struct FirebaseEvent {
        var send: (FireAnalayticsService) -> Void
    }
    
    extension FirebaseEvent: EventProtocol {
    
        static func tap(buttonId: String, onScreen: String) -> FirebaseEvent {
            return FirebaseEvent { transporter in
                transporter.send(event: "fire-event-click", 
                                 metadata: ["fire-screen-name": onScreen, 
                                            "fire-button-id": buttonId])
            }
        }
    
        static func opened(page: String, user: User) -> FirebaseEvent {
            return FirebaseEvent { transporter in
                transporter.send(event: "fire-event-view", 
                                 metadata: ["customer-name": user.name, 
                                            "fire-screen-name": page])
            }
        }
    }
```

We have effectively extended to interpret event as firebase event . A event by default could 
interpreted for GoogleAnalytics. Now it can also be interpreted for FirebaseAnalytics. This is what 
class (product types) lacked. 


<a id="org989494f"></a>

## Summarizing:

1.  This is a cleaver way to model the cases. We used protocol which will return Self. The library doesn't care who 
    Self is.
2.  ForceTapEventProtocol has only GoogleAnalytics Interpretation. When we use it with firebase, the compiler will 
    stop compiling to tell us that it doesnot support it. This is good. We then can either choose to extend this 
    event or discard it.


<a id="org99eeb59"></a>

# Getting a step ahead

Now the fun kicks in. Lets write a backend for our event such that each view event is represented a purple rectangle, 
tap event is represented as red circle. We will then sequence all the events and draw a nice diagram with all events 
that happened with time. 


<a id="orgcf8442d"></a>

## Adding new event which can aggregate other event

```swift
    protocol AggregateEventProtocol: EventProtocol {
        static func aggregate(_ e1: Self, _ e2: Self) -> Self
    }
```

<a id="org6cc0eb7"></a>

## Adding a new drawing log backend

```swift
    struct EventLogger {
        let view: EventLogView
        let frame: CGRect
    }
    
    extension EventLogger: AggregateEventProtocol {
        static func opened(page: String, user: User) -> EventLogger {
            let rect = CGRect(x: 0, y: 0, width: 60, height: 60)
            let view = EventLogView(frame: rect)
            view.event = "open"
            view.screen = page
            view.user = user
            view.backgroundColor = .purple
    
            return EventLogger(view: view, frame: rect)
        }
    
        static func tap(buttonId: String, onScreen: String) -> EventLogger {
            let rect = CGRect(x: 0, y: 0, width: 60, height: 60)
            let view = EventLogView(frame: rect)
            view.layer.cornerRadius = rect.width / 2
            view.event = "tap"
            view.screen = onScreen
            view.user = nil
            view.backgroundColor = .red
            return EventLogger(view: view, frame: rect)
        }
    
        static func aggregate(_ e1: EventLogger, _ e2: EventLogger) -> EventLogger {
            //.....
            let view = EventLogView(frame: viewRect)
            view.addSubview(previousView)
            view.addSubview(lineview)
            view.addSubview(nextView)
    
            return EventLogger(view: view, frame: viewRect)
        }
    }
```

<a id="orgbbc7e8c"></a>

## Using the new backend

```swift
    let logs: GoogleEvent = aggregated([
        .opened(page: "homePage", user: appUser),
        .opened(page: "detailPage", user: appUser),
        .tap(buttonId: "checkout", onScreen: "v"),
        .opened(page: "successPage", user: appUser)
        ])
    
    let logsToDraw: EventLogger = aggregated([
        .opened(page: "homePage", user: appUser),
        .opened(page: "detailPage", user: appUser),
        .tap(buttonId: "checkout", onScreen: "v"),
        .opened(page: "successPage", user: appUser)
        ])
    
    /// Event are google trackable
    logs.send(GoogleAnalyticsService())
    
    /// Events can be rendered to UIView
    extension EventLogger {
        func logDiagram() -> UIView {
            let parentview = UIView(frame: self.frame)
            parentview.backgroundColor = .white
            parentview.addSubview(self.view)
            return parentview
        }
    }
    let eventDiagram = logsToDraw.logDiagram()
    
    import PlaygroundSupport
    PlaygroundPage.current.liveView = eventDiagram
```

<a id="orgca14703"></a>

## the outcome
![Outcome](/img/finalTaglessAnalyticsOutput.png)

Here you can appreciate the same events, are type inferred without any extra transformation to 2 different 
services/backends. One is the GoogleAnalytics tracker. Other is a event diagram logger. Since, Swift 
doesn't allow incomplete type (Protocol with Self ) to be used as type we had to duplicate the events. 


<a id="orgf3f0b1d"></a>

# Conclusion

We saw yet another way to write analytics library. However the intention here is how can we create extensible 
library by allowing client to do 1) add new types/cases  2) alter what a type means. All from outside the 
library code.  

I highly recommend watching this amazing talk by [Brandon Kase on finally solving the expression problem](https://www.dotconferences.com/2018/01/brandon-kase-finally-solving-the-expression-problem) and 
wonderful in-depth talk at [objc.io](https://talk.objc.io/episodes/S01E89-extensible-libraries-2-protocol-composition) to grasp the concept fully. I know I didn't really outline how things work 
in this post. It is intentional and the topic itself spans couple of posts. However, I believe you now saw yet 
another approach and I have linked in the resource if you want to explore this territory. 

I hope you saw something new in this post. I am excited to see more areas to apply the final tagless approach in 
the upcoming days. If you know some specific use case this approach would help, I would love to hear about it. 
Happy weekend. Happy coding!

