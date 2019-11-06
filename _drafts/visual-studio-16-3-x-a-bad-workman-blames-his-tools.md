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

So, back to my functions project a few days later - fire up Visual Studio 16.3.x and all of a sudden compilation fails with:

![](/uploads/2019/11/06/azure functions 22sdk with 30sdk build failure.PNG)

I tried everything, rolling back git commits, changing PackageReferences, removing chunks of code, starting a new project, the works!

I finally figured out that one of the major changes to Visual Studio 16.3.x was the introduction and defaulting of the .NET Core 3.0 SDK and build tools.  Turns out while mostly backward compatible, the publish targets copy files to the wrong folder.  It also turns out I'm not alone:

[https://github.com/Azure/azure-functions-vs-build-sdk/issues/333](https://github.com/Azure/azure-functions-vs-build-sdk/issues/333 "https://github.com/Azure/azure-functions-vs-build-sdk/issues/333")

My fix included adding a `global.json` file to the folder containing my `.sln` file containing the following:

```
    {
      "sdk": {
        "version": "2.2.402"
      }
    }
```

I also had to ensure that a `functions.deps.json` file is copied as a PostBuildEvent / AfterPublish Target to retain explicit references to dependencies when the functions project in a linux Docker container:

[https://github.com/Azure/azure-functions-host/issues/3568#issuecomment-544901620](https://github.com/Azure/azure-functions-host/issues/3568#issuecomment-544901620)