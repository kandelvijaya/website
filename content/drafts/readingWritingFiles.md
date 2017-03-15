+++
author = "kandelvijaya"
date = "2017-03-12T17:16:52+01:00"
description = "GTX March 2017."
draft = "true"
tags = ["gtx", "git", "terminal", "XCode"]
title = "GTX March: Basic tips"

+++

# Reading and Writing Files from Playground

## What is sandboxing?
- Limiting the resources including storage, ports, permission to the application.
- Good to run a third party software without having the device infected.
- Every process runs in their own region and limits making no cross pollination, infection and so on.
- Web browser typically run in a sandbox mode.


# Playground
- Playground is Sandboxed so User's directories are not accessible directly.

# iOS specifics
- You cant access or write outside of sandbox.
- You can read file from the main bundle resource.
- Can you write into bundle's resource folder in iOS?
	__ANS__: No.
	__LONG__: You cannot write to this directory. To prevent tampering, the bundle directory is signed at installation time. Writing to this directory changes the signature and prevents your app from launching. You can, however, gain read-only access to any resources stored in the apps bundle.
- An iOS app may create additional directories in the `Documents`, `Library`, and `tmp` directories. You might do this to better organize the files in those locations. For more info, [see the documentation](https://developer.apple.com/library/content/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/FileSystemOverview/FileSystemOverview.html)

# iOS Simulators
- They can read file from any location that can be located by the user. For example: a mock json located in Desktop can be used to render contents in the simulator instead of downloading each time.
- Writing file to any place?







# OSX specifics
- The OS X file system is designed for Macintosh computers, where both users and software have access to the file system.
- Apps installed from the app store are sandboxed. Third party apps may or maynot be.
-


# The number of file reading APIs:
- NSFileManager
- NSBundle `pathForResource`
- Direct String `string(contentsOfFile: encoding:)`
- NSSearchPathForDirectoriesInDomains

Although it seems like there are lots of options to read/write files, its not ture. Basically, there are two ways to get the file. If the directory being referred is a standard directory then the system provides bunch of API, like NSBundle and NSFileManager. Whereas if the directory or file is not a well known location then the user is required to build the path and then use the appropriate APIs.
