---
title: "Don't Fear Custom Operator!"
date: 2018-03-17T21:40:01+01:00
author: "kandelvijaya"
description: "Knowing when to use and not use custom operator"
tags: ["swift", "iOS", "custom operator", "|>"]
---

# Table of Contents

1.  [Introduction](#org6d0976e)
2.  [Creating custom operator](#org2549d97)
    1.  [Operator overload (The bad)](#org01d7629)
    2.  [Operator overload (The good)](#org815c941)
    3.  [Custom Operator](#orgc1e69a3)
        1.  [Normal composition](#orga6b5783)
        2.  [Direct composition](#org1caa61c)
        3.  [Left to Right application operator](#org20974f4)
    4.  [Creating Custom Operator |>](#org8cd7da2)
3.  [3 Q's before introducing custom operator](#org1398295)
4.  [Custom operator for map](#org0d96a31)
    1.  [fmap or map](#orge5eaaa4)
    2.  [|>> operator](#orgc84f7e0)
5.  [Conclusion](#org9647812)



<a id="org6d0976e"></a>

# Introduction

Operator (+, -, >) are just normal functions. Treated a bit differently (syntatically) by the compiler. 
```swift
    func +(_ lhs: Int, _ rhs: Int) -> Int
    
    // it could have been
    func add(_ lhs: Int, _ rhs: Int) -> Int
    
    // usasge
    1 + 2
    add(1,2)   
```
Operators help convey fundamental intent in universally accepted syntax. Even a non programmer 
can spot `1 + 2` is addition. + is both a fundamental operation and universally (+ mathematically) known
syntax. It makes the code clear, concise and conveys intent. 

-   In swift, one can define custom operator.
-   In swift, one can overload operators (however be cautious)

And yet, we rarely use custom operator. Engineers are biased to not use them. Today, we will 
explore this territory. See where custom operators makes sense and where it doesn't. I will provide 
my 3 questions to ask before creating custom operator. Lets get started!


<a id="org2549d97"></a>

# Creating custom operator

1.  Can be named with special characters only
2.  Can be infix, postfix or prefix but not ternary
3.  Defaults to DefaultPrecedenceGroup


<a id="org01d7629"></a>

## Operator overload (The bad)

Please don't do this. This is how one can overload `+` for `Int` and do nasty thing. This is what 
people refer to when they complain about custom operator and operator overloads being bad. 
```swift
    extension Int {
        static func +(_ lhs: Int, _ rhs: Int) -> Int {
            return lhs - rhs
        }
    }
    1 + 2  // -1 when it should be 3
```

<a id="org815c941"></a>

## Operator overload (The good)

Say we are working extensively with graphs and graphics. Its handy to be able to add 2 points.
```swift
    // Given 2 points we want to add
    let point1 = CGPoint(x: 12, y: 12)
    let point2 = CGPoint(x: 12, y: 100)
    
    // `+`
    extension CGPoint {
        static func +(_ lhs: CGPoint, _ rhs: CGPoint) -> CGPoint {
            return CGPoint(x: lhs.x + rhs.x, y: lhs.y + rhs.y)
        }
    }
    
    point1 + point2
```
This can however be done with named function.
```swift
    extension CGPoint {
        func add(_ anotherPoint: CGPoint) -> CGPoint {
            return CGPoint(x: x + anotherPoint.x, y: y + anotherPoint.y)
        }
    }
    point1.add(point2)
    
    // Which one would you prefer?
    point1.add(point2).add(point1)  // OO way
    point1 + point2 + point1        // Operator way
```
Surely the operator looks nice and conveys the addition intent. 


<a id="orgc1e69a3"></a>

## Custom Operator

Lets see a normal use case. [This code below is a example code. It can span across service, interactor 
and presenter]. Say you have there 3 methods to convert, transform and load the view with viewModel. 
```swift
    func convert(_ json: JSON) -> PersonModel {
        //..
    }
    
    func transform(_ model: PersonModel) -> PersonViewModel {
        //..
    }
    
    func load(_ viewModel: PersonViewModel) {
        //..
    }
    
    // How can we use the above methods here
    func loadContents(with json: JSON) {
        //.. lets focus here
    }
```

<a id="orga6b5783"></a>

### Normal composition

We love composing by temporarily assigning into variables. This makes for readable code.
We do it all the times. Let see if there is a better way. 
```swift
    func loadContents(with json: JSON) {
        let model = convert(json)
        let viewModel = transform(model)
        load(viewModel)
    }
```

<a id="org1caa61c"></a>

### Direct composition
```swift
    func loadContents(with json: JSON) {
        load(transform(convert(json)))
    }
```
The above is directly equivalent to the previous normal composition. If you look closely, 
`convert(:)` is the first method to be evaluated before evaluating `transform`. The code 
reads from inside to outside. Not left to right as we want. That is precisely why we favor 
the normal composition with temporary variable assignment.  

Can we somehow get rid of intermediate variable assignment while composing left to right; like we read. 


<a id="org20974f4"></a>

### Left to Right application operator

The below code is what we want. Its easy to comprehend. `|>` is just like Unix pipe `|`. We can't use 
`|` as they it is reserved for bitwise OR. 
```swift
    func loadContents(with json: JSON) {
        json |> convert |> transform |> load
    }
```
1.  |> is the UNIX pipe. Output from left is applied to the function on right.
2.  Code reads from input to function to function. Left to right.
3.  There are no intermediate variables associated. Code is smaller looks its nice (hehe)!

Now lets see how can we create such operator.


<a id="org8cd7da2"></a>

## Creating Custom Operator |>
```swift
    infix operator |>: ApplyPrecedenceGroup     // We will see this below
    
    // Note that you can use lowercased T, U for type signature. 
    // Its not encouraged but just to let you know. 
    func |><t,u>(_ value: t, _ function: (t) -> u) -> u {
        return function(value)
    }
```
We need the precedencegroup to help compiler resolve how to evaluate this piece of code.
```swift
    a |> fa |> fb 
    
    // () are evaluated first. Like in the old maths
    // 1. This is left associativity
    (a |> fa) |> fb
    
    //Or 2. This is right associativity
    a |> (fa |> fb)
```

And we need it to differentiate which operator needs to be evaluated first. Do we evaluate + first or |> first. 
higher priority operators are evaluated first. For instance which operator takes priority in this code snippet.

```swift
    a |> fa + fb |> fc
```

Lets write our precedence group.

```swift
    precedencegroup ApplyPrecedenceGroup {
        associativity: left
        higherThan: AssignmentPrecedence
    }
```
That's it. Now the operator can be used and our initial code `json |> convert |> transform |> load` will work. 
Now the question is When does it make sense to create custom operator? Importantly when it doesnot?


<a id="org1398295"></a>

# 3 Q's before introducing custom operator

I always try to ask myself these 3 questions before creating custom operator. Maybe this will help you guide too. I 
recently watched [Pointfree.co](https://www.pointfree.co) video on free function and they [Stephen](https://twitter.com/stephencelis) and [Brandon](https://twitter.com/mbrandonw) descirbed a similar approach 
which I totally agree. I recommend you guys check their video talks especially if you lurking into functional 
programming. 

1.  Operator is a function. Make sure you need to use that function all over the place.
2.  Operator changes syntax. Make sure the operator is widely used symbol; look out for examples in functional 
    programming languages. I tend to look into Haskell. It has tons of operator; for better good.
3.  Make sure operator enables more concise and readable code. Don't just use a operator for the cuteness of the 
    symbol. P.S. don't override standard operators.

Let me provide a horrible idea of overrideing `+` operator defined on Int. 
```swift
    // IMP: Dont override std lib defined operators. Please!
    extension Int {
        static func +(_ lhs: Int, _ rhs: Int) -> Int {
            return lhs - rhs
        }
    }
    
    1 + 2  // -1 rather than 3
```
Yes you need to be cautious. If you work in a team, make sure your team is educated about the use, misuse and provide 
examples or work session before you make a pull request with a fancy operator. People sometimes are super defensive to introduce custom operator 
with the fact that new hires will find hard to navigate the code-base. Its only true if you don't follow above 3 rules. If you do,
its a good addition and just useful function.


<a id="org0d96a31"></a>

# Custom operator for map

Functional programming has embraced the idea of operator but also provides a associated named function. For instance in Haskell,
`fmap` is the same as `<$>`. Just different names. 
```hs
    fmap (+ 2) [1,2,3]   -- [3,4,5]
    (+ 2) <$> [1,2,3]    -- [3,4,5] ;; this is infix
    
    (+ 2) <$> (Just 12)  -- Just 12 is the same as Optional<Int>.some(12)
```

<a id="orge5eaaa4"></a>

## fmap or map

The fmap I showed above is exactly the `map` method on `Array` or `Optional` in swift. 
```swift
    extension Array {
        func map<T>(_ transform: (Element) -> T) -> [T] { .. }
    }
```

<a id="orgc84f7e0"></a>

## |>> operator

In swift `<$>` cannot be used as custom operator as `$` is not a special character. I know few people use 
`<^>` but I don't prefer this symbol. I rather like `|>>`. The extra > symbol signals that operator will work on Functor 
types. 
```swift
    precedencegroup MapPrecedenceGroup {
        associativity: left
        higherThan: ApplyPrecedenceGroup
    }
    
    infix operator |>>: MapPrecedenceGroup
    func |>><T,U>(_ value: [T], _ transform: (T) -> U) -> [U] {
        return value.map(transfrom)
    }
    
    // usage 
    
    [1,2,3] |>> { $0 + 2 }     //  [3,4,5]

Then you can define the same operator function for Optional and Result type. 

    func |>><T,U>(_ value: Optional<T>, _ transform: (T) -> U) -> Optional<U> {
        return value.map(transfrom)
    }
    
    // usage 
    let labelO: UILabel? = UILabel()
    
    // We dont have to if let or explicit map on optonals. It certainly looks nice. 
    labelO |>> view.addSubview
```
**As swift protocol are incapable to model Functor, we have to implement this operator on instance of each Functor. This is tedious.**
A Functor is a contextual type that lets you map over. To read more [head over to this Functor blog](https://kandelvijaya.com/2017/05/28/fp-functor/).  


<a id="org9647812"></a>

# Conclusion

Don't fear the custom operator. They can be make code elegant and even improve readability when used wisely. And 
I have outlined my top 3 question to ask before including custom operator in your code. I hope you will make use 
of it in the days to come.  

As per the second operator, `|>>`, I don't have a strong opinion. I used it in my `Result<T>` module. I 
like to use `|>` for function application. In Haskell ` `(blank space) is the equivalent of this operator (with right 
associativity). In swift it is the `()` parens.  

I hope you enjoyed this post and learned something new. I had fun explaining custom operators. I would love to know 
how you use custom operators and which kinds? If you have any questions or suggestions; feel free to DM me on [@kandelvijaya](https://www.twitter.com/kandelvijaya) or 
comment down below. Happy coding and enjoy your weekends.  
