+++
author = "kandelvijaya"
date = "2016-10-10T21:57:04+02:00"
description = "XcodeFormatter: What and How to write consistent styled code?"
tags = ["Swift3", "iOS Engineering", "Xcode8"]
title = "Xcode8 ZStyle Code Formatter: How to write consistent styled code?"

+++

Yet again somebody missed to insert a empty line before the end of file, I missed to provide a empty space after dictionary Key  `[AnyHashable:Any] ` and you might miss to leave any of these kinds of code:

    //.......
        return data
    }
    func compute(a:Int,b:Int)->Int{
    //.......
 
Which should have been:

    //.......
    return data
    }   
      
    func compute(a: Int, b: Int) -> Int {
    //.......
  
So you see where I'm heading with this.

## Bigger Picture
To just briefly explain the scenario, we are 12 iOS Engineers working harder than before (pun intended), cramming hundreds of line of code daily.
We have bunch of features to roll out, release deadline to meet and a generation next to come and read the code we wrote with great thought.
We couldn't possibly achieve if, each of us, with unique style  `(tabs vs spaces) ` worked on. Thus, we have a Zalando iOS coding guidelines.
Which keeps us in the same fashion. But like again, there is no way to actually force the project to have a strict guidelines and we humans are error prone.

## Problem
Some empty line or no empty spaces sneaks in the code base. We need a tool to correct out the our style to Zalando iOS style.
After all we care, because mainly we are a fashionable technology company. All right!

## Solutions Available
 1. [Swift Lint](https://github.com/realm/SwiftLint)
    * Its a good library but we have some serious issue with it.
    * We have a large build time and this adds some more.
    * We already have hundreds of TODO and FIXME that are signaling warnings. This adds even more errors/warnings.
    * The tool has limited possibility of correction. Our solution is not to give things to the devs to correct. They are already busy. It is to work for them.
    

## Our Attempt
*  __Before__ WWDC 2016
    - Integrate SwiftLint and use regex to auto correct. But still the disadvantage was bigger than advantage.
    - Abandoned!
    

*  __After__ WWDC 2016
    - Source Kit Extension API announced.
    - I started secretly using this extension to auto correct only things we cared.
    - The __birth of XcodeFormatter!__
    
## XcodeFormatter

* Its a file based auto formatter. It wont warn or error. It just silently formats to the ZStyle. Pretty silently.
* Reach out to the Editor -> XFormatter -> ...(options) 
* When you absolutely don't want to, then don't. 
* If you want to, which you should, then provide a shortcut or if you are mouse ninja then reach in the Editor  -> 
* Done! 
 
## Technical Side
I prefer to run down the tech side by data flow or message flow.
 
  * When user presses, Editor  -> XcodeFormatter  -> Correct All: Xcode sends the file content to our App Extension. 
  * The app extension, choses a certain formatting method based on the command user clicked. 
  * Inside the app extension, there are  `RegexMatch ` and  `CodeBlockAnalyzer ` which stays at the heart of matching the wrong style. 
  * Inside the app extension, there are  `MatchCorrection ` and  `EmptyLineCorrection ` which stays at the heart of correcting the matched code. 
  *  `MatchCorrectionInfo ` is used to correct the match in case of  `MatchCorrection `. Idea is to replace the Capture Group.  `MatchCorrectionInfo ` provides a  `[Int: String] ` with int for index of Capture Group and String for what to replace the found match with. 
  *  `EmptyLineCorrection ` does not need a correction rule as it trivially inserts/removes empty line above/below each  `CodePosition ` passed into correct. 
  * All the matches and analyzation happens if the current character in code is not inside a  `//Comment ` or  `"String quote" ` 
  * When everything is done, the  `XCSourceEditorExtension ` gets the corrected data, uses it to put the changes back into the  `NSMutableArray ` of lines Xcode provided us initially. 
  * This change is then reflected in Xcode once the completion handler is called. 
  * Boom! Done! 
 
### A more deeper level working will be covered when this article is updated.
For details, check the readme file on github. [XcodeFormatter On Github](https://github.com/kandelvijaya/XcodeFormatter)


## Cheers!