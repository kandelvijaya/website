---
title: "Story of FastDiff."
description: "A delayed project; learnings and diff library"
date: 2020-03-03T22:57:53+02:00
author: "kandelvijaya"
tags: [ "swift", "diff", "toolbox", "experience"]
---


# Table of Contents

1.  [Diff: Just because I can.](#org0b1bb1c)
    1.  [FastDiff](#org73b30a0)
    2.  [Why FastDiff](#org0f95efd)
    3.  [OrgNoteApp.](#orge003977)
    4.  [Back to work](#org70e3a9e)
    5.  [let's try FastDiff](#org7728c9f)
    6.  [The test](#orgc4e0ac4)
    7.  [Conclusion](#orgc4dffd8)
    8.  [PS](#orgafda51e)



<a id="org0b1bb1c"></a>

# Diff: Just because I can.

I failed to publish OrgNoteApp. I created many small modules in the development process. I call them items in my engineering toolbox. Recently,
one of the modules (diffing) is shipped to millions of customers. It's part success and part learning. Enter FastDiff and its little story.


<a id="org73b30a0"></a>

## FastDiff

FastDiff is a diffing library much like any other diff algorithm implementation. It only differs from the rest in 3 regards:

1.  It can diff nested collections. Trees and Graphs.
2.  It uses Paul Heckel's algorithm. Unlike solving LCS; its running time is linear.  [A technique for isolating differences between files](https://dl.acm.org/doi/abs/10.1145/359460.359467)
3.  It's Swift implementation.

You can find the project here [Github Project.](https://github.com/kandelvijaya/FastDiff)


<a id="org0f95efd"></a>

## Why FastDiff

I was working on a UI text editor to accompany OrgMode. OrgMode is a simple emacs mode (text editor) yet powerful due to minimal structure.
Of course, if you use Org-Mode you can manage calendar, events, Gantt charts and todos; write a book and whatnot. The primary structure is a nested heading.
Simple enough. I write almost everything on it. My work log, this blog, papers, and everything and sometimes even code. Since org mode runs on emacs, which requires Mac/Linux/Win.
I wanted an iOS client on the go; sync-ed with remote org notes. This is the inception of OrgNoteApp. In the process, I had to tackle multiple problems.

1.  Is there OrgMode file parser in Swift? No. I created a parser just for it. I was inspired by Haskell and implemented [Parser-Combinator](https://github.com/kandelvijaya/SwiftParserCombinator) & [OrgSwiftParser](https://github.com/kandelvijaya/OrgSwiftParser) using swift.
    This is used for Parsing raw org mode files to a known structure.

2.  I believe UI is a function of State. I am a functional programming enthusiast and nothing beats declarative programming where UI is a function of Model/State.

I needed a declarative list where I could just give it \`[OrgModelItem]\` and it would render it. The moment I edited, added or deleted 1 entry, I would change/create the underlying model. The list would get new models. The list would diff and apply the changes. This was the plan. For this diff was important. Not only that, I needed
to diff on nested level. Would you replace the entire heading if a content of one of its branch sub-heading changes? Of course not. FastDiff helps pinpoint exactly what changed
no matter how deep it is.


<a id="orge003977"></a>

## OrgNoteApp.

I added 3 pieces, FastDiff, [DeclarativeTableView](https://github.com/kandelvijaya/DeclarativeTableView) and Parse-Combinator. The UI was ready. I used git client on iOS to sync org files from my private git server. [ELisp code to sync](https://github.com/kandelvijaya/orgsync)

1.  Every file I opened on my work/personal Mac on the given folder; I had ELisp code to push the changes to my private orgnote repo. This would happen on save. Using git.
2.  On every OrgNoteApp open on iOS, I would sync (git fetch or git pull) the master branch and allow a user to navigate the directory.

It kind of worked. I later used collection view as tableview had issues with large batch updates and animation. CollectionView solved it but there was this 1 weird bug; on
some header collapse; the first or the last index would jump beyond their position during an animation. I struggled with it but couldn't just get it right.

This is where I slowly moved into actual OrgModeTextEditor idea (abandoning the UI due to that 1 bad animation & building UI editor is complex than I anticipated).
I moved into creating yet another text editor with Org syntax highlighting. Mobile phone didn't seem like a place to actually write long texts and then I slowly phased out of this project.
Of course, working on Text Editor, I learn a great deal about the power of NSTextStorage used by TextView, I probably overused AttributedString. It was fun! So what happened?


<a id="org70e3a9e"></a>

## Back to work

At work, I and my team are building AppCraft client for iOS/Android. I lead a mobile team of 4 (3 + me.) iOS and 3 Android engineers. We are responsible to render/draw/produce native UI given a DSL
that we know. The DSL contains

1.  composition of component where each component is somewhat like either Layout, List, Text, Button, Image, Price and so on.
2.  data for either the composition of component or the component itself
3.  tracking info
4.  actions and behaviors when user interacts with it.

We have a ELM inspired architecture (thanks to React and later [Objc.io ELM talk](https://talk.objc.io/episodes/S01E66-the-elm-architecture-part-1)). This means each screen has an \`AppState\` which is the source of truth for what the UI is.
UIViewController typically can be analogous to a Screen. Another way of
putting it is: UI is the function of State. (Those of you with some experience with SwiftUI or ELM itself would be right at home).

For one time rendering (initial) this is fine. However, imagine trying to tap a wishlist button. This changes some part of AppState. Now we have to reflect that on UI. Change the button color or
make it selected/deselected. However since UI = f(AppState); let's just redraw all-over again. This is what we had for some time. However, this wastes a lot of precious CPU & Memory. For smaller screens,
this was even not a problem. \* It's not hardware, its software that is getting slower.\* But we knew deep down, we just had to redraw the thing that changed and nothing else.
Enter the requirement to Diff & Patch. Just be aware the DSL (JSON) is a tree (of course JSON) structure. Diffing tree is different than diffing just a one-dimensional collection ([Int] vs [Int]).


<a id="org7728c9f"></a>

## let's try FastDiff

Home page is a giant vertical list of components. For us. Each component can themselves be another list. Enter recursion or inception. A tap on favorite/wishlist icon, would
first change AppState, we then would drop the whole page and re-render it. Thats plain waste. We knew it. We \`just\` need to diff and apply patch. Of course, easy to say but there didn't exist a
general-purpose out-of-box nested list (tree) diffing library. Even if it did, what about patching. We don't have Virtual-Dom (out-of-box library) unlike in Web. When nothing fits, I enjoy putting
myself to test. That is when I decided to re-factor and add in FastDiff. I quickly ran into patching problem.

FastDiff worked as expected. I wrote quick performance unit tests over a big AppState model change. It verified its fine/fast. AnyCodable or Codable & hashValue computation shows up in
instruments on big diffs. Nothing with FastDiff. After that, I had to tag each Node/View with component-Id. During patch, I would try to find the node with update-Id; simple depth first search (DFS/ recursion) works.
We used BFS (breadth first search) for local optimization.

I wrote unit tests for patching. I usually do them first and then code the implementation second. Sometimes the other way. FastDiff has 100% code coverage. Now Patch had good number tests to assert.
With collectionView / List, patching by finding node with id doesn't work when the node to be patched is not on screen. We also patch the model in this case. However this patching is not general-purpose,
surely lacks what we haven't anticipated yet. Thus it's not a separate library out (yet).


<a id="orgc4e0ac4"></a>

## The test

FastDiff now sits at the core of rendering. Every state change triggers \`ModelView(newAppState) -> UIView\`. Within it lies the diff and patch. Everything has to pass through it. All the time.
Pagination is just addition of items to a list. Refresh is just a replacement of items on the list. However, not all changes to AppState needs UI change. And this is handled in the patching part. Now to the test.

We shipped out home to a small percentage of our user base. My team worked hard for this day. One thing I was particularly fond of is that FastDiff continues to deliver. Once there was a bug on navigating from
AppCraft screen to any other and coming back. The main thread would freeze for half a second. Immediately, I suspected it must be diffing all-over again and somehow took more time. Upon investigation, we
found a different code path to have caused this. Ahh relief.


<a id="orgc4dffd8"></a>

## Conclusion

This post is not merely about FastDiff. If you are interested in it please refer to the project code base documentation or this talk I gave [ModelUpdate talk on MobiConf 2019](https://www.youtube.com/watch?v=wRfZs1ukuws).

Creating small isolated modules with clear interface and focus helped me open source around 4 packages when working on OrgNote. The app hasn't seen the day of light until now; its smaller components
have been tested and refined and some have been used in a completely different scenario than originally thought of.

During the OrgNoteApp, I had to create file explorer. Simple navigation; list of files and folder. Tap folder, it recursively pushes folder explorer. I loved this recursive problem solving and suggested
this as one interesting task for interview candidates. I'm pretty sure some of the newer iOS mates at the company, went through this.

Point is: there are patterns and smaller problems waiting to be solved everywhere. If we just pay attention to those and isolate them we can have a tool-set of smaller modules. I like to think them as my
toolbox; similar to carpenter toolbox. I believe each one of us solves something that matters, keep repeating them again somewhere. When that happens should we remorse that we didn't extract that part of code
last time and thus do it all over OR do we reach the toolbox and grab a solution.


<a id="orgafda51e"></a>

## PS

A big thanks to my team Mobile@AppCraft for being okay with me refactoring the code base multiple times massively & then supporting it. Thank you to @Ramy for giving me the idea that I might just write a
post about it. And lastly thank you for reading all the way. I hope you like the toolbox idea. Lastly, if you do want to know more about the project, even better want to join my team of iOS and Android engineers;
I would be happy to help you join this amazing team. Check [Zaladno Job Portal](https://grnh.se/0c5a1d5c1) for more info.

Thank you for reading all the way. Hope it was helpful.
