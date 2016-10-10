+++
author = "kandelvijaya"
date = "2016-10-10T21:57:04+02:00"
description = "Little glimpse of why AnyHashable exists in Swift3?"
tags = ["Swift3", "iOS Engineering"]
title = "Why Any Hashabel Swift3"

+++

Evolution is predominant. Struggle for Survival applies to just anything that you see. Swift Programming Language is not an exception. Swift continues to change, evolve and mature over time. We can keep our feet wet, migrating year after year to Swift X version. I would. If it strives to be better. This years, `Swift 2 -> Swift 3` was little more than a mini project. We saw lots of changes. For this edition, we will focus on `[AnyObject: NSObject]`, which became `[AnyHashable: Any]`. So does JSON, NSArray and NSDictionary. But why? We will dive deep, bear with me. 
# AnyObject -> Any

1.  Swift focuses on using Value Types / immutable type in all cases possible. Foundation in Objective-C, has in other hand, all reference type. Classes. Which will be imported into reference type in swift.
2.  `AnyObject` is idomatially ObjectiveC flavored and is reference type.
3.  Swift is platfrom independent, it had to move away from relying on ObjectiveC idioms and its runtime. Hence `AnyObject` had to be replace with Value type and Swift flovored `Any`.

## Some more thoughts

*   `Any` is value type. (We will see how it actually is boxing refernce but is Value type later on)
*   All Objective-C Foundation `id` types will be imported as `Any`. 
*   All Swift types including Enum and Struct can be bridged to Objective-C as `id`. This id is minimal. 
*   All Swift types that were bridged to Objective-C `id` can be bridged back to `Swift` as `Any` or casted to their previous Type. Swift doesnot remove the type information during the boxing; internally.
*   For Example:

\

    enum Direction2 : String {
    
        case down = "UP"
        case up = "DOWN"
    }
    var objcArray = NSMutableArray() // [NSObject, NSObject] or [id, id]
    var swiftEnum = Direction2.down
    objcArray.add(swiftEnum)
    
    objcArray.lastObject as? Direction2     //down
    objcArray.lastObject as? NSString //nil


# NSObject -> AnyHashable Consider the situation - 

`[NSObject: AnyObject]`. This turned into `[NSObject: Any]`. 
*   Its natural to get rid of `Object` feeling. Afterall, `AnyObject` became `Any`. 
*   Like before, we wanted to work with Value types. NSObject does 2 things which we want to avoid. 
    *   It is a reference type. (There is also NSObjectProtocol)
    *   It requires us to know about ObjectiveC idiom. Swift is again platform independent.
*   A `Dictionary`, `Array` , `Set` expects Key/Element to be `Hashable`. There is no requirement it can just be some few types. 
*   Hence, its more fluid to represent somewhat alien `[NSObject: AnyObject]` with `[AnyHashable, Any]`. 
*   Wait. Why not `[Hashable: Any]`? Good question. Lets see.

## Why not `[Hashable: Any]`?

`Hashable` conforms to `Equatable` Protocol which has `==` method requirement which has a `Self` associated type. `public static func ==(lhs: Self, rhs: Self) -> Bool` Hence, `Hashable` can only be used to contraint Generic Types but not be used as a Concrete Type. (For more on this `Generic` issue follow this **link**. ) Thus we need a **concrete type conforming to Hashable** that can fit into the Key of dictionary. We also need to enable heteregeneous collection because it needs to bridge to the Objective-C API NSArray and NSDictionary. Hence, we need a type erased container that confroms to `Hashable` to be used inplace of `NSObject`. That contianer is `AnyHashable`. 
# internals of AnyHashable If you already know 

**Boxing** (I mean data boxing. Very essential technique.) and have something else to do, you can stop here. Okay, seems like you want to do **Boxing**. Lets dig a little deep to see how and what `AnyHashable` does? Better yet, lets simulate a similar `Any2Hashable` together. 
## Boxing It is a technique of wrapping a Object inside another container type. It has nothing to do with 

`Decorator` pattern but you are on track. The best example of this is `Optional<Wrapped>` type. It takes anything and wraps it around with `Optional` enum. This gets rid of lots of assumptions we used to do in `Objective-C`. We will see how `AnyHashable` boxes in the following section. 
### Step 1: Basic implementation Then a naive way to wrap this or box all Hashable conformed type would be such. 

    struct Any2Hashable : Hashable {
        private var _box: Any
    
        var hashValue: Int {
            //LOOKOUT 1
            return (_box as? Hashable)?.hashValue ?? 0
        }
    
        public static func ==(_ lhs: Any2Hashable, _ rhs: Any2Hashable) -> Bool {
            return lhs.hashValue == rhs.hashValue
        }
    
        init?(_ any: Any) {
            //LOOKOUT 2
            if let thatAny = any as? Hashable {
                _box = thatAny
            }
            return nil
        }
    }
     However, for 2 

`lookout` the compiler gives us this error and stops from any real success. 
    //ERROR: Protocol Hashable can only be used as genric constraint because it has Self or associated type requirements
     These should throw back some lightbulbs. Its simple way of saying Hashable is just a Generic Type not a complete one. Because it conforms to Equatable which has Self requirements. 

[To read more on Generics][1] 
### Step 2: Improvement with Generics

    struct Any2Hashable{
        var _box: Any
        private var _hashValue: Int
    
        var hashValue: Int {
            return _hashValue
        }
    
        public static func ==(_ lhs: Any2Hashable, _ rhs: Any2Hashable) -> Bool {
            return lhs.hashValue == rhs.hashValue
        }
    
        init<T: Hashable>(_ base: T) {
            _box = base
            _hashValue = base.hashValue
        }
    }

