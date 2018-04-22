---
title: "Equality on Error"
description: "A better way to compare errors!"
date: 2018-04-21T22:57:53+02:00
author: "kandelvijaya"
tags: [ "error", "NSError", "swift"]
---


# Table of Contents

1.  [Introduction](#org23637c6)
2.  [Expectation](#orgd13c4d2)
3.  [Inside Error](#org3d1eb9f)
4.  [Inside NSError](#org11663de)
5.  [NSError is bridged to Error](#org2b73e22)
6.  [Equality on NSError](#orga1037aa)
7.  [Equality on Error](#org6e31021)
8.  [Equality for Error](#org271fe56)
9.  [Reflection to the Rescue](#org78e9f91)
10. [Conclusion](#org5a97a7e)
11. [References](#orgcff0203)



<a id="org23637c6"></a>

# Introduction

When do you really want to compare errors? → In the unit tests. 

Imagine this scenario. By now you should be aware of how important \`Result<T>\` is. If not, 
you can review the concept with the post [Result type in-depth blog](https://kandelvijaya.com/2017/06/25/whymondaiccomputation/). 

```swift
    func testWhenResultWithFailureIsMapped_thenOutputIsResultWithInitialFailure() {
        let failInt = Result<Int>.failure(error: IntError.testError)
        let output = failInt.map { "\($0)" }
    
        switch output {
        case let .failure(error: e):
            let expected = IntError.testError
            XCTAssertTrue(e == expected)   // == cant be applied to operands of Error
        case .success:
            XCTFail("mapping over failed type should propagate error")
        }
    }
```
Where IntError is simply:
```swift
    enum IntError: Error {
        case testError
        case unknown(String)
    }
```
**Q: How can I compare 2 instances of Error type properly?**\
First let's see what Error and NSError looks like behind the scene.


<a id="orgd13c4d2"></a>

# Expectation

What we really want to do by the end of this post is this. Its much readable and involves less code.
```swift
    func testWhenResultWithFailureIsMapped_thenOutputIsResultWithInitialFailure() {
        let failInt = Result.failure(error: IntError.testError)
        let output = failInt.map { "\($0)" }
        XCTAssertTrue(areEqual(output.error!, IntError.testError))
    }
```

<a id="org3d1eb9f"></a>

# Inside Error
Error is a protocol defined in ErrorType.swift file in Swift Standard Library. _domain and _code are both private properties and hence for the public side its just a empty protocol.
```swift
    public protocol Error {
      var _domain: String { get }  // private 
      var _code: Int { get }  // private
    }
```

<a id="org11663de"></a>

# Inside NSError

Each NSError object encodes three critical pieces of information: 
a \`status\` code, corresponding to a particular error \`domain\`,
 as well as additional context provided by a \`userInfo\` dictionary.<sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup>
```swift
    open class ApproxNSError{
        open var code: Int
        open var domain: String
        open var userInfo: [String : Any]
    }
```

<a id="org2b73e22"></a>

# NSError is bridged to Error

Take a example of \`JSONSerialization\` Obj-C API which would take \`NSError\` pointer. This API 
is interpolated to Swift with \`try..catch\` whereas \`try..catch\` uses Error not the NSError. NSError is bridged 
to Error.
```swift
    func test_malformedJSONDataCannotBeDeSerialized() {
        let jsonData = "{ \"name\": \"Swift\"".data(using: .utf8)!
    
        do {
            let json = try JSONSerialization.jsonObject(with: jsonData, options: .allowFragments)
        } catch  {
            print(error)
            // NSError is bridged to Error
            // Error Domain=NSCocoaErrorDomain Code=3840
            // "Unexpected end of file while parsing object."
        }
    }
```

<a id="orga1037aa"></a>

# Equality on NSError
Equality on NSError is given by `isEqual()` API which is inhabited on every single NSObject subtype.
```swift
    let er = NSError(domain: "a", code: 1, userInfo: ["name": "something"])
    let er2 = NSError(domain: "a", code: 1, userInfo: ["name": "something"])
    let erd = NSError(domain: "b", code: 2, userInfo: nil)
    
    er.isEqual(er2)  // true
    er.isEqual(erd)  // false
```
This looks great.


<a id="org6e31021"></a>

# Equality on Error

1.  We can't extend \`Error\` to conform to \`Equatable\` as Error is a protocol.
```swift
    extension Error: Equatable { // Extension of `protocol` cant have inheritance clause
    
    }
```

2.  Lets try using the bridge to NSError and comparing there.
```swift
    let (e1,e2) = (IntError.testError, IntError.testError)
    (e1 as NSError).isEqual(e2 as NSError)  // true
    
    let (e3,e4) = (IntError.unknown("float"), IntError.unknown("hex"))
    (e3 as NSError).isEqual(e3 as NSError)  // true
    
    // NOTE
    (e3 as NSError).isEqual(e4 as NSError)  // trure :: should be false
```
The solution kind of works but does provide false positive for Error types with 
associated values. It is because the associated values are solely for swift purpose, 
they are not bridged to Obj-C. There is no API that represents the associated values 
in \`NSError\`. However, rest assure that the metadata is kept safe if you were to bridge 
back to \`Error\` type again.
```swift
    let flaotUnknown = IntError.unknown("Float")
    let floatUknownNS = (flaotUnknown as! NSError).copy() as! NSError
    floatUknownNS.code      // 0
    floatUknownNS.domain    // "__lldb_expr_29.IntError"
    floatUknownNS.userInfo  // [:]
    
    // associated value is only representable with Swift enum types
    let backToErr = floatUknownNS as! Error  // unknown("Float")
```

3.  This begs the fact that equality has to be done in 2 levels. Equality of Error
    by bridging to NSError and Equality on associated values of Error.

4.  One cleaver way to check for associated values      are identical in `Error` is by swift 
    standard representation of types with `String(describing:)`. As we know there can't be 
    two cases of `parseFailed` cases, the types of associated values are irrelevant to be checked for.
```swift
    let parseFailed = IntError.parseFailed(column: 10, row: 30, reason: "Expected digit but got {")
    let parseFailedEOF = IntError.parseFailed(column: 0, row: 0, reason: "Unexpected EOF")
    
    let pf1 = String(describing: parseFailed)  // "parseFailed(column: 10, row: 30, reason: "Expected digit but got {")"
    let pf2 = String(describing: parseFailedEOF)
    
    pf1 == pf2  // false
```

<a id="org271fe56"></a>

# Equality for Error
```swift
    // Equality on instance of same typed Errors
    public func areEqual<T: Erorr>(_ lhs: T, _ rhs: T) -> Bool {
        // Swifty check 
        guard String(describing: lhs) == String(describing: rhs) else {  
            return false
        }
    
        // Sanity check
        let nsl = lhs as NSError
        let nsr = rhs as NSError
        return nsl.isEqual(nsr)   
    }
```
1.  Swifty check depends on the fact that `T` either doesn't conform to `CustomStringConvertible` or 
    that `CustomStringConvertible` conformance does the right job. There is no way of making sure something like this can 
    happen.
```swift
    extension IntError: CustomStringConvertible {
        var description: String {
            return "Case independent string"
        }
    }
    
    // without sanity check this would be TRUE
    areEqual(IntError.testError, IntError.unknown("a")) 
```
2.  Sanity check relies on the point 1 where there might be programmer error on conforming to correct 
    `CustomStringConvertible`. In such case, we want to gracefully check for bridged `NSError` comparison.

2.  Is there a better way? Sure there is!


<a id="org78e9f91"></a>

# Reflection to the Rescue

A better idea would be do the reflection on Error and compare the reflected values. 
This is neither affected by improper implementation of `CustomStringConvertible` nor we need to do 
additional sanity check on `NSError`.
```swift
    /**
     This is a equality on any 2 instance of Error.
     */
    public func areEqual(_ lhs: Error, _ rhs: Error) -> Bool {
        return lhs.reflectedString == rhs.reflectedString
    }
    
    
    public extension Error {
        var reflectedString: String {
            // NOTE 1: We can just use the standard reflection for our case
            return String(reflecting: self)
        }
    
        // Same typed Equality
        public func isEqual(to: Self) -> Bool {
            return self.reflectedString == to.reflectedString
        }
    
    }
    
    
    public extension NSError {
        // prevents scenario where one would cast swift Error to NSError
        // whereby losing the associatedvalue in Obj-C realm.
        // (IntError.unknown as NSError("some")).(IntError.unknown as NSError)
        public func isEqual(to: NSError) -> Bool {
            let lhs = self as Error
            let rhs = to as Error
            return self.isEqual(to) && lhs.reflectedString == rhs.reflectedString
        }
    }
```

<a id="org5a97a7e"></a>

# Conclusion
I knowingly went through the post in a exploration manner where I showed you what `Error` and `NSError` types look like. How are they bridged and How one might be tempted to use `String(describing:)` to compare them but might run into issues if `Error` instances conformed to malformed `CustomStringDescription` protocol. We finally had a solid reflection based 1 liner that would work in all the case. During the time of writing the blog, I also made a pull request to the `Apple Swift` github repo (Bonus point for me 🤓) on ErrorType.swift file. 

With that information, now its time you can compare \`Error\` correctly and write nice 
unit test case scenario. 

I hope you liked the format of this post rather than presenting you the answer right away. Should there be any comments, suggestions or issues feel free to DM me at https://www.twitter.com/kandelvijaya or comment down below. Enjoy coding!

<a id="orgcff0203"></a>

# References

1.  [Testing Swift Error Type: In depth exploration by Marius Rackwitz](https://academy.realm.io/posts/testing-swift-error-type/)


# Footnotes

<sup><a id="fn.1" href="#fnr.1">1</a></sup> [NSerror post on NSHipster](http://nshipster.com/nserror/)
