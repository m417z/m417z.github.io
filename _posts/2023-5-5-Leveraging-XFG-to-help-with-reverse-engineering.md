---
layout: post
title: Leveraging XFG to help with reverse engineering
image: /images/Leveraging-XFG-to-help-with-reverse-engineering/Example-function-in-x64dbg.png
---

[Microsoft eXtended Flow Guard (XFG)](https://en.wikipedia.org/wiki/Control-flow_integrity#Microsoft_eXtended_Flow_Guard) is a control-flow integrity (CFI) technique that extends CFG with function call signatures. It was presented by Microsoft in 2019, and it's an interesting mitigation, but this blog post isn't going to discuss its security implications. Instead, I'm going to show how XFG can be used to help with reverse engineering.

## At first glance, just a nuisance

The idea of XFG is to add a signature before each function that can be invoked indirectly, and to verify that the signature is correct before executing the function. Each XFG signature is 8 bytes long, and is located right before the target function. Since the signature is located in the code section, a disassembler might get confused and show it as random instructions. That's what was happening with x64dbg, and while it's possible to mark the signature as a 8-byte integer manually, I got annoyed enough by doing that for every function and created a plugin that does it automatically: [XFG Marker](https://github.com/m417z/x64dbg-xfg-marker).

![x64dbg XFG Marker]({{ site.baseurl }}/images/Leveraging-XFG-to-help-with-reverse-engineering/x64dbg-XFG-Marker.gif)

## Indirect calls are a nuisance, too

Talking about indirect calls, they're not convenient for reverse engineering. Unlike direct function calls for which the target function is available at a glance, that's not the case for indirect calls. Looking at an indirect call in a disassembly listing, all you can see is that some function is called. To find out more, you either have to do some analysis, or resort to debugging. Either way, it's not as quick and convenient as looking at a direct call.

## XFG to the rescue

It took me a while to notice what now seems to be obvious - the indirect call XFG signature can be used to identify the possible target functions that may be called. The function call signature is [based on the function prototype](https://blog.quarkslab.com/how-the-msvc-compiler-generates-xfg-function-prototype-hashes.html), and is designed to be as specific as possible.

I added this functionality to the [XFG Marker](https://github.com/m417z/x64dbg-xfg-marker) plugin, and the result looks quite promising. Here's an example:

![Example function in x64dbg]({{ site.baseurl }}/images/Leveraging-XFG-to-help-with-reverse-engineering/Example-function-in-x64dbg.png)

As you can see, even though all four calls on the screenshot are indirect, the relevant functions can be deduced via XFG signatures, and it's now much easier to understand what the code does at a glance.

In addition to adding the comments, the plugin adds xrefs between the signature commands and the target functions, so you can easily jump to/from the target functions in the debugger.

As far as I know, no other reverse engineering tools leverage XFG signatures, which makes x64dbg together with the XFG Marker plugin a great choice for some reverse engineering tasks.
