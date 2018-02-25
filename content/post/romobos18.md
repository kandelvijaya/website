---
title: "Romobos 2018"
date: 2018-02-18T12:20:27+01:00
author: "kandelvijaya"
description: "Explaining Monadic types"
tags: ["haskell", "swift", "functional programming", "fp"]
draft: false
---

# Table of Contents

1.  [Romobos 2018: My take!](#orgff0627b)
    1.  [Day 1: Presentation day](#orgc79a63a)
        1.  [John Sundell on CoreAnimation](#org5682cd8)
        2.  [Paul Ardeleanu on Test for GTFO](#org74de003)
        3.  [Simone Civetta on Kotlin for cross platform](#orgb8c24d5)
        4.  [Others](#org698c72d)
    2.  [Day 3: The workshop](#org8e4f2c2)
    3.  [Day 3: Back to Berlin](#org56a6a70)


<a id="orgff0627b"></a>

# Romobos 2018: My take!


<a id="orgc79a63a"></a>

## Day 1: Presentation day

RoMobos, Romania's biggest Mobile operating System conference. I'm here today on the first day of the 
conference. I will be preseting about Monads in Swift today and doing a workshop on the same topic 
tomorrow. I wanted to keep my thoughts organised and keep note on key takeaways and this is my effort. 
This is opinionated summary which I want to publish someday.

On the flight from Berlin to Cluj, I met Pedro, iOS engineer at Shopify. We talked about improving build 
systems and functional programming. He is a nice guy and has good knowledge 
on build systems. Reach out to him and say hi here <https://twitter.com/pepibumur>. Interesting that Romania's 
passport control held me asking many questions for 10 minutes. I met couple of other engineers on the way 
to hotel on the cab. It was 2 AM when we reached the hotel so I went straight to bed.

On breakfast, I met Danny (Android Enginner). We had small talk. He is doing a workshop on TDD. You can 
reach out to him <https://twitter.com/PreusslerBerlin>. We met few months back at UAMobile conference. 

The conference started with Human Computer Interaction which to be honest did have good intent but the 
presentation could be made much interesting. Followed was a interesting presentation on Beyond Engineering
by Oleg Anghelov which demonstrated difference between working in out-sourcing and product company. To me the 
take-away was to give a damn about the product and the company you are working. Engineering is still the top priotity. Oleg did a good job making us stay awake and interested at all times. 

Now I'm heading to meet John Sundell and see his presentation. The man behind <https://swiftbysundell.com>. 

*/ I'm back to my room skipping the final presentation. Im sorry but I got tons of content to process in a 
/* day. 


<a id="org5682cd8"></a>

### Takeaways from John Sundell presentation

1.  CoreAnimation is great intermediate compromise for playing with CALayer, creating icons, assets and animation.
2.  There exists a CAReplicatorLayer (I didn't knew before) that replicates instances. It allows one to 
    apply transformation function on each. This way one can build a loading indicator in some handful 
    lines of code.
3.  ImagineEngine and the code structure. <https://github.com/JohnSundell/ImagineEngine>
    
    I talked to John about how is he managing to write a post each week, creating podcast and working 
    on open source projects alongside his job. It was a nice conversation and I will follow up with hom on 
    that.


<a id="org74de003"></a>

### Paul Ardeleanu on "Test or GTFO – a guide on how to write better Swift for iOS"

Paul had amazing presentation on TDD, practical tips and todos. Despite the fact that I have been doing TDD for some 
time, I could get some real takeaways. One of them was to not use literal values in test. For instance, 
not to put "ThatString" literal on some struct Initializer or function parameter while under test. Get 
a random generator and use it. Similar is the case for using Factory pattern. The other thing I learned 
is how one can test existence of correct storyboard and Xcode configuration in unit test. Also 
how to test NSLayoutConstraints in unit tests. Its pretty cleaver. Good talk. I enjoyed a lot.


<a id="orgb8c24d5"></a>

### Simone Civetta on "Reusable Cross-Platform Frameworks with Kotlin/Native"

Simone did a interesting talk where he showed how one can use kotlin to write business/domain logic and produce 
iOS framework and Android package which then can be consumed by both platform. I was surprised to know 
JetBrains is working hard to make Kotlin interoperable with Swift/Obj-C. Good step! 


<a id="org698c72d"></a>

### Others

There were couple of talks that were generic and common.
There was a talk on Security which had solid content on how to preserve users secure data from leaking. 
For instance blurring the screenshot of credit card entry during the task switcher or disallowing 
third party keyboards to log everything. I completely missed Fumiya Nakamura on “Optimizing iOS Development” which I wanted to attend. I was busy preparing for my talk.


<a id="org8e4f2c2"></a>

## Day 3: The workshop
{{< tweet 964810856704954369 >}}

Needless to say the previous night we partyed. Van Der Lee and I started with Țuică (Romanian fire water), 
later joined by the likes of John, Pedro, Simione, Ostop, Nakamura-san and Assaf. Later we went to club, 
then to a bar and then back to Doner (as expected) place before heading to bed pretty late at night. The 
following day (today, 16 Feb 2018) I had a 3.5 hours to explain what the heck is Monad in Swift. Needless 
to confess I had hoped for a open ended discussion model workshop, I had to do some last minute preparation.  

I planned the workshop in the morning after late breakfast therby missing one of the most interesting 
session on Build System by Pedro. Explaining Monad is hard, although is based around the generalization 
for composing 2 functions where each one has a signature of a -> M b. Where M stands for any kind of Contextaul type (i.e. Optional, Result, Collection)

    (a -> M b) -> (b -> M c) -> M c

I often struggled in times to give proper example that would be used in real life. I had to 
use MAD.A.DAM game. I gave attendees some space to think, time to implement map and flatMap on Result and List types. 
I did't bother asking them to model Monad as protocol like I did back in SwiftAveiro (It was fun exercise). This is such a nice brain teaser. At times, I found people 
dropping off and had to get them back. I was constantly pushing code changes so people could follow. They were interested to stick around and see what monad was. Several people 
did manage to get the hang of it and struggle through the 3.5 hour to see and create a Monad for themselves. 
Seeing that satisfaction on their face is rewarding. Next time around I would love to increase the smiling face at the end. 

Later that evening, we went to old city center and enjoyed authentic romanian cousine at the "wheels" restaurant. Bogdan, Iaran
and Mircha were wonderful organisers. We then went for walk, see the roman wall remains and some 
other monuments of history. Later we went to Enigma bar which was decorated interestingly with a rounded bulbs on 
ceiling synchornized to move vertically, a giant wall clock with mechanical wheels attached to motor and a robot that 
was pedalling a cycle all night long. Soon we were back to the hotel, already past 12 midnight. 


<a id="org56a6a70"></a>

## Day 3: Back to Berlin

Today morning, John, I and Nakamura-san planned for city sightseeing in the daylight. I joined them for few hour 
as I had a plane to catch at 1PM. Romania being in EU means I could use 4g with my existing data plan and therby 
Uber. Uber was pretty cheap for transportation. 

Finally, I parted John and Nakamura-san, met with Alex and Danny and headed towards the Airport. Oh Alex, works 
on Microsoft's TODO iOS app in Berlin (Next building to my previous apartment was). And now, I'm trying to wrap 
up my experience from Romobos conference as I fly over somewhere between Cluj and Munich. 

Thank you `@Romobos` for inviting me, orginising a wonderful conference. Great work guys! Thank you all the 
participant who might be reading this for enduring Monads (talk on first day, workshop on second); I believe 
you learned something. Thank you all the speakers turned friends, most of whom I never met before, for having good
conversation, drinks and fun.

### Materials
1. Slides 
    {{< speakerdeck 4f90f7572a8042d2b0dbde1cf0b36e85 >}}
2. [Workshop material for Monadic Types can be found here](https://github.com/kandelvijaya/mobosFPWorkshop)
3. Presentation (from UAMobile conf)
    {{< youtube 6UHGDrFcCdA >}}