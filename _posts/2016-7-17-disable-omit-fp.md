---		
layout: post
category: mono
title: MONO_DEBUG=disable-omit-fp
tagline: unwinding JITed methods
tags: [mono, profiling]
---
*You can skip whole story and just go to **TL;DR** if you are just interested into "MONO_DEBUG=disable-omit-fp".*

I have been working on [Xamarin Studio(MonoDevelop)](https://github.com/mono/monodevelop) for more than 2 years now. During this time, we made migration to Roslyn and UI is now much nicer looking. Debugging, writing code... Has less and less bugs every day. There are few more areas that needs some UX love like unit testing story, version control integration... But overall IDE is very nice to work with, except when you get [beachball](https://en.wikipedia.org/wiki/Spinning_pinwheel) spinning, waiting for file switch, laggy/jerky code typing... you get the idea.

<!--more-->

So now I'm on mission to eliminate these performance problems. I realize this might be very hard to get in garbage collected language, but it feels like our problems and delays are way bigger than just GC collections. Plus Mono Performance team is working on Concurrent GC which will make this even less of a problem if you are interested in this you might want to watch [Evovle 2016 Performance Adventures session](https://www.youtube.com/watch?v=Rsa54USiAJ4) by [Mark Probst](https://twitter.com/markprobst).

First thing I wanted to do in this area is figure out what code is causing beachball problems. I know that after few hours of working with IDE. It can get laggy and on some projects this can be bigger issue than others. I wanted tool to point users with this problems to and be able to get nice report of problem.

Requirements I had:

  * Work on unmodified Xamarin Studio, so no AOT or valgrind(which is too slow anyway)
  * Profile managed and unmanaged code(Mono profiler only handles managed code)
  * It has to be ran from command line(Instruments doesn't seem to support parameters)
  * Format that I can easily parse(Instruments, has some binary format)
  * Preferably doesn't need sudo(DTrace/Instruments need sudo)
  * "Unlimited" stack depth(Instruments seem to fail here)

I decided to use "sample" very simple command line tool built into macOS, which produces call tree with number of samples each stacktrace had while sampling app per thread.

In combination with converting JITed method address to full method name via Mono runtime helper method `string mono_pmip(long addr)`.
And conversion to FlameGraph result for UI thread is something [like this](~/assets/Xamarin_Studio_UIThreadHang.html) when opening some solution. We can see some time goes for GC but most goes for loading native libraries like libsvn and executing static constructors(.cctor) for editor extensions.

## TL;DR

But there was a problem ~50% of samples stack traces were cut off in middle of stack trace. So I had no idea what called this method that was causing performance problems, also [FlameGraph](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html) looked like useless mess. So I decided to figure out why this was happening after a lot of googling and trying to understand what was happening, learning a lot about stack unwinding. I found this awesome [blog post](http://blog.reverberate.org/2013/05/deep-wizardry-stack-unwinding.html) which pointed me at x64 RBP register which I noticed via lldb that Mono doesn't set. Searching Mono source code for RBP register I discovered there is logic to set this register but by default it's not set. Which let me to discover environment variable `MONO_DEBUG` which can be set to `disable-omit-fp` which runtime team member [Andi McClure](https://twitter.com/mcclure111) added recently via [this PR](https://github.com/mono/mono/pull/2755). I also noticed that now DTrace probes works much better, before probes were reporting bunch of `invalid address (0x0) in action` errors.