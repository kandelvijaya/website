+++
author = "kandelvijaya"
date = "2016-12-10T21:57:04+02:00"
description = "Constrasting Range<String.Index> and NSRange API"
tags = ["Swift3", "iOS Engineering"]
title = "Why String Manipulation is alien in Swift3?"

+++

# Objective-C era
> NSRange has a very simple API. 
Finding a range, replacing, splitting or chopping are some few tricks off the bat for simple string manipulation. Objective-C and its idiomatic NSRange API looks such:

    public struct _NSRange {
        public var location: Int
        public var length: Int
    }

Lets say we wanted to extract just the name from a JSON string we got. 

    let a: NSString = “name: Bj P. Kandel”
    let name = a.substring(from: (“name: “ as NSString).length)


# Swift3 Era

    let aSwift = “name: Bj P. Kandel”
    let nameSwift = aSwift.substring(from: <String.Index>) 

## So what is the mess with String.Index?

NSString (and its NSRange) is not unicode aware. Swift intends to have great support for unicode. But do we care? 

> If you like Emoji then you do.

    let emojiOBJC: NSString = “🤓”
    emojiOBJC.length   //2 

You can see the emoji is actually 1 character for you and our user. But NSString doesn’t co-relate to natural understanding. It thinks its 2 character. If we were to substring the Emoji, we could get this familiar unknown representation symbol.

    emojiOBJC.substring(from: 1) //� 

### Why?
`NSString` uses `UTF-16` or 16 bits to encode a character into memory. When reading, __16 bit of memory is treated as 1 character__. So to find the length of a string, count the 16 bit memory. Straightforward. Shall we think a bit more. 

>16 bits == 2^16 possibilities == 65536 distinct characters that can be represented uniquely

> However, There are roughly 6,500 spoken languages in the world today. However, about 2,000 of those languages have fewer than 1,000 speakers. The most popular language in the world is Mandarin Chinese.  __This 16 bit cannot all the characters from all of those language + emojis__

That’s why Swift String were made more __unicode__ correct. Unicode is somehow not limited to specify `16 bits` for 1 Character or `64 bits` . It doesn’t matter if a 😘 takes `32 bit` or `128 bit`(_just example_) for user.  You, me and the other developers. Its 1 character afterall. 

__Hence, counting X bit memory to find number of characters went like PUFF! Length didn’t make sense.__ 

    let swiftyEmoji = (emojiOBJC as String)
    (emojiOBJC as String).characters.count   // 1
    (emojiOBJC as String).utf16.count        // 2 :: like the Objc length

If you think swift treats all characters as `32 bit` memory then thats wrong. We don’t care how it stores. The interface that swift provides is what we care.

    “go”.characters.count   //2

For us, developers, `"go"` is 2 character String. So is `"🍻👯"` is 2 character String. Swift manages the details for us. __String.characters__ provides the most unicode aware interface to us. However, feel free to visit the UTF16 and UTF8 view. Remember those are just a `VIEW` to the String. 

    “go”.utf16.count        //2
    “go”.utf8.count         //2

Okay lets move on to substring some Swifty String. And came swifty `Range`

## Swifty String Manipulation

    public struct Range<Bound : Comparable> {
        public let lowerBound: Bound
        public let upperBound: Bound
        public init(uncheckedBounds bounds: (lower: Bound, upper: Bound))
    }

So substring operation becomes:
    
    “Mr. X”.substring(from: <String.Index>)

Like we discussed we cannot just consider a Swift String as Array of Fixed Length Bits; like C `char*` where char is 8 bit. Thus, to not chop our emoji, we need a unicode safe way. Swift provides `String.Index`. You cannot get the `Int` from the `String.Index`. 

## The details 
- This will be updated in the coming days.

## Some Observations
- There is no public API that turns `Range<Bounds>` to `NSRange` primarily because `lowerBound` and `upperBound` are not `Int` and hence not convertible to `Int`.
- Its however trivial to make `Range` from `NSRange` although you cant make a range without actually specifying whose Range is this. In our case its String. 

        public extension NSRange {
            func toRange(forString: String) -> Range<String.Index> {
                 let lowerIndex = forString.index(forString.startIndex, offsetBy: location)
                 let upperIndex = forString.index(forString.startIndex, offsetBy: location + length)
                 return Range(uncheckedBounds: (lowerIndex, upperIndex))
            }
        }

- Like above you need to specify WhoseRange to create startIndex and endIndex.
- Range created for one string shouldn’t be used to manipulate another string directly. In essence, range is tightly owned and is applicable to its owner only.

        let a = “this iz it”let aor = a.range(of: “iz”)!
        var another = “th iz it”another.replaceSubrange(aor, with: “IS”) //OUTPUT =“thISz it”
    
- If we wanted to offset by +1 and use the same range then it can be done as such
    
        func rangeFrom(range: Range<String.Index>, forString: String, offset: Int) -> Range<String.Index> {
             let lowerIndex = forString.index(range.lowerBound, offsetBy: offset)
             let upperIndex = forString.index(range.upperBound, offsetBy: offset)
             returnRange(uncheckedBounds: (lowerIndex, upperIndex))
        }

- In the above code, range is calculated from another string. 
    
        let nr = rangeFrom(range: aor, forString: another, offset: 1)
        another.replaceSubrange(nr, with: “IS”) //OUTPUT = “th IS it”

- The above rangeFrom function will produce error such as these:
- When the lower/upperBounds of the fromString are computed which will fall outside of the range of the entire fromString.

        //fatal error: cannot decrement invalid index
        //fatal error: cannot increment beyond endIndex


### Should we use simple `NSRange` and `NSString` or go through swifty pain to be unicode aware?
- No support for 😎💔
- No support for Image Literals and other literals Apple will add on with time.
- Only supports with reasoning when UTF16 is used
- When there is a emoji like 💔  it will be counted as 2 characters if you use NSRange api and it gets worse if you try to insert a space in-between those 2 characters.

This all depends on your use case. If you are sure there are not special characters and emojis involved in the text then `UTF-16` suffices for english languages. However, the more cryptic and global your content is the more precaution is needed. After all, Swift does all the heavy lifting, you just don’t get the `Int` for `lowerbound` and `upperbound`.  Why not deal with it?
