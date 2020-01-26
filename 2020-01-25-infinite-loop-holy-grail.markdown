---
layout: post
title: "The Infinite Loop That Wasn't: A Holy Grail Bug Story"
date: 2020-01-25
authors:
- endrift
tags:
- emulation
- debugging
toc: true
---
Once upon a time there was a game for the GBA by the name of Hello Kitty Collection: Miracle Fashion Maker. It was a cutesy game based on the famous Sanrio Hello Kitty franchise, developed by Imagineer. But hidden beneath the surface of the seemingly innocuous title was an insidious problem. Somehow, this simple game just did not boot in any GBA emulator. This alone was enough to qualify it as a Holy Grail bug. As with all Holy Grail bugs the bug itself was absolutely baffling. Here the explanation was simple: sometime during the game's startup sequence, it enters a loop that can never exit due to expecting a specific value reading out of memory _that does not exist_. While many games have similar bugs, such as the intro sequence in the high-profile game _The Legend of Zelda: The Minish Cap_, those rely on a specific behavior from reading invalid memory addresses. But this loop seemed to defy that behavior. And yet, it works on actual hardware. Moreover, this same bug also occurs in when loading a save in Sonic Pinball Party after a cold boot. Could the expectation of these invalid memory accesses be wrong somehow? But how?

{: .hero }
![Hello Kitty Collection: Miracle Fashion Maker](/assets/hai-kitty.png)
<!--more-->

## But That's Illegal, Isn't It?

