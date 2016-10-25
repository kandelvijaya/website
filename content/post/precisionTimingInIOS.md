+++
author = "kandelvijaya"
date = "2016-10-25T19:36:52+02:00"
description = "A comparison of fast and precise timing API for Swift."
tags = ["swift","iOS","Xcode"]
title = "Precision Timing in iOS & Swift"

+++

> Time is what we want most, but what we use worst. -_William Pen_

Timestamp is a very important issue we deal with in every single iOS/OSX project. Unlike timestamp, sometimes we want to measure method performance. Practically, i would use it for fun exploration. However, there are cases especially in games development where the precise time helps maintain consistent gameplay with scores. I explored a bit on how we can leverage the systems provided APIs to just get the current time stamp as precisely as possible. 

Two metrics to consider while going through are the precision in microseconds or beyond is better, i.e __100.0123212321321321__ seconds is better than __100.012321__ seconds. Second, how fast can we retrieve the time. 

Various API that one can use to get the time in iOS/ OSX. Not every method works on Linux.

### Obvious 
- NSDate().timeIntervalSince1970

### Foundation C API
- CFAbsoluteTimeGetCurrent()
- CACurrentMediaTime()
- ProcessInfo.processInfo.systemUptime

### Kernel level C API
- getTimeOfDay()
		
		 var timeOfDay = timeval()
         gettimeofday(&timeOfDay, nil)

- mach_absolute_time();
	The finest-grained timepiece available on the system. This is a lower level kernel call. The value depends on the processor and conversion is required to get time information. The process is somewhat tedious and C level. The precision is in Nanoseconds. 

	The way to set it up in Swift is as:

		        var info = mach_timebase_info()
        		guard mach_timebase_info(&info) == KERN_SUCCESS else { return -1 }
        		let currentTime = mach_absolute_time()
        		let nanos = currentTime * UInt64(info.numer) / UInt64(info.denom)

So the obvious winner for precision is `mach_absolute_time()`.

## Performance comparison:
These are calculated on MPB running i7 2.7 Ghz with 8 GB RAM. Take the ratio between different API calls into consideration. Actual run time for each call differs on the hardware and the environment used. 

	mach_absolute_time()			                : 0.90 µs/call 
	gettimeofday()					                : 1.10 µs/call
	CFAbsoluteTimeGetCurrent()		                : 1.13 µs/call
    ProcessInfo.processInfo.systemUptime            : 1.14 µs/call
	CACurrentMediaTime()			                : 1.15 µs/call
	NSDate().timeIntervalSince1970	                : 4.55 µs/call


As we saw, `mach_absolute_time()` is very fast. Compared to NSDate(), its 5 times faster. The reason is, NSDate has to be allocated and initialized before we access the time. For `mach_absolute_time()` we are making kernel level C API call.

### Timing Function Execution

One can time a block of code by using this utility function that I wrote to produce the above comparision result set. You can find the full list of utility [on this Gist](https://gist.github.com/kandelvijaya/8095de4ec37f225b7e3fee171d8909fb).

    func timeBlockWithMach(_ block: () -> Void) -> TimeInterval {
        var info = mach_timebase_info()
        guard mach_timebase_info(&info) == KERN_SUCCESS else { return -1 }
        
        let start = mach_absolute_time()
        //Block execution to time!
        block()                         
        let end = mach_absolute_time()
        
        let elapsed = end - start
        
        let nanos = elapsed * UInt64(info.numer) / UInt64(info.denom)
        return TimeInterval(nanos) / TimeInterval(NSEC_PER_SEC)
    }

## final words

Sometimes, `getTimeOfDay()` and `CFAbsoluteTimeGetCurrent()` tend to match the efficiency of `mach_absolute_time()`. This is at least what i found during my testing in playgrounds. However, `mach_absolute_time()` is always the fastest one. 

`CACurrentMediaTime()` is a wrapper around the most accurate time function in the system: mach_absolute_time(). `mach_absolute_time()` will give you a really accurate number, but it's based on the Mach absolute time unit which doesn't actually map to anything pesky humans think in (and every CPU has a different scale). That's why we have `CACurrentMediaTime()` to make our lives easier. Its 0.2 microsecond slow than the `mach_absolute_time()` because it runs the conversion on behalf of you from CPU ticks to the human parsable timestamp.

For the simple API, I would prefer `CFAbsoluteTimeGetCurrent()`. Using `mach_absolute_time()` would be ideal case for timing method execution. However, a caution to take is that, not always the execution time is the same. Hence, sampling for 100 or so times of a method executing time and averaging will provide a good heuristic. 

A [interesting blog post](https://bendodson.com/weblog/2013/01/29/ca-current-media-time/) does cover some pitfalls of both `CACurrentMediaTime()` and `mach_absolute_time()`. 

## Resources

A full playground utility and usage of all the above mentioned Timing API can be found on the [gist i created](https://gist.github.com/kandelvijaya/8095de4ec37f225b7e3fee171d8909fb). 

Hope you enjoy. Cheers!