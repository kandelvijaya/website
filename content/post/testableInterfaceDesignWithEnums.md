---
title: "Testable Interface design with Enums"
date: 2018-03-02T12:20:27+01:00
author: "kandelvijaya"
description: "Using enums for API design"
tags: ["swift", "ios"]
draft: true
---

# Table of Contents

1.  [Writing Testable Swift API with Enums](#org3b22e12)
2.  [Modeling interface based on Enum](#org8feb29f)
3.  [Using the OneTapFeature](#org7bc4b44)
4.  [Testing Feature](#orgafdc04b)
5.  [Refactoring the OneTapFearure to make it testable](#orgfcff107)
6.  [How about testing the present()](#org6cc2c5a)
7.  [What if our client module is using Obj-C](#orga963038)
8.  [Conclusion](#org13ee52f)



<a id="org3b22e12"></a>

# Writing Testable Swift API with Enums

Encapsulate everything and try to show nothing. Greedy approach.  Works quite well in software 
development. Today we are going to focus on the interface API for a fictious module: OneTapCheckout.
Basic idea: when you tap checkout button on a product/cart page, it buys the product without asking 
address, payment and coupon code (given you have them already). We want to provide this feature as 
a iOS framework (aka module). A easy Public interface is all we need to expose so that cart viewcontroller 
can make use of it. Moreover product page is also interested to integrate this feature already. So lets get to 
code.  

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


<a id="org8feb29f"></a>

# Modeling interface based on Enum

And this is how we could have implemented the OneTapFeature. This makes for a concise and readable API.

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


<a id="org7bc4b44"></a>

# Using the OneTapFeature

The above enum based API is very nice and declarative. This is the only public facing entry to the 
feature. The only 2 things it lets you do is ask if the feature is enabled from either cart or product 
page and second present the feature. Lets see how they can be used by client.

    class CartViewController: UIViewController {
        @IBAction func checkout() {
            if OneTapFeature.cart.isEnabled {
                OneTapFeature.cart.present()
            } else {
                // old path
            }
        }
    }

And for the product page, we want to only present oneTap button when it is enabled. 

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

Important thing to note about the \`isEnabled\` checks weather feature is ready for the asking entry point. The feature
could be AB testing value, internal feature toggle or a feature toggle strategy to push code but not display until everything is ready. It 
uses \`AppFeature\` to get this information, which can be accessed anywhere from the app. To read more on feature toggle and 
strategy [read indepth on Martin Fowler's blog.](https://martinfowler.com/articles/feature-toggles.html) Lets move on! Although recently I have started TDD with test first, I dont 
expect everyone is doing it. I recommend. Had we done TDD then we could see the pitfalls of the above approach already. However, 
like usual lets now finally write retroactive tests so there wont be breaking changes in the feature. If there will, CI will detect.  


<a id="orgafdc04b"></a>

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

We will write unit tests for 2 cases here (1) and (3), others is left as exercise
On the second case (3) we will need to rethingk how we test. (Lets hold on to this thought until 
we see a counter argument and then we will improvise).

Test Case 1: 

    class OneTapFeatureTests: XCTestCase {
    
        func test_whenCartFeatureToggleIsOff_thenCartOneTapIsDisabled() {
            FeatureToggle.setOneTapEnabled(false)
            XCTAssertFalse(OneTapFeature.cart.isEnabled)
        }
    
    }

There is a problem. We know that the \`isEnabled\` is going to use OneTapCartFeatureToggle internally. 
Unit testing is suppoed to be a black box test. Meaning, you provide a input and test for output. 
NOT, how it computes and what it uses to compute the output. That is white box testing.  

The above approach is not only bad principally but also practically. Lets say sometime later, \`isEnabled\` is refactored to be enabled only 
in Germany. If you were running the test on german locale app, then the above would pass. I work in Berlin. 
So if this was my test, it would pass. I wouldn't spot a difference. Now I bet you spotted the difference. 
Dont test the internal implementation. Unit test is strictly tesing output from the input.  

How can we write proper test here? Tough! We need to get rid of \`enum\`. Sad but true. Enums don't allow you to inject
dependencies. They don't have constructor. If this were a class/struct, we could use constructor injection to provide the feature toggles. 
Lets do that up next.


<a id="orgfcff107"></a>

# Refactoring the OneTapFearure to make it testable

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

Now lets revisit the test case.

    class OneTapFeatureTests: XCTestCase {
    
        func test_whenCartFeatureToggleIsOff_thenCartOneTapIsDisabled() {
            let feature = OneTapFeature(from: .cart, isCartToggleEnabled: false)
            XCTAssertFalse(feature.isEnabled)
        }
    
    }

We now can control the input and hence test for the output. This is future proof too: if I were to 
refactor the \`isEnabled\` to use country locale after few months, I would change the constructor and 
my CI would notify me as the tests won't compile. Not compiling tests is failed test too.  

Notice that the default parameter uses the FeatureToggle right in the function declaration. This allows the 
calling client to not worry about passing in these values. They can omit them. It will sensibly default. 
In tests however, we will swap them. Nice right!


<a id="org6cc2c5a"></a>

# How about testing the present()

We can now follow the same principle and in constructor argument inject the AppNavigator type. Since AppNavigator 
can be modeled as protocol this can be mocked in test independently.

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

As we have used the constructor injection with default parameter for navigator, client implementation of production code can
be as is. They dont need to pass in Navigator type. It will sensibly default. For the test, we have an oppurtunity to use 
our mocked Navigator. This helps us check for side effects (some call it behavior). I usually prefer
to nest the mock type declaration inside the namespace of the test case. This avoids leaking mocks outside. However if there 
are other tests that need the same mock then create a factory to generate the mocks. Other important thing to note is, 
\`.product(with: UUID().uuidString)\` rather than using \`.product(with: "someId")\`. I leanred form Paul to not use literal as 
values in test.  

However, although all looks fine. We are certainly not following proper Unit Testing rule. We said its supposed to be black box 
testing. In the \`present()\` test, we are not testing for the output directly. \`present()\` doesnot return anything back. We are testing 
outcome of calling present. The outcome is calling into the navigtor object. Its a side effect. This is very common sight in 
Object Oriented Programming but certainly could be improved. We will see how to do so in the follow up post!


<a id="orga963038"></a>

# What if our client module is using Obj-C

As much as I like to use swift everywhere, not every app is swift only. So what if the client module is Obj-C? It can't 
represent Enum with associated value. It thereby cannot initialize OneTapCheckoutFeature as its public initializer uses 
enum which cannot be represented in Obj-C.  

There are couple of strategy to solve this issue. I wont muster the details here but present a solution I resort most of the 
times. Its by wrapping the enum into a class. All the cases would in turn be static functions so that the API still feels the 
same. Lets see it in action.

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

There are minimal changes to be made in the actual OneTapCheckoutFeature type.

-   \`convenience init\` just exposes the entryPoint to be passed. Others will default.
    Its good when the main initializer uses types that cannot be represented in Obj-C but 
    have default argument.
-   both \`isEnabled\` and \`present()\` are exposed by \`@objc\` and \`public\`
-   on the switch, we now switch on the internal \`value\` (the backing storage swifty enum)

    ```
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

Other approaches that Im aware of is either by extending the type and creating factory method. Or creating 
a class that waraps each enum case. I however believe this is very neat way to expose swift enum based 
api to obj-c. Note that, Obj-C can only create the OnePointEntryPoint. Given two instances of different 
OneTapEntryPoint, Obj-C doesnot have the ability to read what case a instance has internally. You can still do  memory comparision. 


<a id="org13ee52f"></a>

# Conclusion

Enum are powerful type. Swift is awesome; it features enum with associated values. This allows for neat 
pattern matching and api design. However it does have few corner cases especially on testability and objc interoperability.  

We saw that for unit test to be practical we need to be able to control input and assert on the output. Enums don't 
allow arbitary argument via constructor injection nor property injection (I dont prefer this mutation). 
****Thereby methods on enum that depend on anything other than the enum cases immediately makes it hard to test.****  

Second, in case we use enum based api then we might run into problems when exposing to Obj-c caller. In this instance,
we mimicked enum type with class types where each case is modelled to a static member or function.  

We left 1 point to disucuss on further follow up post. 

1.  Why testing for side effect is a bad practice? How can we improvise it?

Thank you for reading this post. If you learned something, consider spreading the knowledge. If you have some suggestion,
improvement or comments please leave a comment below or DM me on twitter at [@kandelvijaya](https://www.twitter.com/kandelvijaya). Enjoy!
