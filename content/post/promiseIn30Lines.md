+++
author = "kandelvijaya"
date = "2017-08-06T22:05:02+02:00"
description = "Liner composition of async tasks using Monad."
tags = ["Monadic Networking", "Promise", "Monad", "Swift"]
title = "Monadic Networking: I Promise!"

+++

In functional programming (Haskell), Monad is used to encapsulate side effects. This is how IO (printing to console, getting line feed) and networking (getting things asynchronously from cloud) is done. In the end, we hide all the impurity behind the Monad and work in pure functional grounds. Can we hide our dirty async tasks and callback behind a monad in Swift? Can we achieve the same purity, declarative and composable properties in Swift? 

This post is about taking a simple async task and encapsulating in a monad and then composing on top of this abstraction. This is not the definitive way to solve such problem but it is to me very simple and elegant. We will talk about some alternatives when possible.

## In the end we will have done 3 things:

- Understand and know how to encapsulate async tasks (getting rid of callback hell)
- See how we can use this type to encapsulate and create declarative animation API. 
- See how 30 lines can do all these above tasks. 

### For minors:
- The type we will implement is common or popular as **Promise**. We will see what it is.
- We will see how **Functor** is a requirement for **Monad**. 


# The callback hell
Problem: Say you need to get string from [This site](www.kandelvijaya.com) and then from this amazing [www.objc.io](www.objc.io) site for reasons you can think of.  

Obviously, this looks familiar. Say I need to chain other sites only if the second on succeeds.

```swift
let session = URLSession(configuration: .default)
session.dataTask(with: kvurl) { (dataO, responseO, errorO) in
    if let data = dataO, errorO == nil {
        session.dataTask(with: objciourl) { (data2O, response2O, error2O) in
            if let data2 = data2O, error2O == nil {
                // doSomething
            }
        }.resume()
    }
}.resume()
```

The problem however is not that this is a playground code and is thus not fine tuned. The problem we face is, this is how we think of aysnc code. This kind of nested callback baked code gets into release and when we want to add 1 more feature down the line; we add 1 more nesting. See the picture; we go off the screen. This to me is not real bad. I can scroll the screen. The real trouble here is; Its a lot manual, concerned and fragile process of tackling repeating pattern. DRY. Is it?  See the two `resume()` method calls. 

A immediate solution to clean up is to move the nested network call and supply via a completion handler. Its 1 step in our direction but to me it doesn't provide a good enough abstraction. 

# Our solution:

```swift
Network(kvurl).get().bind { _ in
        return Network(objciourl).get()
    }.then { _ in
        /* do something */
    }.execute()
```

Thats it!. Don't worry about `bind` and `then`. We shall see they are very simple functions too.

Some might find it coming. The promise / future which ever you call it. Lets not get there yet. 

Let me show you yet another example:

### Nested animation!!  (The imperative)
```swift
    UIView.animate(withDuration: 2, animations: {
        view.backgroundColor = .red
    }) { _ in
        UIView.animate(withDuration: 3, animations: {
            view.backgroundColor = .green
        }) { _ in
            UIView.animate(withDuration: 4) {
                view.backgroundColor = .purple
            }
        }
    }
```

See that innermost animation block. This is just a trivial example. You might nest 10 level deep or live with 3. You got the point: we went literally deep. Our code has repeating `UIView.animateWith....` and brackets. 

### Declarative Animations!!! (The Functional)

```swift
    view.animate(with: 2) {
        $0.backgroundColor = .red
    }.animate(with: 3) { 
        $0.backgroundColor = .green
    }.animate(with: 4) {
        $0.backgroundColor = .purple
    }.execute()

```

Think how the second code would look if you had to do 10 chained animations. Just more lines, not nesting!

One could step up one more and write a sequence function to execute animation tokens from an array in order like such.

```swift
Promise(view).animate(with: [
    AnimationToken(duration: 4) { $0.backgroundColor = .white },
    AnimationToken(duration: 2) { $0.backgroundColor = .green },
]).execute()
```

If you are interested into declarative animation then I highly suggest **John Sundell**'s recent [Blog post](https://www.swiftbysundell.com/posts/building-a-declarative-animation-framework-in-swift-part-1) where he takes you from ground up to build declarative animation API. My aim here was to show, how you can use this `Promise` type to reach to similar ground. 

The only price you pay to do such declarative programming is either language to support directly or third party libraries which you can embed and forget about how it works. The cost you pay for if you want to use Promise / Future is to understand them. 

There exists numerous third party libraries that provide this feature of linear progression of async tasks and more. However, I find it more interesting to implement them myself and with a different twist. 

## Enter Promise (It shall be monadic)
A promise is a return type of a async task. All it represents is the type of the eventual result it will have, immediately. 

Think of it as a box which will provide a liquid pipe to the caller. The caller knows what kind of liquid he/she will get when the box is filled. Maybe water, alcohol or beer. This is the BOX's promise (the liquid type). While the box might take time to fill before pushing to caller, the caller can act as if he already has such liquid. Thats all.

In our case, a async task can produce a promise. The pipe will be the function `then` through which you supply what work you will do when you get the liquid (type of result).

