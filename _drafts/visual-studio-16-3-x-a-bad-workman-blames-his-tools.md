---
layout: post
title: Visual Studio 16.3.x - A bad workman blames his tools?
date: 2019-11-06 00:00:00 +0000
summary: Nothing like moving the goalposts when your in the middle of something
minutes: 2

---
I must confess - from the title and abstract of this post it sounds like I'm being petulant and unnecessarily annoyed.  However, it's important to remember to look at the tools you're using before you go bald pulling out your hair.

At the time of writing, I'm working on an Azure Functions V2 C# project in targeting the .NET Core 2.2 SDK.   There are various dependencies -  but they, along with everything else is referenced as a `PackageReference` in the `.csproj` file.

So far so good - everything working well - no compilation issues - right up until the point I accidentally upgraded Visual Studio some time in late September.  I think i was struggling with another project that suffered from Visual Studio not being able to perform 'Edit and Continue' and I happily upgraded away without really giving it a second thought.

So, back to my functions project a few days later - fire up Visual Studio 16.3.x and all of a sudden compilation fails