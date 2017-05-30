---
layout: patreon
title: "\"Holy Grail\" Bugs in Emulation, Part 1"
date: 2017-05-29
authors:
- endrift
tags:
- emulation
- debugging
- patreon
toc: true
---
When developing a video game emulator the goal is to run software for that system on different hardware than the original. Further, the goal of an *accurate* video game emulator is to run *every* piece software for that system perfectly—or at least as close an approximation as possible. A decent emulator can be developed working from documentation of how a system behaves when running software. For the Game Boy Advance and Nintendo DS, such documentation is [GBATEK](http://problemkaputt.de/gbatek.htm); for the Game Boy, the [GBDev wiki](http://gbdev.gg8.se/wiki/articles/Main_Page) is invaluable.

Many of these document only say what will happen when the software is well-behaved. Software and hardware generally have a defined set of behaviors which, when given valid inputs, will (barring bugs) have well-defined outputs. For invalid inputs, documents often either say not to do that, or they don't even mention them. But inevitably some software will do things that documents won't talk about.

There is the concept of [Garbage In, Garbage Out](https://en.wikipedia.org/wiki/Garbage_in,_garbage_out) where if the input is not valid, the output is not well defined. In such cases software may become buggy and act unpredictably; however, for perfect compatibility, emulators must maintain *[bug-for-bug compatibility](https://en.wikipedia.org/wiki/Bug_compatibility)* with the original system. While some sources document the "garbage-out" from invalid inputs, this information is often sparse and can be full of errata or missing edge cases.

The actual debugging to discover these edge cases can be grueling. Yet even after days or weeks of research into just a single bug at a time, there are many bugs that are still incomprehensible and unsolved to the most seasoned of developers. Due to the extreme difficulty of tracking down and solving these issues I like to refer to them as *Holy Grail* bugs.
<!--more-->

## Game Boy Bugs

One thing about older game systems is that the hardware is often less refined and more full of edge cases and glitches. The Game Boy is a paragon example. Creating a basic Game Boy emulator is a relatively easy task and is often recommended to beginner emulator developers who want a simple target for one of their first projects. However, being full of edge cases and hardware bugs actually makes Game Boy an extremely difficult platform to emulate faithfully.

Without a doubt bringing up a basic Game Boy emulator is far easier than bringing up a basic Game Boy Advance emulator. Counterintuitively however, I've found that debugging Game Boy emulation bugs can be far harder than debugging Game Boy Advance emulation bugs.

### Pinball Fantasies

{: .note }
This section was partially written by Lior Halphon, who helped contribute much of the research involved.

{: .gallery }
![Pinball Fantasies title screen](/assets/pinball-fantasies-title.png) ![Pinball Fantasies crash](/assets/pinball-fantasies-crash.png)

Amongst Game Boy emulator developers there is a notorious game that works seemingly properly in inaccurate emulators but invariably crashes after seconds of gameplay in accuracy-focused emulators: Pinball Fantasies (as well as the nearly-identical title Pinball Deluxe[^pinball-titles]). Even [gambatte](https://github.com/sinamas/gambatte) and [BGB](http://bgb.bircd.org), by far the two most accurate Game Boy emulators, fail spectacularly in this game.

So why does the game break so utterly the more accurate the emulator gets? No one quite knows. The issue appears to stem from a slight timing issue, although where remains unknown. While a slight timing issue of this sort generally can cause frames to be duplicated erroneously or other small (but noticeable) issues, Pinball Fantasies appears to have been written in such a way that the issue actually cascades and causes the whole game to crash. But then why does this game work well in inaccurate emulators? That's also a bit of a mystery, but it's possibly because their timings are just wrong enough that things do not end up cascading the same way.

[^pinball-titles]: Beyond the name and logo, I'm not entirely sure what the difference between these two titles is. Furthermore, there is also the related title Pinball Dreams, but I've not investigated if this game has similar issues.

Typical speculation involves screen timing issues. The Game Boy's pixel processing unit (PPU) operates in a series of "modes" that tell the software what the PPU is currently doing. Mode 1 means that there are no scanlines currently drawing (either because it's in-between frames, or the screen itself is off), and the other modes tell the game what part of a scanline is being processed: sprites, the actual pixel data, or if it's in-between scanlines. However, various configurations of screen settings and sprites cause minute differences in the timings of these modes. The Game Boy can draw up to 10 sprites (also known as objects or OBJs) per scanline but the exact configuration of these sprites very slightly affects the timing of the PPU itself. Furthermore screen panning and "window" features also affect the timing. A scanline that has no sprites, no panning and no windows on it takes a well defined, well understood number of cycles for each of the modes. While how panning and windows affect timing is understood, once sprites are added  the timings become harder to decipher. Gekkio, author of the accuracy-focused emulator [Mooneye GB](https://github.com/Gekkio/mooneye-gb), created a test suite for many of the PPU timings, including the effects of sprites, and neither BGB nor gambatte pass this test.

However, [SameBoy](https://github.com/LIJI32/SameBoy) developer Lior Halphon did some investigation. By disabling certain features that affect scanline timings and comparing behavior in SameBoy with behavior on a Game Boy Color he determined that scanline timings were likely not the culprit. In fact, while experimenting with timing of other portions of the system Lior and I temporarily introduced many incorrect behaviors, mostly involving interrupts, each one seemed to fix the game. But all of these behaviors was incorrect and would undoubtedly break other games.

Another possibility beyond timing effects of the PPU is just one of several of the other incorrectly emulated timing quirks of the Game Boy. While it is possible that each one of these quirks are emulated correctly on at least one emulator, for the game to actually work an emulator needs to either emulate all quirks perfectly or emulate specific subsets of these quirks that by chance happen to work. This also both explains why some inaccurate emulators manage to run this game, and why *introducing* timing bugs to accurate emulators sometimes fix this game.

### STAT IRQ glitches

{: .gallery }
![Altered Space without STAT glitches emulated](/assets/altered-space-bugged.png) ![How Altered Space is supposed to run](/assets/altered-space.png)

One of the less-well explained, less-well known issues with the Game Boy is a series of strange behaviors regarding the STAT register and its related [IRQs](https://en.wikipedia.org/wiki/Interrupt_request_(PC_architecture)). The register itself contains information about the current mode of the PPU, as well as flags that allow it to trigger an interrupt upon entering different modes. It also has a flag to enable LYC, a way to trigger an interrupt on a specific scanline.

However, there are two very noteworthy bugs relating to this register that are fairly poorly documented. The first glitch actually involves how IRQs work at a hardware level. The Game Boy CPU has several possible IRQs, but all of the separate conditions that the STAT register controls are mapped to the same IRQ, collectively known as the STAT IRQ.

The PPU triggers the STAT IRQ on the CPU itself by adjusting the level of the STAT IRQ line — changing one signal from a 0 to a 1[^line-level] to tell the CPU that a STAT IRQ condition has been fulfilled and an interrupt should occur. STAT IRQs are [edge-triggered](https://en.wikipedia.org/wiki/Signal_edge), meaning that the interrupt occurs when the signal changes from a 0 to a 1, and not whenever a signal is 1. This prevents an interrupt from occurring a second time while the signal is 1 and the interrupt has already been acknowledged by the system.

[^line-level]: Both the CPU and the PPU are contained within the same chip (the LR35902), so it's hard to say if it's a rising edge or falling edge without die shots. However, for simplicity I'm describing it as rising edge.

Edge triggers are very common in electronics engineering. Level-triggered interrupts, as mentioned, can cause interrupts to occur multiple times even if the state hasn't changed[^ds-gxfifo-irq], making edge triggers far more common for this. However, in this case by trying to prevent the IRQ from occurring more than once it introduced a bug where IRQs can get missed.

There is very little documentation that talks about this bug, which is sometimes referred to as *STAT IRQ blocking*[^subjectivity]. It has long since been (at least partially) implemented in BGB and gambatte, but many other Game Boy emulators lack emulation of this bug due to the fact that it is unintuitive and very little documentation mentions it.

Gekkio provided insight into the matter explaining that when the PPU transitions between different modes that are both configured to trigger IRQs the IRQ line does not lower back to 0 before raising back to 1. The signal stays at 1 the whole time. Since this eliminates signal edges, the edge-triggered IRQ does not occur for that mode at all. This is where the name STAT IRQ blocking comes from: the STAT IRQ for the next mode is effectively blocked from occurring at all. This makes sense from a hardware perspective but to someone unfamiliar with signal triggering it might seem completely unintuitive. 

[^ds-gxfifo-irq]: One example of a level-triggered IRQ is the GXFIFO IRQ on the DS, which occurs when the graphics FIFO gets below a low-water mark. Until the FIFO fullness exceeds the low-water mark, an IRQ will always occur, even if the state of the FIFO does not change, after acknowledging the IRQ.

[^subjectivity]: I dislike the term STAT IRQ blocking because it implies an intent behind the bug. Perhaps a better term for this may be STAT IRQ dropping.

I independently discovered this issue while [debugging the game Altered Space](https://board.byuu.org/viewtopic.php?p=25527#p25527). Without emulation of STAT IRQ blocking the graphics get corrupted and the game is otherwise generally broken. Altered Space is notorious for being buggy in many emulators, but once STAT IRQ blocking is implemented most of the bugs disappear entirely.

The second glitch only occurs on the Game Boy, not on the Game Boy Color. When writing to the STAT register there are actually circumstances in which an interrupt will spuriously trigger *without any of the STAT IRQ conditions being fulfilled*. As with seemingly all obscure hardware bugs there are games that rely on this bug as well. A commonly cited game is Legend of Zerd (ザードの伝説, also known as Legend of Xerd or Zerd no Densetsu), which immediately crashes if the bug is not emulated. Likewise, the game will not run on a Game Boy Color since the bug was fixed.

## Game Boy Advance Bugs

{: .gallery }
![Madden 06 glitch](/assets/madden-06.png)

For the Game Boy Advance there are three games that stand out as having Holy Grail bugs: Lady Sia has a few lines in the middle of the screen that are glitched in every emulator, Madden NFL 06 and 07 have an issue where the screen is stretched to the point where it's impossible to make anything out, and Mega Man Battle Network 4 will crash in a certain area of the game. While it's relatively easy to work around these issues, they still indicate insufficiencies in the accuracy of emulation.

### Mega Man Battle Network 4

The most interesting game from this list is Mega Man Battle Network 4, specifically the Blue Moon version. Until very recently, the only option for emulating this game and avoiding a crash on the WoodMan stage was to patch the game. But more notably the game is also broken *on hardware* on the original model Nintendo DS…but *not* on the DS Lite! This is even noted on [Nintendo's website](http://www.nintendo.com/consumer/systems/gameboy/trouble_specificgame.jsp#megaman) alongside mentions of the venerable MissingNo glitch and the "internal battery has run dry" message from third generation Pokémon games.

{: .article-aside }
![The relevant portion of Mega Man Battle Network 4: Blue Moon](/assets/mmbn-woodman.png)
The infamous crashing level

The root cause of this issue involves the behavior of the Game Boy Advance when reading from invalid memory. Several things can occur if the CPU tries to read from a region of memory does not exist. On modern computers this is handled by what's known as the [memory management unit](https://en.wikipedia.org/wiki/Memory_management_unit) and gives a well defined result. On systems lacking an MMU the behavior can vary widely between approaches such as getting a [bus error](https://en.wikipedia.org/wiki/Bus_error) or reading a fixed value such as 0.

On the GBA what happens when reading from invalid memory locations is something emulator developers often refer to as "open bus". While specific hardware has different root causes for what triggers open bus behavior, the general idea behind open bus behavior is that instead of triggering an exception or reading 0, invalid reads get back the last value given to the CPU from that bus.

Due to how the [instruction prefetch]({{ site.baseurl }}{% post_url 2014-12-28-classic-nes %}#trick-5-prefetch-abuse) works on the Game Boy Advance the data for an upcoming instruction is fetched at the beginning every instruction. This is a few bytes ahead of where the current instruction is located. That value is therefore returned by the open bus.

Of course, invalid reads should never happen in well programmed games. On the GBA there are quite a few games that are not well programmed. Even high profile games such as Minish Cap can break if open bus is not implemented. What makes the Mega Man Battle Network 4 bug unique involves an even more specific edge case relating to how the memory is read. This case is so specific and so obscure that even Nintendo didn't realize it existed while developing the DS, leading to a hardware erratum.

Nintendo is no stranger to hardware errata. I mentioned some of the Game Boy hardware bugs previously, but there are far more issues than just the handful covered. By the time the Game Boy Advance rolled around all models of the same device generally had the same errata. Thus, if one model of the Game Boy Advance had a bug (and indeed, the Game Boy Advance does have some strange hardware bugs), all further models would contain the same bugs.

Yet there was one exception. Sure enough it was an edge case of an edge case of undefined behavior that would only occur in exceptionally specialized circumstances within buggy code. That buggy code happened to be called Mega Man Battle Network 4: Blue Moon.

In April of 2015 Martin Korth, author of the historic NO$GBA emulator and other ["nocash" emulators](http://problemkaputt.de), posted [a thread on the NGEmu forums](http://ngemu.com/threads/gba-open-bus.170809/) that described a newly discovered behavior that he had encountered while reading invalid memory locations. The Game Boy Advance runs an old ARM CPU, and ARM CPUs of that vintage can run in two modes: ARM mode and Thumb mode. While the previous description of invalid memory reads was correct for ARM mode, the description for Thumb mode turned out to be incorrect…but only sometimes.

As it turns out, depending on where the currently executing code was located, prefetch loads had slightly different effects on the memory bus. When running code directly off of the cartridge, from RAM, or from a handful of other places, the old description of Thumb behavior was correct. But there was one crucial case (and two minor cases) wherein this was incorrect.

The Game Boy Advance has a second, smaller RAM bank that is located on the CPU itself. While the larger RAM bank is much larger, it is a discrete chip, making it slower to access than on-CPU memory. Thus this "internal working RAM", as it is often called, is used for storing code that is timing-crucial. One such example is code for doing "mode 7" effects that must be executed once per scanline.

The prefetch operations have slightly different effects on the open bus behavior that wasn't discovered until much later. Mega Man Battle Network 4 would crash when finishing a battle in a certain section of WoodMan's stage if this behavior wasn't properly implemented. Nintendo noticed this before releasing the DS Lite, but it took emulator developers far longer.

NO$GBA and mGBA became the first two emulators to properly implement this behavior, thus solving the first GBA Holy Grail Bug.

### Lady Sia

![Lady Sia glitch](/assets/lady-sia.png){: .article-aside }

If there is an emulation bug that will haunt me to my grave it's this one. I've dissected this bug down to the exact instruction where everything goes awry and I'm still not sure what is being emulated wrong to cause this issue. The visual effect of the glitch is simple: there's a bar in the middle of the screen that shouldn't be there. It flickers sometimes. It's there in every emulator, but not on hardware. Tweaking the actual game itself to try and figure out if any perturbations cause the bug to appear on hardware didn't reveal anything. It's the most bizarre issue I've seen while working on mGBA.

These sorts of graphical bugs are usually relatively simple to track down: figure out what is actually appearing when that bar shows up, figure out what's causing it to appear there, and then figure out why the code is telling it to do that. Quite often, expectations about what the PPU should be doing in an odd case doesn't reflect what a game's expectations are, and the emulator developer needs to reconcile these to figure out what the actual behavior of hardware is.

In this instance the bar is actually the bottom few pixels of the repeating pattern that makes up the ocean spray graphic. What's causing it to appear there is that the game turns off that graphic layer in between frames and then says to turn it on about three scanlines higher than the actual repeat point of the texture. As for why the game is doing this? *I still don't know*. Even more interestingly, the exact timing of the toggle happens with some variability: due to audio timing the toggle can happen a few scanlines late, which is what leads to the signature flickering. However, this doesn't happen at all on hardware.

My best guess as to what's happening is that background layer enabling and disabling is latched in such a way that enabling a layer won't take effect until sometime after the layer itself is enabled by software. However, beyond this one bug I've seen no evidence to support this. While I have yet to test it on hardware I've seen enough games that I would expect to have run into another game that depends on this behavior.

Changing rendering parameters in between scanlines is extremely common and the changes tend to take effect on the very next scanline. The most extreme example I've seen of this is Rayman 3 which actually changes the background mode in between scanlines. This fundamentally alters how VRAM is parsed by the PPU and what various graphics settings mean. I'd assumed up until I saw this that changing the mode while a frame is rendering would be impossible; I was proven wrong. Even this appears to effect the next scanline.

If the background enable bits are indeed latched beyond the current scanline it would require thorough testing to figure out what the exact behavior is and extensive testing to make sure that changing this behavior to fix one game does not regress behavior in dozens of others. This bug defies everything I know about how background parameters work on the GBA and I have very few other ideas about what could be happening.

### Madden NFL 06 and 07

{: .article-aside }
![Madden 07 glitch](/assets/madden-07.png)
There are supposed to be graphics here…

The last of the Holy Grail bugs for the GBA is a bit more mundane than the others. While it doesn't crash the game like Mega Man does it provides a bit more than a minor flickery nuisance that is in Lady Sia. When a game begins in Madden the player is presented with a coin-flip to determine who first has possession of the ball. Or something like that; I'm not familiar with the rules of the game. Regardless, due to this bug the entire screen is smeared and only sprites are visible. The entire background layer is just a mess of vertical stripes.

As with the Lady Sia bug this should be relatively easy to track down. What's going on is that the layer is no longer being scaled 1:1; instead it's being stretched to being infinitely tall. This happens because the game overwrites the scaling parameters right before the frame is drawn. But what's weird is that I have no idea why the game is overwriting the parameter in the first place. Before being overwritten the parameter is valid. If that write is blocked the game works fine. But for some reason the parameter gets overwritten and causes the jumble that is the game screen.

While the exact solution to this issue might be as mundane as a simple timing issue with one of the memory subsystems this bug appears to plague every single GBA emulator. I'm really not quite sure why, and it leads me to believe there may be a deeper issue. The GBA may have another undocumented hardware bug. The write may be being blocked by hardware for an unknown reason. Again the solution is just unknown.

### D.K.: King of Swing

An example of a bug that affected every single GBA emulator I tried it in (although it apparently worked in ancient VBA) but was *not* in fact a Holy Grail bug was a glitch that would cause the game D.K.: King of Swing to crash upon entry to the Treacherous Twister level. What prevents this from qualifying is that the fix for the bug was actually to just follow the available documentation properly in one case. Even when working from correct documentation it is possible to misread or misremember a piece of the documentation and introduce bugs into the system. The root cause of this was such a case, and it just happened that all major emulators had overlooked this one small bit of information.

The version of GBATEK I was working off of stated that one specific bit on a specific setting register[^specific-bit] was read-only. This bit, bit 15 of the WAITCNT register (address 0x4000204), specifies if the GBA is running in AGB or CGB mode. The game would try to set this bit to imply the hardware was in CGB mode, and fail if bit did get set. It's unclear if this was intentional but it would crash upon entry to this level if the bit had the incorrect value. Despite this particular case being well-documented, every single emulator — including NO$GBA, which is by the author of GBATEK — had this bug. Given the wide presence of this bug and the simplicity of the fix I quickly reached out to developers of other GBA emulators. The bug has been fixed in all of them since.

## Bugs, Bugs, Bugs…

As emulator development becomes increasingly in-depth focused on accuracy more and more obscure, poorly written games uncover more and more bizarre bugs. Undocumented behavior and bugs in the hardware itself are the bane of all emulator developers and there are certainly issues that have not yet been uncovered in virtually all consoles to date.

There are many bugs and many stories that come from the pursuit of accurate emulation and this article only scratches the surface. In future articles I will be reaching out to other emulator developers for their Holy Grail bug stories.
