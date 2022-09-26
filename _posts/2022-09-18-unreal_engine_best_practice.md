---
layout: post
title: UE4 Best Practices
modified:
categories: 
excerpt: UnrealEngine
tags: [Notes]
image:
  feature: note.jpg
  thumb: thumb1.jpg
date: 2022-08-08T02:54:48+05:30
---

## UE4 Best Practices

 For the past couple of years working in unreal engine i've been collecting what i consider to be the best practices of how to work in it uh and i made a document of this which i figure might be good to share

TODO fix links:

Derived Data Cache (DDC)
This cache allows sharing of the cost to compile shaders and other operations. It’s a must when working in a team to prevent hours of waiting when working locally. It doesn’t make sense to use when working remotely because it would take longer to download than to generate, however you can specify the location of a local and shared DDC to prevent the warning messages from popping up.

Here’s Epic’s documentation:
Derived Data Cache
Team Setup (while in office)
The project should be set up to use a shared cache. An example setup used in Slingshot (DefaultEngine.ini):

[DerivedDataBackendGraph]
MinimumDaysToKeepFile=7
Root=(Type=KeyLength, Length=120, Inner=AsyncPut)
AsyncPut=(Type=AsyncPut, Inner=Hierarchy)
Hierarchy=(Type=Hierarchical, Inner=Boot, Inner=Pak, Inner=EnginePak, Inner=Local, Inner=Shared)
Boot=(Type=Boot, Filename="%GAMEDIR%DerivedDataCache/Boot.ddc", MaxCacheSize=512)
Local=(Type=FileSystem, ReadOnly=false, Clean=false, Flush=false, PurgeTransient=true, DeleteUnused=true, UnusedFileAge=34, FoldersToClean=-1, Path=%ENGINEDIR%DerivedDataCache, EnvPathOverride=UE-LocalDataCachePath, EditorOverrideSetting=LocalDerivedDataCache)
Shared=(Type=FileSystem, ReadOnly=false, Clean=false, Flush=false, DeleteUnused=true, UnusedFileAge=10, FoldersToClean=10, MaxFileChecksPerSec=1, Path=\\Slingshot\ddc, EnvPathOverride=UE-SharedDataCachePath, EditorOverrideSetting=SharedDerivedDataCache)
AltShared=(Type=FileSystem, ReadOnly=true, Clean=false, Flush=false, DeleteUnused=true, UnusedFileAge=23, FoldersToClean=10, MaxFileChecksPerSec=1, Path=\\Slingshot\ddc2, EnvPathOverride=UE-SharedDataCachePath2)
Pak=(Type=ReadPak, Filename="%GAMEDIR%DerivedDataCache/DDC.ddp")
EnginePak=(Type=ReadPak, Filename=%ENGINEDIR%DerivedDataCache/DDC.ddp)

Set up a machine on the network so that it has shared folders with everyone read/write access, and set the path in the DefaultEngine.ini as shown.
Locally (remote work)
When working remotely, it doesn’t make sense to try and use a shared DDC, because downloading it through the VPN would be slower than generating it yourself locally. We will disable the shader DDC to prevent duplicating data in the following steps.
By default, the engine will set up a local DDC in Users\username\AppData\Local\UnrealEngine\Common. I prefer having it somewhere easy to find (and delete), but you can leave it there if you don’t mind that location.
Working from home, I suggest using the following environment variables, which specifies the location of the local DDC and has no shared DDC for all projects. You can leave out the local DDC one if you are fine with where it’s currently located.