## There it is, in the entirety!
```swift
public struct Promise<T> {

    public typealias Completion = (T) -> Void

    var aTask: ((Completion?) -> Void)? = nil

    public init(_ task: @escaping ((Completion?) -> Void)) {
        self.aTask = task
    }

    @discardableResult public func then<U>(_ transform: @escaping (T) -> U) -> Promise<U> {
        return Promise<U>{ upcomingCompletion in
            self.aTask?() { tk in
                let transformed = transform(tk)
                upcomingCompletion?(transformed)
            }
        }
    }

    public func execute() {
        aTask?(nil)
    }

}
```

That's almost it! I removed the documentation which you can find at [Github MPromise Repo](https://github.com/kandelvijaya/MPromise/blob/master/PromiseMonad.playground/Sources/Promise.swift).


#### Couple of points worth noting:

1. Promise is a Value type. 
2. Promise is Generic in terms of **T** where **T** represents the type of eventual value that the task produces. 
3. Initializer, is just a closure which represents usually async task that will call into the completion parameter with proper type when it is done. 
4. `then` is a decorator (in reverse) that wraps initial task inside another one. Essentially, this is what we were doing manually with nested callbacks.
5. Finally, nothing gets executed until `execute()` is invoked on the promise. 

Promise is a big expression. This is a subtle design choice I made on this type. The implication being we can build a big expression that doesn't necessarily have to evaluated when the program sees. We can store it in a property, pass to a function or copy it. We can also call the execute multiple times. The compiler then can also choose to optimize the expression when possible. One can compose 2 promises in ways never thought. And all this is good because a **Promise is a Value type**. 

### The constructor 

```swift

public class Network {

    public init(_ url: URL) {
        self.url = url
    }

    public func get() -> Promise<Result<Data>> {
        return Promise { aCompletion in
            let session = URLSession(configuration: .default)
            let task = session.dataTask(with: self.url) { (data, response, error) in
                if let d = data, error == nil {
                    aCompletion?(.success(d))
                } else if let e = error {
                    aCompletion?(.failure(e))
                } else {
                    aCompletion?(.failure(NetworkError.unknown))
                }
            }
            task.resume()
        }
    }

}

```

Just the same old networking code. However, notice the Promise we return immediately. This allows us to work in linear progression mode while Promise hides the asynchronous part from us. Remember, this is not anything new. Its just perspective change, right!

### The animation promise

```swift
extension UIView {
    func animate(duration: TimeInterval, animation: @escaping (UIView) -> Void) -> Promise<UIView> {
        return Promise<UIView> { aC in
            UIView.animate(withDuration: duration, animations: {
                animation(self)
            }) { finished in
                if finished {
                    aC?(self)
                }
            }
        }
    }
}

```

which allows one to then do these things:

```swift
view.animate(duration: 2) {
    $0.backgroundColor = .red
}.then { view -> Promise<UIView> in
    return view.animate(duration: 3) {
        $0.backgroundColor = .green
    }
}.then { promise -> Void in
    promise.then {
        $0.animate(duration: 4) {
            $0.backgroundColor = .purple
        }
    }
}
```

Fair enough this is quite dumb code. 

1. `then :: ((T) -> U) -> Promise<U>`

This is because `then` expects `(T) -> U` which does not refrain one to supply `(UIView) -> Promise<UIView>` in which case the result of the `then` is going to be `Promise<U> == Promise<Promise<UIView>>>`. 

This is exactly what happened up there. Now 1 way to solve it is by flattening/ joining the nested `Promises`. 

1. `join :: Promise<Promise<A>> -> Promise<A>`


This is exactly what monad's `bind` solves.

## Enter Monadic Promise.

```swift
    public func bind<U>(_ transform: @escaping (T) -> Promise<U>) -> Promise<U> {
        let transformed = then(transform)  //1. Promise<Promise<U>>
        return Promise.join(transformed)   //2. Promise<U>
    }

    static public func join<A>(_ input: Promise<Promise<A>>) -> Promise<A> {
        return Promise<A>{ aCompletion in
            input.then { innerPromise in
                innerPromise.then { innerValue in
                    aCompletion?(innerValue)
                }.execute()
            }.execute()
        }
    }
```

Its important to note the type signature of `bind` and `then`.

-  `then :: ((T) -> U)          -> Promise<U>`
-  `bind :: ((T) -> Promise<U>) -> Promise<U>`

Our animation API can now be in terms of `bind`:

```swift

view.animate(duration: 2) {
    $0.backgroundColor = .red
}.bind {
    return $0.animate(duration: 3) {
        $0.backgroundColor = .green
    }
}.bind {
    return $0.animate(duration: 4) {
        $0.backgroundColor = .purple
    }
}

```

Nice!!! Can we make it more readable. Sure we do!

```swift
extension Promise where T == UIView {
    func animate(with duration: TimeInterval, animation: @escaping (UIView) -> Void) -> Promise<UIView> {
        return self.bind { view in
            view.animate(duration: duration, animation: animation)
        }
    }
}
```

Note:
- We now generalize over the repeating code `return $0.animate(duration: ) { }`

Then our animation code looks like these:

```swift
view.animate(with: 2) {
    $0.backgroundColor = .red
}.animate(with: 3) { 
    $0.backgroundColor = .green
}.animate(with: 4) {
    $0.backgroundColor = .purple
}.execute()
```

