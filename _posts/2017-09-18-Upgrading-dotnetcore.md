---
layout: post
title: Upgrading dotnetcore broke NuGet in VS for Mac
---

It has been about a full year since I posted anything here, but yesterday I had quite an interesting journey while I was trying to update dotnetcore. I thought it worth a blog post, because I’m pretty sure others WILL encounter the same issue eventually.

So a little background info. I’m working on one of my projects, which involves an **angular4** frontend with a **dotnetcore** API backend. Pretty standard setup. Yesterday I decided that it’s time to do the **1.1 -> 2.0** upgrade.

It’s supposed to be really simple, just a matter of changing the target framework to netcoreapp2.0 for the API, and netstandard2.0 for the libraries, right?
I went ahead and did that. The dotnet restore and dotnet build  cli commands went green, everything looked fine.

Until I wanted to open the solution **Visual Studio for Mac 7.2** (*aka. Monodevelop*). That’s what I prefer for .NET development on my macbook. I could have used VS Code, but imho the debugging experience is still generally better with VS Mac.

After opening the solution, it tried to restore the NuGet packages as always. This time though, it resulted in the following errors:

```
Unable to find package Microsoft.AspNetCore.Hosting.Server.Abstractions with version (>= 2.0.0)
Found 11 version(s) in api.nuget.org [ Nearest version: 2.0.0-preview1-final
Unable to find package Microsoft.AspNetCore.Hosting.Http.Features with version (>= 2.0.0)
Found 11 version(s) in api.nuget.org [ Nearest version: 2.0.0-preview1-final

<...more omitted text with similar errors...>

Restore failed for 'Microsoft.AspNetCore.Server.Abstractions (>= 2.0.0)
Restore failed for 'Microsoft.AspNetCore.Http.Features (>= 2.0.0)
```

Just a sidenote, VS Mac uses its own nuget package manager instead of calling the cli tool (as opposed to VS Code)

What happened here? It looks like it cannot find the required packages for dotnetcore 2.0.
Unfortunately, I don’t have the full package manager log, but I can tell you that it used the [https://api.nuget.org/v3/index.json](https://api.nuget.org/v3/index.json) NuGet feed to look up the package information. I went ahead and opened the urls in my browser, and low and behold...the proper versions were there!

What the hell? The package manager sent the Get requests, resolved the package information (which was there on the server), and still couldn’t find the proper versions. _Weird._

Honestly, I will spare you the details of going mad and pulling my hair out, and will jump to the solution.

Running **nuget locals all -list** listed all the package caches on my system:
```
http-cache: /Users/IWC-Tamas/.local/share/NuGet/v3-cache
global-packages: /Users/IWC-Tamas/.nuget/packages/
temp: /var/folders/b2/54h2sx5x44524r_f385mgdcc0000gp/T/NuGetScratch
```

After I removed the complete http cache folder, and re-ran the NuGet restore in VS Mac, it went green. YAY!

The package manager log was a red herring. Even though it looked like it worked from the real source of truth, it used an old cached version of the package information.

I hope google will help me spread the word and spare you those 3 hours it took me to figure this out. Feel free to leave a comment if so!
