---
title: "Testable Interface design with Enums"
date: 2018-03-02T12:20:27+01:00
author: "kandelvijaya"
description: "Using enums for API design"
tags: ["swift", "ios"]
draft: true
---


# Table of Contents

1.  [Writing Testable Swift API with Enums](#orgb21c5f6)
2.  [Modeling interface based on Enum](#org721b560)
3.  [Using the OneTapFeature](#org70192d6)
4.  [Testing Feature](#org8ff6392)
5.  [Refactoring the OneTapFearure to make it testable](#org5045844)
6.  [How about testing the present()](#org6f5951f)
7.  [What if our client module is using Obj-C](#org3cd2019)
8.  [Is the feature modular? Can it be resused by other apps?](#org0be6f81)
9.  [Conclusion](#org3558e4f)



<a id="orgb21c5f6"></a>

# Writing Testable Swift API with Enums

Encapsulate everything and try to show as little as possible. Greedy approach.
Works quite well in software development. Today we are going to focus on the
interface API for a fictious module: OneTapCheckout. Basic idea: when you tap
checkout button on a product/cart page, it buys the product without asking
address, payment and coupon code (given you have them already). We want to
provide this feature as a iOS framework (aka module). A simple public interface
is all we need to expose so that cart viewcontroller can make use of it.
Moreover product page is also interested to integrate this feature already. So
lets get to code.  

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-left" />

<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">Client's view</th>
<th scope="col" class="org-left">Interface API</th>
<th scope="col" class="org-left">Internal Details</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left">just 2 methods</td>
<td class="org-left">present()</td>
<td class="org-left">100s of data structure and objects interacting</td>
</tr>


<tr>
<td class="org-left">&#xa0;</td>
<td class="org-left">isEnabled</td>
<td class="org-left">&#xa0;</td>
</tr>
</tbody>
</table>


<a id="org721b560"></a>

# Modeling interface based on Enum

And this is how we could have implemented the OneTapFeature. This makes for a concise and readable API.
```swift
    public enum OneTapFeature {
    
        case cart
        case product(withId: String)
    
        var isEnabled: Bool {
            switch self {
            case .cart:
                return FeatureToggle.isOneTapEnabledInCart
            case .product:
                return FeatureToggle.isOneTapEnabledInProductPage
            }
        }
    
        func present() {
            guard isEnabled else { return }
            let controller = OneTapController()
            switch self {
            case .cart:
                controller.stylizeForCart()
            case .product:
                controller.stylizeForCart()
            }
            AppNavigator.shared.navigate(to: .oneTapCheckout)
        }
    
    }
```

<a id="org70192d6"></a>

# Using the OneTapFeature

The above enum based API is nice and declarative. This is the only public facing
entry to the feature/module. It only lets you do 2 things. First check if
feature is enabled from either cart or product page. Second present the feature.
Lets see how they can be used by client.
```swift
    class CartViewController: UIViewController {
        @IBAction func checkout() {
            if OneTapFeature.cart.isEnabled {
                OneTapFeature.cart.present()
            } else {
                // old path
            }
        }
    }
```
And for the product page, we have 2 choice. We want to show OneTap button when the feature is enabled/ready. 
If not, we will show the old checkout button. 
```swift
    class ProductViewController: UIViewController {
    
        @IBOutlet var oneTapButton: UIButton?
        @IBOutlet var checkoutButton: UIButton?
        var productId: String = ""
    
        override func viewDidLoad() {
            super.viewDidLoad()
            let feature = OneTapFeature.product(withId: productId)
            if feature.isEnabled {
                oneTapButton?.isHidden = false
                checkoutButton?.isHidden = true
            } else {
                oneTapButton?.isHidden = true
                checkoutButton?.isHidden = false
            }
        }
    
    }
```
\`isEnabled\` checks weather feature is ready for the asking entry point. The
feature could be AB test value, internal feature toggle or a feature toggle
strategy to push code but not display until everything is ready. It uses
\`AppFeature\` to get this information, which can be accessed anywhere from the
app. To read more on feature toggle and strategy [Martin Fowler's blog](https://martinfowler.com/articles/feature-toggles.html)  

Lets move on! Although recently I have started TDD with test first, I don't
expect everyone to do the same. I can only recommend. Had we done TDD then we
could see the pitfalls of the above approach already. However, like usual, lets
now finally write retroactive tests so there wont be breaking changes in the
feature. If there will, CI will detect.


<a id="org8ff6392"></a>

# Testing Feature

Lets think how can we test this piece of interface. 

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-right" />

<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<tbody>
<tr>
<td class="org-right">S.No.</td>
<td class="org-left">Test Case</td>
<td class="org-left">Assertion</td>
</tr>


<tr>
<td class="org-right">1</td>
<td class="org-left">when cart feature is off</td>
<td class="org-left">cart isEnabled returns false</td>
</tr>


<tr>
<td class="org-right">2</td>
<td class="org-left">when product feature is off</td>
<td class="org-left">product isEnabled returns false</td>
</tr>


<tr>
<td class="org-right">3</td>
<td class="org-left">when feature is disabled</td>
<td class="org-left">app doesnot navigate to oneTapVC</td>
</tr>


<tr>
<td class="org-right">4</td>
<td class="org-left">when feature is enabled</td>
<td class="org-left">app does navigate to oneTapVC</td>
</tr>
</tbody>
</table>

We will write unit tests for 2 cases here (1) and (3), others are left as
exercise On the second case (3) we will need to rethink how we test. (Lets hold
on to this thought until we see a counter argument).  

Test Case 1: 
```swift
    class OneTapFeatureTests: XCTestCase {
    
        func test_whenCartFeatureToggleIsOff_thenCartOneTapIsDisabled() {
            FeatureToggle.setOneTapEnabled(false)
            XCTAssertFalse(OneTapFeature.cart.isEnabled)
        }
    
    }
```
There is a problem. We know that the \`isEnabled\` is going to use
OneTapCartFeatureToggle internally. Unit testing is suppoed to be a black box
test. Meaning, you provide a input and test for output. ****NOT****, how it computes
and what it uses to compute the output. That is white box testing.  

I mentioned unit testing should be black box testing, this can be debatable.
However, my perspective here is a method is a unit. You are responsible to
define what a unit means to your specific need. It could be a entire class or a
method or a couple of methods. The point of unit testing is that you treat that
unit you defined as a black box. Given a input, you test for output. Not side
effects. This however can be debated to be white box testing if you think your
app is a unit and your classes are internal pieces. I don't recommend this
thinking. I believe in divide and conquer, recursively. A function is a smallest
unit. A class is merely collection of functions (collection of smaller units =
unit). A program is composition of classes (collection of smaller units ==
unit). Lets not debate here! Define your unit.  

The above approach is not only bad principally but also practically. Lets say
sometime later, \`isEnabled\` is refactored to be enabled only in Germany. If you
were running the test on german locale app, then the above would pass. I work in
Berlin. So if this was my test, it would pass. I wouldn't spot a difference.
****Dont test the internal implementation. Unit test is strictly tesing output
from the input.****  

How can we write proper test here? Tough! We need to get rid of \`enum\`. Sad
but true. Enums don't allow you to inject dependencies. They don't have
constructor. If this were a class/struct, we could use constructor injection to
provide the feature toggles. Lets do that up next.


<a id="org5045844"></a>

# Refactoring the OneTapFearure to make it testable
```swift
    struct OneTapCheckoutFeature {
    
        enum OneTapEntryPoint {
            case cart
            case product(withId: String)
        }
    
        private let entryPoint: OneTapEntryPoint
        private let isCartToggleEnabled: Bool
        private let isProductToggleEnabled: Bool
        private let navigator: Navigator
    
        // note the constructor injection and sensible default argument
        init(from: OneTapEntryPoint,
             isCartToggleEnabled: Bool = FeatureToggle.isOneTapEnabledInCart,
             isProductToggleEnabled: Bool = FeatureToggle.isOneTapEnabledInProductPage,
             withNavigator navigator: Navigator = AppNavigator.shared) {
            self.entryPoint = from
            self.isCartToggleEnabled = isCartToggleEnabled
            self.isProductToggleEnabled = isProductToggleEnabled
            self.navigator = navigator
        }
    
        var isEnabled: Bool {
            switch entryPoint {
            case .cart:
                return isCartToggleEnabled
            case .product:
                return isProductToggleEnabled
            }
        }
    
        func present() {
            guard isEnabled else { return }
            let controller = OneTapController()
            switch entryPoint {
            case .cart:
                controller.stylizeForCart()
            case .product:
                controller.stylizeForCart()
            }
            navigator.navigate(to: .oneTapCheckout)
        }
    
    }
```
Now lets revisit the test case.
```swift
    class OneTapFeatureTests: XCTestCase {
    
        func test_whenCartFeatureToggleIsOff_thenCartOneTapIsDisabled() {
            let feature = OneTapFeature(from: .cart, isCartToggleEnabled: false)
            XCTAssertFalse(feature.isEnabled)
        }
    
    }
```
We now can control the input and hence test for the output. This is future proof
too: if I were to refactor the \`isEnabled\` to use country locale after few
months, I would change the constructor and my CI would notify me as the tests
won't compile. Not compiling tests is failed test too.  

Notice that the default parameter uses the FeatureToggle right in the function
declaration. This allows the calling client to not worry about passing in these
values. They can omit them. It will sensibly default. In tests however, we will
swap them. Nice right!


<a id="org6f5951f"></a>

# How about testing the present()

We can now follow the same principle and in constructor argument inject the
AppNavigator type. Since AppNavigator can be modeled as protocol this can be
mocked in test independently.
```swift
    class OneTapFeatureTests: XCTestCase {
    
        func test_whenProductFeatureIsNotEnabled_thenPresentDoesNotNavigate() {
            let mockNavigator = MockNavigator()
            let feature = OneTapCheckoutFeature(from: .product(withId: UUID().uuidString),
                                                isProductToggleEnabled: false,
                                                withNavigator: mockNavigator)
            XCTAssertFalse(feature.isEnabled)
            XCTAssertNil(mockNavigator.lastNavigatePoint)
        }
    
        class MockNavigator: Navigator {
            var lastNavigatePoint: NavigationPoint?
            func navigate(to: NavigationPoint) {
                self.lastNavigatePoint = to
            }
        }
    
    }
```
As we have used the constructor injection with default parameter for navigator,
client implementation of production code can be as is. They dont need to pass in
Navigator type. It will sensibly default. For the test, we have an oppurtunity
to use our mocked Navigator. This helps us check for side effects (some call it
behavior). I usually prefer to nest the mock type declaration inside the
namespace of the test case. This avoids leaking mocks outside. However if there
are other tests that need the same mock then create a factory to generate the
mocks. Other important thing to note is, \`.product(with: UUID().uuidString)\`
rather than using \`.product(with: "someId")\`. I leanred form [@Paul](https://twitter.com/pardel) to not use
literal as values in test.  

However, although all looks fine. We are certainly not following proper Unit
Testing rule. We said its supposed to be black box test. In the \`present()\`
test, we are not testing for the output directly. \`present()\` doesnot return
anything back. We are testing outcome of calling present. The outcome is calling
into the navigtor object. Its a side effect. This is very common sight in Object
Oriented Programming but certainly could be improved. We will see how to do so
in the follow up post!


<a id="org3cd2019"></a>

# What if our client module is using Obj-C

As much as I like to use swift everywhere, not every app is swift only. So what
if the client module is Obj-C? It can't represent Enum with associated value. It
thereby cannot initialize OneTapCheckoutFeature as its public initializer uses
enum which cannot be represented in Obj-C.  

There are couple of strategy to solve this issue. I wont muster the details here
but present a solution I resort most of the times. Its by wrapping the enum into
a class. All the cases would in turn be static functions so that the API still
feels the same. Lets see it in action.
```swift
    public final class OneTapEntryPoint: NSObject {
    
        enum _OneTapEntryPoint {
            case cart
            case product(withId: String)
        }
    
        internal let value: _OneTapEntryPoint
    
        internal init(_ point: _OneTapEntryPoint) {
            value = point
        }
    
        @objc public static var cart: OneTapEntryPoint {
            return OneTapEntryPoint(.cart)
        }
    
        @objc public static func product(withId: String) -> OneTapEntryPoint {
            return OneTapEntryPoint(.product(withId: withId))
        }
    
    }
```
There are minimal changes to be made in the actual OneTapCheckoutFeature type.

-   \`convenience init\` just exposes the entryPoint to be passed. Others will default.
    Its good when the main initializer uses types that cannot be represented in Obj-C but 
    have default argument.
-   both \`isEnabled\` and \`present()\` are exposed by \`@objc\` and \`public\`
-   on the switch, we now switch on the internal \`value\` (the backing storage swifty enum)
```swift
    public final class OneTapCheckoutFeature: NSObject {
    
        // instance members as is
    
        // main initializer as is
    
        @objc convenience public init(from: OneTapEntryPoint) {
            self.init(from: from)
        }
    
        @objc public var isEnabled: Bool {
            switch entryPoint.value {
            // ..
        }
    
        @objc public func present() {
            // ..
            switch entryPoint.value {
            // ..
        }
    
    }
```
Then the Obj-C class can call the module very similar to Swifty Enum like this:
```obj-c
    /// Obj-C usage
    
    // very similar to Enum
    OneTapEntryPoint* cart = OneTapEntryPoint.cart;                          
    
    // all possible with static method
    OneTapEntryPoint* capItem = [OneTapEntryPoint productWithId: "CapId"];   
    
    
    /// Obj-C usage for OneTapCheckoutFeature `init(from:)`
    OneTapCheckoutFeature* feature = 
        [[OneTapCheckoutFeature alloc] initFrom: OneTapEntryPoint.cart];   
    if (feature.isEnabled) {
        [feature present];
    }
```
Other approaches that I'm aware of is either by extending the enum type and
creating factory method. Or creating a class that waraps each enum case. I
however believe this is very neat way to expose swift enum based api to obj-c.
Note that, Obj-C can only create the OnePointEntryPoint. Given two instances of
different OneTapEntryPoint, Obj-C doesnot have the ability to read what case a
instance has internally. You can still do memory comparison.


<a id="org0be6f81"></a>

# Is the feature modular? Can it be resused by other apps?

Not at all. Please don't bother with the details of this section if you dont
want to share this module with other teams in your company. We cannot
share this to outsiders because the checkout is natively domain logic of the
company. Lets say there is another app within the same business unit wanting to
integrate this amazing feature. We need to give them a swift framework. We have
obvious problem as our \`OneTapCheckoutFeature\` module knows the app environment.
It is using \`AppNavigator\` and \`FeatureToggle\` within the module interface
constructor default argument. What if the other app doesnot have this types?
There is a very good chance they wont. How can we inverse this dependency?  

Thank you [@Hrvoje](https://twitter.com/hstanisic) for critically evaluating and suggesting this point.  

There are couple of ways to get around this issue. Using delegate, passing in
configuration object, using dependency container and so on. I will focus on the
classic delegate pattern to tackle this issue. We can go into the others in the
follow up post, if needed.  

****Goal here is to remove AppNavigator and FeatureToggle from
OneTapCheckoutFeature****. I don't like to enforce embedding app to use 
specific navigation pattern or feature toggle strategy.  

Enter classic delegate pattern:
```swift
    protocol OneTapCheckoutFeatureDelegate: class {
        func isEnabled(from: OneTapEntryPoint) -> Bool
        func present(controller: UIViewController)
    }
```
We are asking 2 questions:

1.  Is the feature ready for the calling entry poitn? Thus getting rid of
    knowledge of what means a feature ready in the module.
2.  present the configured controller? This gets rid of the responsibility of
    navigation logic from the module. The embedding app can uses its own pattern.

The interface in full. 
```swift
    public final class OneTapCheckoutFeature: NSObject {
    
        private let entryPoint: OneTapEntryPoint
        private let delegate: OneTapCheckoutFeatureDelegate  // weak is left out
    
        @objc init(from entryPoint: OneTapEntryPoint,
             delegate: OneTapCheckoutFeatureDelegate) {
            self.entryPoint = entryPoint
            self.delegate = delegate
        }
    
        @objc public var isEnabled: Bool {
            return delegate.isEnabled(from: entryPoint)
        }
    
        @objc public func present() {
            guard isEnabled else { return }
            let controller = OneTapController()
            switch entryPoint.value {
            case .cart:
                controller.stylizeForCart()
            case .product:
                controller.stylizeForCart()
            }
            delegate.present(controller: controller)
        }
    
    }
```
Lets see the tests and make sure we can test our interface after all.
```swift
    class OneTapFeatureTests: XCTestCase {
    
        func test_whenProductFeatureIsNotEnabled_thenPresentDoesNotNavigate() {
            let delegate = MockDelegate(productEnabled: false)
            let feature = OneTapCheckoutFeature(from: .product(withId: UUID().uuidString),
                                                delegate: delegate)
            XCTAssertFalse(feature.isEnabled)
            XCTAssertFalse(delegate.presentCalled)
        }
    
        // internal namespacing for mocks
        class MockDelegate: OneTapCheckoutFeatureDelegate {
    
             // instance members 
    
            init(cartEnabled: Bool = true, productEnabled: Bool = true) {
                 //assignments
            }
    
            func isEnabled(from: OneTapEntryPoint) -> Bool {
                switch from.value {
                case .cart:
                    return cartEnabled
                case .product:
                    return productEnabled
                }
            }
    
            func present(controller: UIViewController) {
                presentCalled = true
            }
        }
    
    }
    
    // When in playgrounds
    OneTapFeatureTests.defaultTestSuite.run()
```
When the client is Obj-C, one can mark the protocol \`@objc\` and they are good to
go. The client can supply however he computes \`isEnabled\` and handle
presentation in its own way. Maybe with segue, custom presentation animation,
coordinator pattern. Therby we removed all the app logic from the
OneTapExpressCheckout module. Meaning we can now reuse it with other app.


<a id="org3558e4f"></a>

# Conclusion

Enum are powerful type. Swift is awesome; it features enum with associated
values. This allows for neat api design with powerful pattern matching ability.
However it does have few corner cases especially on testability and objc
interoperability.  

We saw that for unit test to be practical we need to be able to control input
and assert on the output. Enums don't allow arbitary argument via constructor
injection nor property injection (I dont prefer this mutation). 
********Thereby methods on enum that depend on anything other than the enum cases immediately
makes it hard to test.********  

Second, in case we use enum based api then we might run into problems when
exposing to Obj-c caller. In this instance, we mimicked enum type with class
types where each case is modelled to a static member or function.  

Third, We saw the limitation of default argument in constructor injection. It
was stopping us from making reusable module. we went step ahead and made the 
module reusable by inversing dependencies it needed with delegate pattern.   

We left 1 point to disucuss on further follow up post.

1.  Why testing for side effect is a bad practice? How can we improvise it?

Thank you for reading this post. If you learned something, consider spreading
the knowledge. If you have some suggestion, improvement or comments please leave
a comment below or DM me on twitter at [@kandelvijaya](https://www.twitter.com/kandelvijaya). Enjoy!