So wait, if you access invalid memory the game should just crash, right? An illegal operation has occurred or a [segfault](https://en.wikipedia.org/wiki/Segmentation_fault) or whatever you want to call it. Right?

Well. Sort of. Not quite. Not on the GBA at least.

On ARM, the CPU architecture that powers the GBA, this error condition is called a data abort and occurs when you try to access memory that the [memory management unit](https://en.wikipedia.org/wiki/Memory_management_unit) does not have mapped with read permissions[^pabt]. When a data abort occurs, the CPU stops what it was doing and jumps to the [exception vector](https://en.wikipedia.org/wiki/Interrupt_vector_table) associated with data abort exceptions. The operating system can then decide if it should kill the current process, map in memory for a [page fault](https://en.wikipedia.org/wiki/Page_fault), try and let the process handle it like some emulator JITs do with "fastmem", or various other things.

[^pabt]: In the case of trying to execute memory that's not mapped with execute permissions this is called a prefetch abort.

So how does the GBA handle a data abort? The exception vector entry for data aborts resides in the GBA boot ROM (or colloquially, "BIOS"). If a GBA gets a data abort it tries to jump to the DACS[^dacs] handler if it exists, otherwise it hard-locks. No commercial games have DACS handlers. So why doesn't it hard-lock? Simply, the GBA does not ever raise data aborts. It doesn't have an MMU (or even a memory protection unit like the DS has), so it just keeps going and reads invalid memory out.

[^dacs]: Short for Debugging and Communication System, DACS is part of the GBA's dev kit.

## Beep Beep! Here Comes the Memory Bus!

{: .article-aside }
![A bus](/assets/bus.png)
This isn't a _memory_ bus...

What the heck is invalid memory? What does it look like? That's the crux of the problem here. And it's a complex situation; what it reads out is highly dependent on what the CPU has been doing recently, or more accurately, what the _memory bus_ has been doing recently. The short explanation is that on an invalid memory access the CPU reads out whatever was last on the memory bus. To figure out what that would entail requires a bit of insight into what a memory bus is and how it works.

The memory bus is the bit of circuitry that connects the CPU to all of the different memory components on a platform. On the GBA there are several things attached to the memory bus, such as working RAM, video RAM, and the cartridge bus. Whenever the CPU tries to access memory it tells the memory bus which address it wants to access, which lights up the particular component that corresponds to that address. The component then puts the value at that address onto the bus, which may take a few cycles[^ws], and finally the CPU can read the value off of the bus. On the GBA, when no hardware is associated with that address, no value is put onto the bus, and the CPU reads out whatever the _last_ value put onto the bus was. This can be transformed in various ways, such as if it was a 16-bit read but the CPU is trying to do a 32-bit read, but it's always going to be the value on the bus.

[^ws]: These stall cycles while reading the bus are sometimes referred to as wait states.

## That Doesn't Sound So Bad...Right?

So you can just cache the last memory access, right? And then return that again? In general, that approach would work, but there are some complications. First, you need to make sure your memory accesses are all in the right order. This is harder than it sounds due to the fact that the CPU accesses memory every instruction to fetch the next instruction in the pipeline. And in fact, in general\* the memory lingering on the bus is the last instruction that's been fetched. This simplifies the process to just returning that last prefetched value. But since the last prefetched value depends solely on where you're currently executing from in memory, it should always be the same. Even if the address you're fetching changes, so long as it's invalid you'll always get the same memory.

Uh. Wait. But that loop exits, and it won't exit if that value is the prefetch value. So what's going on? If it's always fetching the next instruction what could happen in between? I tried running similar infinite loops on test ROMs to see if, for example, the value could deteriorate. This can definitely happen if the value isn't refreshed recently, but the value is refreshed every single instruction, so it doesn't have time to deteriorate. My tests never exited the loop. I was doing something differently than these games, even though I had reproduced the loop exactly the same. What was I doing wrong?

## Pokémon Emerald and the Hardware-Only ACE

Fast forward to this month, January 2020. The Sonic Pinball Party bug report is about three and a half years old at this point. It's been known in other emulators for years. I'm out of working theories. Late that month a user named **merrp** joins the mGBA Discord and mentions that there's a new arbitrary code execution glitch for Pokémon Emerald that only works on hardware. Moreover, the glitch is likely going to be used by speedrunners, who might want to practice on emulator. Obviously this is a tempting target for a bugfix, though I wish I'd known before releasing 0.8.0. I start taking a look at the glitch and confirm merrp's assertion that it only works on hardware. In every emulator I try it hangs on a black screen. But merrp tells me the thing it's hanging on reading from invalid memory in a loop and I realize I'm probably not going to be able to fix it anytime soon. It's **that** bug again.

Pinpointing the function that's looping gives me an edge this time. Thanks to the [pokeemerald](https://github.com/pret/pokeemerald) decompilation project that means I can easily make targeted changes to the function to try and figure out how it manages to break out of the loop. A simplified version of the loop in question looks somewhat like this:

```c
uint16_t type = /* ... */;
for (int32_t i = 0; table[type][i] != 0xFFFF; ++i) {
	uint16_t value = table[type][i] & 0xFE00;
	if (value > 0x7E00) {
		break;
	}
	/* ... */
}
```

What this is doing is pretty simple: there's a two-dimensional table of values. For each row in this table for the specific column `type`, it first tries to determine if it's a specific "sentinel" value. If it is, stop looping. Otherwise, mask off the value and see if it's larger than a check value. If it is, stop looping. Otherwise, do something else later in the loop. In the specific case of this glitch, the value of `type` is out of the bounds of the table, leading to an invalid pointer. This means that the trying to access the `i`th element of this non-column will always access invalid memory. Though the offset into the table increases each iteration of the loop it would have to iterate hundreds of millions of times before it loops back to valid memory. So it's clearly not doing that. So how does it break out of the loop?

To investigate this I modified the loop to see what would happen if it just broke out of the loop immediately. Simply enough, the ACE worked on both hardware and emulator at that point, and did not hang at all. So instead I tried to set the screen color to the value it reads when it exits the loop and then hard-lock so the color doesn't get changed. Recompiling it and running on a real GBA turns the screen a brilliant shade of blue after several seconds of hanging on a black screen.

<div style="color: #0084e7; background-color: #0084e7; width: 240px; height: 160px; margin: auto; text-align: center">SO BLUE</div>

On emulator it just hung on a black screen still. So what value would it read out if it was reading the prefetch value? Instead it turns a bit of a dull teal.

<div style="color: #008484; background-color: #008484; width: 240px; height: 160px; margin: auto; text-align: center">ew</div>

So it definitely was going through the loop at least once before it managed to break out. The amount of time it takes to break out of the loop on hardware seemed to be variable too. Anywhere from 2 to 30 seconds seemed pretty common. What was going on?

## A New Working Theory

Then I noticed a difference between my test ROM and Pokémon Emerald when they hang. There was music playing in Pokémon. There was also music playing in Sonic Pinball Party. There wasn't music playing in Hello Kitty, but this gave me an idea. What happens if an interrupt gets raised between prefetch and the data load? Will it start prefetching the interrupt vector before the invalid memory access? I quickly mocked this up in mGBA, turned on interrupts in the test ROM, and sure enough it broke out of the loop. So I tried the same test ROM on hardware and…it did not break out of the loop. So there goes that theory. Eventually I realized something. You saw that asterisk earlier I'm sure, so yes, there is one thing that can happen in between prefetch and the memory access, but only if the memory bus gets queried by something other than the CPU between the prefetch and invalid memory access.

I said that the memory bus was managed by the CPU. This is mostly accurate, but there's another piece of important hardware that can also access the memory bus, bypassing the CPU. This process is known as _direct memory access_. I've described DMA in [a previous article]({{ site.baseurl }}{% post_url 2018-03-09-holy-grail-bugs-revisited %}#the-great-gba-dma-disaster) so I won't go in much additional depth on the basics of DMA here. You might notice if you read the article again that it said the main CPU is paused while the DMA is running. So that means that while a DMA is running the bus value will now be the last memory access from the DMA. This is mostly relevant if a DMA runs off the end of valid memory into an invalid region; it will duplicate the last good value.

It's been known for a while that if you load invalid memory within a DMA that you get the last DMA value, but it's been a while since I implemented that in mGBA and I had forgotten about that. When I saw that in the invalid memory access code while contemplating the bug it suddenly clicked. What if the DMA value lingers on the bus for one instruction? If the first instruction after the DMA finishes loads invalid memory before it fetches the next value it should in theory load the DMA value again. Moreover, playing music on the GBA usually entails using DMAs to transfer the audio data to output. Though to do this properly would require a cycle accurate emulator that can block the CPU in the middle of instruction execution between the instruction start and the memory access, mGBA's GBA emulation is not cycle accurate. [Sounds familiar]({{ site.baseurl }}{% post_url 2017-07-31-holy-grail-bugs-2 %}#speedy-gonzales-los-gatos-bandidos). Fortunately, I was able to figure out a workaround. It's not perfect, but I can compare the expected CPU location for the instruction after the DMA with the current CPU location on an invalid load and use the DMA value instead of the prefetch value for that one address.

## The Long-Sought Solution

I turned on H-blank DMAs in the test ROM I made and synced to V-blank so timing would be consistent, put it on hardware and…this time it worked! The test ROM consistently exited the loop after a specific number of iterations with the DMA value read out instead. I was correct! Implementing this in mGBA took a few tries to get right but sure enough it exited from the loop with the same results as on hardware. Suddenly I got the blue shade in mGBA. Hello Kitty booted. Sonic Pinball Party saving worked.

I'd done it.

This might be the longest I've spent on a single bug. Across over three years I've poured more hours into debugging this than I care to count, and I'm sure other developers have had similar experiences with their emulators too. Without that flash of insight it might have taken another year, or more, but staring at that black screen with nothing happening beyond music playing was the domino that knocked over the entire problem.

Now that this solution has been found it can be implemented in other GBA emulators too, marking a definitive end to this bug. This bug will be fixed in mGBA 0.9.0 when that comes out hopefully later this year, and is already fixed in the development builds. You can finally play Hello Kitty Collection: Miracle Fashion Maker to your heart's content. If that's what you want. I won't judge.

{: .hero }
![The same bus](/assets/bus-out.png)