This will compile fine and work too. However, Swift stdlib has a longer implementation for a reason. Lets take a look at a scenario before proceeding. 

    let iHA = Any2Hashable(12)
    let i2HA = Any2Hashable(UInt8(12))
    let sHA = Any2Hashable("bj")
    iHA == sHA     // FALSE
    iHA == i2HA    // TRUE :: Lookout
     As you can see, although the first comparion looks correct, the second one is somewhat a lie. 

**Swift is TypeSafe**. "A Int with 12 is not equal with Int8 with 12." The underlying memory representation are different and it should not be equal. Although it seems. With our implementation of Any2Hashable we completely ignored the underlying type for sake of brevity. However, Swift standard library goes in length to fix this subtle fact. 

    let swiftInt64Hashable = Int(12) as AnyHashable
    let swiftInt8Hashable = Int8(12) as AnyHashable
    swiftInt8Hashable == swiftInt64Hashable  // FALSE
     
We shall see how do they actually preserve type info during the comaprision although AnyHashable, from the outside, is a type erased container for Hashable. 

### Step 3: Bringing back the Type info

`_box: Any` is limiting us from type checking in our current implementation. What if we wrap the value that is being sent to initializer into a concrete internal struct. We could be on the way to storing typed type. 
    
    struct Any2Hashable{
        //LOOKOUT
        var _box: _InternalConcreteBox
    
        init<T: Hashable>(_ base: T) {
            _box = _InternalConcreteBox(base)
        }
    }

Nothing has changed here except we got rid of other helper methods to be concise. 

    struct _InternalConcreteBox<Base: Hashable> {
    
        var _baseHashable: Base
        var _hashValue: Int {
            return _baseHashable.hashValue
        }
    
        init(_ base: Base) {
            _baseHashable = base
        }
    
        //more code....
    }
     Although in the right direction, Compiler wont allow us to use 

`_InternalConcreteBox` as concrete type for `_box` as this is a Generic Placeholder and incomplete Type (like before). Other than that everything looks good. We could say, `_InternalConcreteBoc`_ is bocing a Type but it tries to preserve the original type info. Whereas, `Any2Hashable` is a type erased container. 
### Step 4: Solving Generics yet again with Protocol One way to solve this issue is by providing a complete Protocol Conformance with the required methods like so: 

    protocol _Any2HashableBox {
        var _hashValue: Int { get }
        func _isEqual(to: _Any2HashableBox) -> Bool?
    }
     and the Internal Box looks like so. Then we can use 

`_Any2HashableBox` as complete type. `struct _InternalConcreteBox<Base: Hashable>: _Any2HashableBox { ….` While at our Any2Hashable site, the below change compiles perfectly. 
    
    struct Any2Hashable{
        var _box: _Any2HashableBox
    

### Step 5: Typed Equality, finally! Now that we have every structure in place, lets fill in the `isEqual` detail. 

    struct Any2Hashable{
        var _box: _Any2HashableBox
    
        public static func ==(_ lhs: Any2Hashable, _ rhs: Any2Hashable) -> Bool {
            return lhs._box._isEqual(to: rhs._box) ?? false
        }
    
        init<T: Hashable>(_ base: T) {
            _box = _InternalConcreteBox(base)
        }
    }
    

#### Protocol for BoxType

    protocol _Any2HashableBox {
        var _hashValue: Int { get }
        func _isEqual(to: _Any2HashableBox) -> Bool?
    }
    

#### Concrete Box Struct

    struct _InternalConcreteBox<Base: Hashable>: _Any2HashableBox {
    
        var _baseHashable: Base
        var _hashValue: Int {
            return _baseHashable.hashValue
        }
    
        init(_ base: Base) {
            _baseHashable = base
        }
    
        func _isEqual(to: _Any2HashableBox) -> Bool? {
            //LOOK OUT
            if let other : Base = (to as? _InternalConcreteBox)?._baseHashable {
                return _hashValue == other.hashValue
            }
            return nil
        }
    
    }
     In this 

`//LOOK OUT ::`, we are getting hold of the `<Base: Hashable>` that was used to create `_InternalConcreteBox`. In swift current implementation this line is replaced by a method `_unbox<T:Hashable>() -> T?`. True, it requires deeper hair pulling. The above **lookout** line retrieves the original type only if it is same concrete underlying type as `Base` for self is. Sure, the line is not so clear on how it does in first sight but it uses Type inference to deduce the type while casting. Only if both self and other are same type we check for the hashValue and return. Lets see the tests: 
       
        let iHA = Any2Hashable(12)
        let swiftInt64Hashable = Int(12) as AnyHashable
        let swiftInt8Hashable = Int8(12) as AnyHashable
    
        swiftInt8Hashable == swiftInt64Hashable  // FALSE
        iHA == Any2Hashable(12)                  // TRUE
    

## Some Key Learnings: 
So far, we have seen how to box types. "NOTE: Boxing should be done only when absolutely necessary." We also saw how we can box types that actually preserves their original type and how it can leverage for cases like AnyHashable. This is the whole idea how AnyHashable in Swift core library works. There are other pieces of functionality I havent added just to make the topic concise. They can be found 

[on Swift github repo page][2]. They have rich documentation but requires a lot of researching to get to know the why were they created like the way they are. This is how, swift bridges NSArray to [AnyHashable] and NSDictionary to AnyHashable: Any] providing a homogeneous boxed collection to work with. 
### Cheers! Feel free to edit this post, if needed.

 [1]: http://kandelvijaya.com/index.php/2016/06/24/comparision-of-swift-programming-language-on-the-support-for-generics/
 [2]: https://github.com/apple/swift/blob/master/stdlib/public/core/AnyHashable.swift