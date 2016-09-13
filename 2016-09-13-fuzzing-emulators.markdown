---
layout: post
title:  "The Importance of Fuzzing…Emulators?"
date:   2016-09-13 15:00:00
authors:
- endrift
tags:
- debugging
---
{% hero_big afl.png %}

Anyone familiar with computer security should be familiar with the concept of [fuzzing](https://en.wikipedia.org/wiki/Fuzz_testing). You throw garbage data at a program, over and over again, to see if it crashes. If it does, you might have a security issue. It's a great way to do automated security testing of software, and has uncovered countless critical issues in software across the board. A popular fuzzing framework, [American Fuzzy Lop](http://lcamtuf.coredump.cx/afl/) (usually called afl or afl-fuzz for short), even has a "_trophy case_" for only a small percentage of the bugs it has uncovered—and there are over 150 bugs listed!

Although usually not very intelligent, and limited in the scope of the bugs it can find, fuzzing is a common and effective practice for finding security bugs in software that is complex enough for issues to not be immediately obvious upon source inspection. Being a stochastic process, fuzzing can take a lot of time and careful selection of input cases (for mutational fuzzers) to produce good results. Conversely, it can also be left running processing as a background task for weeks or months with little interaction. As such, fuzzing is often employed in commonly deployed libraries such as libPNG, and widely used software such as Flash. A myriad of different projects use fuzzing to help find bugs, especially as software security comes more to the forefront of engineers' minds.<!--more-->

However, not all software is generally considered to be an important target of security research. Some software routinely interacts with untrusted online sources, such as web browsers. These are obvious and frequent targets for cracking. Some software never touches the network or downloaded files at all. But what I work on consists of an uncomfortable middle ground.

Video game emulators have the common, if not dubious, use case of tons of people downloading untrusted files (either ROMs, save games or savestates) from sketchy sources (hey look at all these fake download buttons on this ROM site!) and then loading them in the emulator without a second thought. Although not an obvious target to a hacker, as there are higher profile projects with a wider scope, it stills seems an obvious entry point to patch.

There has been increasing awareness of security implications of emulators recently. The ancient emulator, ZSNES, which was written before the concept of software security was on the forefront of engineers' minds, recently had publish a ROM that would [pop open a web page in an external program](https://www.youtube.com/watch?v=Q3SOYneC7mU) with no user interaction other than loading it. Just yesterday, a buffer overflow in the cheat file parsing of VisualBoyAdvance was published in [a rather satirical video](https://www.youtube.com/watch?v=L-L8qWpd_74). Neither of these emulators are actively maintained anymore, but are still widely used. This compounds the problem, as these vulnerabilities may never be patched in a widely used version of the software.

{% youtube L-L8qWpd_74 520 300 %}

To help stem this problem, I wrote a fuzzing harness for mGBA. This allows you to load various types of files and automatically spin a core for some number of frames to see if this input causes any crashes. I've put likely hundreds if not thousands of CPU-hours into fuzzing mGBA and a handful entry points: ROMs, save games, savestates…although cheat files remain un-fuzzed at the moment, this is next after I polish up fuzzing Game Boy savestates. I only started fuzzing the Game Boy core yesterday, and I already have a slew of important bug fixes. Following are all of the bugs I've found through fuzzing the GB core, with basic analyses of the bugs, and I'm sure there are more to come.

- [7b86d5c](https://github.com/mgba-emu/mgba/commit/7b86d5cec770712915edbcd7bf1b0ad96d1938dd): __GB Serialize: Prevent loading savestates that aren't about to load an instruction__
  - Description: mGBA uses a state machine for the CPU that keeps track of the next instruction to run. Savestates can't store this all of this information, so they need to be created when the state resets the machine.
  - Impact: Null pointer deference
  - Severity: Minor
- [740f7a0](https://github.com/mgba-emu/mgba/commit/740f7a0f66a1dc78af39db981dcc332bab8bdff6): __GB Serialize: Check for LY when loading state__
  - Description: LY is the current scanline. It could be placed out of bounds, causing the renderer to try to render a non-existent scanline
  - Impact: Out of bound memory write
  - Severity: Major
- [ff788a0](https://github.com/mgba-emu/mgba/commit/ff788a017c4daeab34156bc0ea30cbd71b6a74b1): __GB Serialize: Check DMA destination when loading state__
  - Description: The destination of a DMA to OAM could be placed out of bounds, causing and invalid write
  - Impact: Out of bound memory write
  - Severity: Major
- [c292f7e](https://github.com/mgba-emu/mgba/commit/c292f7ea93a0d653b56891bf1adb318ab9ac3063): __GB Serialize: Check for X when loading state__
  - Description: X is the index into current scanline. It could be placed out of bounds, causing the renderer to try to render a non-existent pixel
  - Impact: Out of bound memory write
  - Severity: Major
- [54cd85d](https://github.com/mgba-emu/mgba/commit/54cd85d236f149c66994dd1bc6334ac63effc058): __GB Video: Prevent dot clock from going negative__
  - Description: The dot clock keeps track of how many pixels to draw in a row. However, due to an integer overflow, it's possible to get this value out of bounds, causing the renderer to try to render a non-existent pixel
  - Impact: Out of bound memory write
  - Severity: Major
- [4c38f76](https://github.com/mgba-emu/mgba/commit/4c38f769565e8ddd7d3a8eef1a41975206c129a0): __GB Video: Prevent BCPS and OCPS from going out of bounds__
  - Description: BCPS and OCPS are the indices into the background and object palettes, respectively. They are used for read/write operations on the palettes and are stored due to the values being used in a state machine. However, the range of the indices in savestates exceeds the allowable range, leading to a limited out of bounds condition.
  - Impact: Out of bound memory read/write
  - Severity: Minor
  
All of these bugs were found within the past 24 hours via automated testing. Fuzzing is invaluable in its ability to find overlooked issues. None of these bugs affect the current stable version of mGBA (at time of writing, 0.4.1) having been introduced recently with Game Boy support, and they've all been fixed in master, so they won't affect the next release either. But notice that many of the bugs are unchecked memory accesses. These couldn't be hit during normal program flow due to the values being sanitized as they come from the emulated program, but savestates are a whole different beast, capturing the post-sanitization values. It is crucial to re-sanitize savestates as they are loaded!

I highly encourage all emulator developers to put the little bit of time needed into writing a fuzzing harness for their emulator and letting a fuzzer spin on it for weeks. You **will** find bugs, and you **will** be glad to have fixed them. And if you're writing software other than an emulator? Well, you should fuzz that too. Fuzz everything. Security is paramount.