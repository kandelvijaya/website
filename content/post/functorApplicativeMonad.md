---
title: "Functor >> Applicative >> Monad"
description: "Short summary"
date: 2018-03-25T22:57:53+02:00
author: "kandelvijaya"
tags: ["fp", "functor", "applicative", "monad", "swift"]
---


# Table of Contents

1.  [What the heck are this FP terminology?](#orgd3b7290)
2.  [Function](#org4106d78)
3.  [Partial application](#org6dac770)
4.  [Lifting](#org9eb080f)
5.  [Currying](#orgd2a5a72)
6.  [Function Composition](#orgaae5c67)
7.  [Functor](#org7a4bfd5)
8.  [Applicative](#orgff1f53e)
9.  [Monad](#org3a37ba0)
10. [Conclusion](#org9fbc2b4)



<a id="orgd3b7290"></a>

# What the heck are this FP terminology?

Functional programming is out there. Partly rumor, partly practiced, partly understood. I have been 
writing about them for around 6 months. In this post, I want to summarize the main terminologies (jargons)
very concisely, with example and best of all, with swift code. For a full explanation, I will link to my 
previous blogs and external resources as we go along. Lets get going. 


<a id="org4106d78"></a>

# Function

Really this is the simplest. You all know it. However in functional programming, the function are equivalent to 
functions from mathematics. They were further enhanced based on lambda calculus. However please note that, 
pure function can be treated as data, function is type that helps abstraction and encapsulation and function is 
all that is in Functional Programming language. Function can be seen as atoms.\ 

A function is the thing we know but with this limitations:

1.  A function can take only 1 argument and must return 1 argument.
2.  A function should produce the same result for the same input every single time. No matter if this is 
    executed in radioactive environment where bit flipping might corrode CPU registers thereby producing bad 
    result. (SpaceX's falcon rocket uses 3 or more CPU, computes a function on each and compares for the correctness.) Now think,
    if `arc4random_unifrom()` would satisfy this rule or not.


<a id="org6dac770"></a>

# Partial application

Since a function can only take 1 argument, how do you make a function that take 2 int's to add. Good point! 
Since a function can return 1 argument, that 1 argument can be a function. That output function takes second 
input to produce the final result. 
```swift
    func add(_ a: Int) -> (Int) -> Int {
        return { b in
            return a + b
        }
    }
    
    let partialAdd5 = add(5)
    let add10to5 = partialAdd5(10)
    let add15to5 = partialAdd5(15)
```
When you apply a value to a function and get another function. You partially applied the function. In the above example it 
was `partialAdd5: (Int) -> Int`.


<a id="org9eb080f"></a>

# Lifting

There can be 2 kinds of values. A concrete value like `5` or `"Malcolm Gladwell"`. A wrapped value like Optional(5). When a concrete value is wrapped or boxed 
in another type its a lifted value. Usually the wrapper or box adds some context. For instance, while any instance of `Int` is concrete, 
any instance of `Optional<Int>` is a lifted value. `Optional` box adds a context that there might be value of `Int` or none.\
Presence of concrete value inside optional is the context. The process of taking concrete value to Wrapped/Contextual value is called lifting.
Why would anyone want to lift? Good point, and yet we do lifting all the times. Its needed!
```swift
    func div(_ a: Int, _ b: Int) -> Optional<Int> {
        if b == 0 { return nil }
        return a / b
    }
```
Here, function `div` will produce lifted value from concrete values. 


<a id="orgd2a5a72"></a>

# Currying

Remember the first point, a function in FP can only take 1 argument. How can we write the above `div` considering, in FP, a function
can take 1 input and no more. Enter currying. (Nothing to do with cooking skill!).
```swift
    func curriedDiv(_ a: Int) -> (Int) -> Optional<Int> {
        return { b in
            if b == 0 { return nil }
            // ⤵️ swift compiler lifts the type Int to Optional<Int> by default
            return a / b
        }
    }
```

<a id="orgaae5c67"></a>

# Function Composition

Given a value and a function, how can you apply the value to the function? That was pretty simple right! That's composition.\  

**Q: How can I apply f: `A -> B` when I have a concrete value: `A`**\
```swift
    let value = 12
    
    func function1(_ a: Int) -> String {
        return "Say \(a)"
    }
    
    func function2(_ a: Int) -> Int {
        return a + 2
    }
    
    // ⤵️ This is function composition
    function(value)
    
    // ⤵️ This is function composition
    function1(function2(value))
```

<a id="org7a4bfd5"></a>

# Functor

The word comes from category theory. Let me explain a bit of it. No, don't run away. I was kidding, I wont explain.
Keeping the math away, its very very simple. It answers this simple question.\

**How can I apply f: `A -> B` when I have a contextual value: `F<A`>?**\
Let me rephrase this way: Its the same as above function composition, the only thing that changed is the `let value = Optional.some(12)`. 
Let me rephrase yet another way: how can I map contextual value with a normal function.  
```swift
    let value = Optional.some(12)
    
    func say(_ a: Int) -> String {
        return "Say \(a)"
    }
    
    // functor:: apply say to value. 
    
    // Solution 
    // fmap in haskell
    // ⤵️ this map functions does lifting too...................⤵️
    func map<T,U>(_ transform: ((T) -> U), to: Optional<T>) -> Optional<T> {
        switch to {
        case let .some(v):
            return .some(transform(v))
        case .none:
            return .none
        }
    }
    
    // functor helps composition when value is contextual
    // functor type is mappable
    // optional is a functor
    map(say, to: value)
    
    // swift's optional is a functor as it provides `map`
    value.map(say)
```
Following on, keep this in mind. How do we compose when we have a value and a function. Remember both value and function can be in 2 states. 
Normal and Contextual (lifted).\

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-left" />

<col  class="org-left" />

<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">Value</th>
<th scope="col" class="org-left">Function</th>
<th scope="col" class="org-left">Output</th>
<th scope="col" class="org-left">How do we compose</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left">T</td>
<td class="org-left">T -> U</td>
<td class="org-left">U</td>
<td class="org-left">Normal application</td>
</tr>


<tr>
<td class="org-left">Contextual&ltT&gt</td>
<td class="org-left">T -> U</td>
<td class="org-left">Contextual&ltU&gt</td>
<td class="org-left">Functor, map</td>
</tr>


<tr>
<td class="org-left">T</td>
<td class="org-left">Contextual&ltT -&gt U></td>
<td class="org-left">&#xa0;</td>
<td class="org-left">??</td>
</tr>


<tr>
<td class="org-left">Contextual&ltT&gt</td>
<td class="org-left">Contextual&ltT -&gt U></td>
<td class="org-left">&#xa0;</td>
<td class="org-left">??</td>
</tr>


<tr>
<td class="org-left">T</td>
<td class="org-left">T -> Contextual&ltU&gt</td>
<td class="org-left">Contextual&ltU&gt</td>
<td class="org-left">normal</td>
</tr>


<tr>
<td class="org-left">Contextual&ltT&gt</td>
<td class="org-left">T -> Contextual&ltU&gt</td>
<td class="org-left">&#xa0;</td>
<td class="org-left">??</td>
</tr>


<tr>
<td class="org-left">T</td>
<td class="org-left">Contextual&ltT&gt -> U</td>
<td class="org-left">&#xa0;</td>
<td class="org-left">??</td>
</tr>


<tr>
<td class="org-left">Contextual&ltT&gt</td>
<td class="org-left">Contextual&ltT&gt -> U</td>
<td class="org-left">U</td>
<td class="org-left">normal</td>
</tr>


<tr>
<td class="org-left">T</td>
<td class="org-left">Contextual&ltT -&gt Contextual&ltU&gt></td>
<td class="org-left">&#xa0;</td>
<td class="org-left">??</td>
</tr>


<tr>
<td class="org-left">T</td>
<td class="org-left">Contextual&ltContextual<T&gt -> U></td>
<td class="org-left">&#xa0;</td>
<td class="org-left">??</td>
</tr>


<tr>
<td class="org-left">Contextual&ltT&gt</td>
<td class="org-left">Contextual&ltT -&gt Contextual&ltU&gt></td>
<td class="org-left">&#xa0;</td>
<td class="org-left">??</td>
</tr>


<tr>
<td class="org-left">Contextual&ltT&gt</td>
<td class="org-left">Contextual&ltContextual<T&gt -> U></td>
<td class="org-left">&#xa0;</td>
<td class="org-left">??</td>
</tr>
</tbody>
</table>

Out of all these permutation, we can compose if we know these 4 ways. The 4th can be express in term of the third.
However we are going to see both.

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-left" />

<col  class="org-left" />

<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<thead>
<tr>
<th scope="col" class="org-left">Value</th>
<th scope="col" class="org-left">Function</th>
<th scope="col" class="org-left">Output</th>
<th scope="col" class="org-left">How do we compse</th>
</tr>
</thead>

<tbody>
<tr>
<td class="org-left">T</td>
<td class="org-left">T -> U</td>
<td class="org-left">U</td>
<td class="org-left">normal</td>
</tr>


<tr>
<td class="org-left">Contextual&ltT&gt</td>
<td class="org-left">T -> U</td>
<td class="org-left">Contextual&ltU&gt</td>
<td class="org-left">functor</td>
</tr>


<tr>
<td class="org-left">Contextual&ltT&gt</td>
<td class="org-left">T -> Contextual&ltU&gt</td>
<td class="org-left">&#xa0;</td>
<td class="org-left">&#xa0;</td>
</tr>


<tr>
<td class="org-left">Contextual&ltT&gt</td>
<td class="org-left">Contextual&ltT -&gt U></td>
<td class="org-left">&#xa0;</td>
<td class="org-left">&#xa0;</td>
</tr>
</tbody>
</table>


<a id="orgff1f53e"></a>

# Applicative

This answers the question to:
**How can I compose a contextual value: `F<A>` to a function: `F<(A->B)>` that is wrapped in contextual type**

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<tbody>
<tr>
<td class="org-left">Value</td>
<td class="org-left">Function</td>
</tr>


<tr>
<td class="org-left">Contextual&ltT&gt</td>
<td class="org-left">Contextual&ltT -&gt U></td>
</tr>
</tbody>
</table>

I will write about when can one encounter such scenario during iOS development in the next post on Applicative. For now lets see the code.
```swift
    let value: Optional<Int> = Optional.some(12)
    let wrappedFunction: Optional<Int -> Int> = Optional.some(partialAdd5)
    
    // applicative
    // <*> in haskell
    func apply<T,U>(_ wrappedF: Optional<T -> U>, to value: Optional<T>) -> Optional<U> {
        switch (wrappedF, value) {
        case let (.some(f), .some(v)):
            return .some(f(v))
        default:
            return .none
        }
    }
    
    apply(wrappedFunction, to: value)
```

<a id="org3a37ba0"></a>

# Monad

This is interesting one. It answers the question:
**How can I compose when I have a contextual value and a function that lifts a concrete value?**

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">


<colgroup>
<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<tbody>
<tr>
<td class="org-left">Value</td>
<td class="org-left">Function</td>
</tr>


<tr>
<td class="org-left">Contextual&ltT&gt</td>
<td class="org-left">T -> Contextual&ltU&gt</td>
</tr>
</tbody>
</table>

We can encounter this scenario all the time in iOS development. Say your network service fetch returned you `Result<Data>`, 
now your transformer function takes `Data -> Result<JSON>` or `Data -> Result<MovieModel>`. How do you compose?
```swift
    let value: Optional<Int> = Optional.some(12)
    let div13: Int -> Optional<Int> = curriedDiv(13)
    
    // bind or >>= in haskell
    func flatMap<T,U>(_ transfrom: T -> Optional<T>, to value: Optional<T>) -> Optional<U> {
        switch value {
        case let .some(v):
            return transfrom(v)
        default:
            return .none
        }
    }
    
    // use case
    value.flatMap { div13($0) }
```

<a id="org9fbc2b4"></a>

# Conclusion

This post is a very concise way of looking at what each functor, applicative and monad means. You might have 
guessed by the theme of the article; they all have to deal with composition without boilerplate switch case. 
In Swift, Collection has both `map` and `flatMap` which makes it monadic. Optional<T> is monadic, there exists 
`map` and `flatMap`.\ 

The result type we saw earlier is Monadic. Check-out [Kekka Repository](https://github.com/kandelvijaya/Kekka) to see full implementation of Result<T> and 
Future<T>.\

I will try to build on this article for the next week, where we will see how Collection conforms to Monadic type. 
Following that post, We will build 2 parser: JSON and Org file (For those of you who don't know org-mode, you are missing out.)\

For further reference materials; here's my most easy and interesting picks:

-   [Monads for functional programming by Philip Wadler (Great insight)](http://homepages.inf.ed.ac.uk/wadler/papers/marktoberdorf/baastad.pdf)
-   [Functional programming Design patterns by Scott Wlaschin (Video)](https://vimeo.com/113588389)
-   [Functor (previous blog article)](https://kandelvijaya.com/2017/05/28/fp-functor/)
-   [Why monadic computation in swift? (previous blog article)](https://kandelvijaya.com/2017/06/25/whymondaiccomputation/)\

I hope you enjoyed this brief summary of core functional composition patterns. I hope you are ready to go in the wild 
and research more. If you liked this article, enjoyed it, share it. If there are any suggestions or comments you can 
reach out to me on Twitter [@kandelbj](https://www.twitter.com/kandelvijaya) or down below in comment section. Happy coding and see you next week!

