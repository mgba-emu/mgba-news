---
layout: patreon
title: "\"Holy Grail\" Bugs in Emulation, Part 2"
date: 2017-07-31
authors:
- endrift
tags:
- emulation
- debugging
- patreon
toc: true
---
It's been two months since first writing on Holy Grail bugs in emulation and while it did pique the interest of many emulator developers there has been little movement on solving the bugs. Lior did much more research on solving Pinball Fantasies and byuu dug up a forum post implying that there is in fact a three scanline delay when enabling background layers on the GBA. However, none of the issues have been conclusively resolved.

Of course, the previous article only covers two emulated platforms. There are far more than just two emulated systems out there and along with them a large share more incomprehensible issues.

<!--more-->

## Nintendo DS Bugs

With currently only two major contenders for mature DS emulators what counts as a Holy Grail bug in DS emulation is a bit more difficult to define. While development has slowed, [DeSmuME](http://desmume.org) is still the standout DS emulator for PCs. Seemingly contradictorily however, the oft-considered best DS emulator is [DraStic](http://www.drastic-ds.com): it's much faster, more accurate and less buggy, but has the limitations of being closed source and solely for Android[^1], making it hard to peer into its development process. DraStic's developer Exophase provided some insight into its development, citing several problematic, bizarre behaviors including games that relied on reading unmapped memory, Art Academy relying race conditions involving waiting for IRQs, or worse: expecting proper behavior for accessing. VRAM that has overlapping mappings. Yes, is a thing that the DS can do and is well-defined.

[^1]: Old versions of DraStic are available for the [Pandora](http://openpandora.org) open handheld, and there are rumors of other non-Android versions either upcoming or otherwise available to private parties. Further, there has been [recent discussion](http://www.drastic-ds.com/viewtopic.php?f=5&t=4828) of DraStic going open source in the not-so-longterm future.

There are several up-and-coming DS emulators however: [melonDS](http://melonds.kuribo64.net) from Arisotura, [dasShiny](https://github.com/Cydrak/dasShiny) from Cydrak, [GBE+](https://github.com/shonumi/gbe-plus) from Shonumi, [CorgiDS](https://corgids.wordpress.com) from PSISP, and [medusa]({{ site.baseurl }}{% post_url 2017-04-08-medusa %}) from myself. We've been collaborating some, but for the most part we've been doing independent research. As such, we've discovered many of the same bugs, which has led us to put our heads together to figure out some of the more insidious ones.

### Pokémon Black/White

Pokémon Black and White versions are infamous for being difficult to emulate, and have historically been one of the most buggy games in DeSmuME. Or at least the buggiest games that people wanted to play. While the DeSmuME devs have been unwilling to put effort towards directly fixing these games there have been slow improvements to the point where the game is fairly playable. Meanwhile, DraStic also generally runs them quite well, but the newer DS emulators still struggle.

{: .article-aside }
![Pokémon White failing to boot](/assets/bw-save.png)
Pokémon's own "Blue Screen of Death"

There are three big issues that earlier phase DS emulators face with Pokémon Black and White: getting the game to boot in the first place, overcoming the anti-piracy mechanism that prevents EXP gain, and preventing seemingly random crashes. Of course, without being able to boot the game the anti-piracy issues are meaningless, so getting it to boot is the first and foremost issue.

Pokémon Black and White, along with a small handful of other DS games, use special cartridges with an IR sensor onboard. When booting the game using the typical save type detections and support, which apply to 99% of DS games, players are greeted by a nasty blue screen telling them that the save data cannot be accessed. The error cannot be dismissed and the game cannot be played. As it turns out, the IR port is accessed by using a slightly modified version of the protocol that the other 99% of DS games use to load and store save data. If this variant is not properly detected and supported, the game will present this error instead. Fortunately, the variation of the save protocol used in these games is fairly easy to detect and easy to implement (save for the IR communications themselves), which in turn makes overcoming this first hurdle fairly easy. The variant merely adds a 0 byte at the beginning of sending a command for the save protocol, and other values for this first byte entail that it's an IR command instead.

![Pokémon Black anti-piracy triggering](/assets/bw-no-exp.gif){: .article-aside }

Although getting past the save error screen does resolve the biggest game breaking bug–not being able to play the game at all–it still does not resolve quite another gargantuan game breaking bug. When the game detects that it's not running an official cartridge on an actual system it triggers a particularly evil anti-piracy measure: your Pokémon do not gain experience. Pokémon at its core is an RPG and experience is crucial to progression. While in theory it's possible to power through the game without gaining experience, this is only feasible for the most ~~masochistic~~ hardcore of players; it prevents the game from being generally accessible to players of any skill level or penchant for self-flagellation.

While there are patches and Action Replay codes online to prevent the anti-piracy from failing there is no reason that the emulator should allow itself to be seen as a pirate platform. Unfortunately these unofficial game patches do not contain a description of what is being patched or how the patch works. Attempts to reverse engineer the patch have had middling results on my end. With enough research it seems that the issue actually pertains to the time it takes for the cart itself to respond with loaded data. This makes sense since the flash carts that this code is designed to protect against are designed to work with any and all cart types (in theory). As such they may not always present the same timing characteristics as other game carts. And while I've made strides in trying to correct this issue in medusa, it persisted.

Recently Arisotura uncovered another issue that would easily catch most flash carts. When starting a game it would poll the cartridge to see if the IR sensor was present. Since flash carts do not have IR sensors the reply it received was incorrect. Further, I had not implemented any semblance of IR support into medusa yet. Once I implemented a proper reply to the polling, despite the lack of any additional IR functionality, experience began working. The first two issues have now been flattened out in medusa, but only very recently.

The third issue is random game crashes. These are also theorized to stem from cart timing issues and indeed may be an extension of the anti-piracy measures. Although I have not encountered this issue myself, I have been told that it results in partially loaded maps and corrupted graphics before ultimately crashing. The crashes only occur after hours of gameplay, so saving often can help avoid lost time from crashes.

### Final Fantasy III

{: .article-aside }
![Final Fantasy III intro getting stuck](/assets/ff3-intro.png)
There's actually a whole intro after this frame

Final Fantasy III for the DS is noteworthy for being the first official release of Final Fantasy III outside of Japan. While Final Fantasy I, IV and VI were released as I, II and III internationally, and II and V eventually got released as parts of various compilations, III was isolated to a Japan-only release for sixteen years. In 2006, Square Enix released a remake for the DS that featured 3D graphics and a pre-rendered intro cutscene. At least in theory.

In practice, the title screen presents issues for emulators. In DeSmuME, it locks up on a black screen unless you mash the buttons before the intro even starts. On older versions of medusa it would lock up after a few seconds, looping the last audio sample. Adjusting memory timings directly affected how long it took for the intro to freeze: the slower memory timings were, the longer it would play for. However, adjusting the memory timings also caused slowdown and other issues in other games. While the timings in medusa are currently a stand-in and very inaccurate, it was curious how adjusting the memory timings seemed to have the opposite effect in Final Fantasy III than in other games. Making things faster generally caused other games to run better due to not missing frame targets, but the intro movie would lock up sooner and sooner the faster I made memory timings. Meanwhile, if the memory timings were significantly slower the intro would work far better.

I talked with Cydrak who provided some insight into the issue. Sie had previously mentioned a game which require extremely tight synchronization between the two CPUs of the DS. Tighter synchronization between the CPUs leads to far better accuracy, but much, much slower emulation. Fearing that I'd need to tighten up the synchronization, I started to take a deeper look.

Medusa's synchronization timings between the two CPUs are fixed at compile time. This means that while it can be adjusted, it cannot be adjusted without recompiling the project. The standard value is roughly 2000 cycles at 66 MHz, which amounts to about 30µs of emulated CPU time. Making the synchronization tighter, I found a lower bound at around 1500 cycles before games would refuse to run, and an upper bound of around 3000 cycles. However, adjusting the synchronization timing *had no noticeable effect* for me. As such, I left it alone and moved onto other issues.

While I did eventually find the proper fix to allow the entire intro to play flawlessly, it wasn't until I had looked into fixing another game that I figured out what the issue was…

### Tongari Boushi to Oshare na Mahou Tsukai

{: .article-aside }
![Tongari Boushi loading forever](/assets/tongari-loading.gif)
Even worse than the macOS beachball

It's well known that DeSmuME is…a bit buggy. Most games work quite well with only smaller bugs, but there are a handful of games that just do not boot or get in-game. One such game is the obscure Japan-only title Tongari Boushi to Oshare na Mahou Tsukai (とんがりボウシとおしゃれな<ruby>魔法使<rp>(</rp><rt>まほうつか</rt><rp>)</rp></ruby>い). When loaded in DeSmuME and development versions of other DS emulators it would get to a loading screen and…spin. Forever. Given that it's an obscure Japan-only title there had probably not been much investigation into why it was broken, either.

Fast-forward to Arisotura doing research for melonDS. While looking into issues relating to cartridge timings she realized that the GBATEK documentation on cartridge timings omitted some important information. GBATEK is *extremely* good documentation for GBA (unsurprisingly) but does contain a few errata. Meanwhile, GBATEK's DS documentation is far more lackluster: while it covers most topics relating to DS hardware, many of the sections contain underwhelming or flat-out wrong information. One such section was the DS cartridge section. It does contain mostly factual information and certainly enough information to cover more than just the basics. It mentions basic information about the cartridge timings as well, but it has one glaring omission.

The DS cartridge protocol supports a handful of modes, mostly corresponding to cartridge encryption. In addition to raw access mode, there are also KEY1 and KEY2, which are different encryption modes for ROM startup (as boot code can be put into a region known as the "secure area") and general ROM data access. For the most part games only use KEY2 encryption after the initial boot process. Since the initial boot process is handled by the firmware before control is passed off to the ROM it is not critical to emulate KEY1 mode if firmware booting is not supported. As such emulator developers tend to ignore the KEY1-specific information in GBATEK, at least until far later in development. As it turns out, that's where the glaring omission is. There is timing information pertaining to both KEY1 and KEY2 modes that is only marked as KEY1.

DS cartridge transfers take place in three parts. First, the game sends a command to the cartridge itself, telling it what address to read from, in which mode to operate, and various other parameters. Second, data is transferred from the cartridge back to the DS, visible from the software side as four bytes at a time. However, some processing is done on the cartridge side to prepare the data for transfer, which necessitating operations to be done in blocks. Thus, the third part is the delay taken for the processing in between blocks. There is also a delay taken from sending the initial command to the cartridge and data being available to be read from the cartridge. These timings are referred to by GBATEK as "gaps". Notably, though, the gaps are not handled automatically by the hardware: these timings are not universal between all DS cartridges and as such games will set the timing parameters in software for the gaps before performing transfers.

GBATEK does document these settings but says they only apply to KEY1. There is no mention whatsoever of how gaps work for KEY2, or even a mention of their existence. But this doesn't make sense: the data still needs to be buffered or otherwise processed before it can be transferred. Since cartridges have [random access](https://en.wikipedia.org/wiki/Random_access) buffering must be able to be done at transfer start time. This in turn means that a startup gap must exist. Indeed, when using the parameters for KEY1 gaps for KEY2 accesses the infinite loading time of Tongari Boushi becomes finite, Final Fantasy III's intro movie functions properly, and various other glitches just disappear. And again, it is important to remember that documentation, especially reverse-engineered documentation, is not infallible and must be subjected to scrutiny when seeming contradictions are present.

## SNES Bugs

I personally have only worked on emulating three different platforms. Many other platforms are known for some of their emulation difficulties, so I've sought out seasoned accuracy-focused developers of other platforms, starting with the obvious choice: with his hyperfocus on accuracy and the SNES byuu had some stories to tell.

### Speedy Gonzales: Los Gatos Bandidos

{: .article-aside }
![The infamous button](/assets/speedy-button.png)
Game over I guess?

Speed Gonzales for the SNES has a notorious bug that historically affected all emulators. In level 6-1, part one of the level Galactical Galaxies, there is a button to be pressed for the game to progress. A casual player would run up to the button, press it, and…the game would lock up. The music continues but the player can no longer do anything. The game has no save system, so that means quite a lot of play time down the drain. And until 2010, this affected every single SNES emulator. Fifteen years after the game came out. So what causes such an issue? In the interest of perfect accuracy, byuu dug into the details.

The basic premise of the bug is that the game hits a tight loop where it is perpetually reading a value from unmapped memory at `$225C`. The loop will only exit if this points to a memory address that contains a value meeting specific criteria. As discussed in my [previous article on Holy Grail bugs]({{ site.baseurl }}{% post_url 2017-05-29-holy-grail-bugs %}#mega-man-battle-network-4) some systems have open bus behavior, which means that reading from an unmapped address leads to reading back the last value that was passed on the bus. In this case, the value is the last byte of the load instruction, `$18`, which is read back twice from the bus—once for the low byte, once for the high—yielding the address `$1818`. The value at this memory address would never fulfill the exit condition so the loop would never exit. And in most emulators, the bus reads could never change, always yielding the same address. Another previous article, this time on the differences between [cycle accuracy and other forms of accuracy]({{ site.baseurl }}{% post_url 2017-04-30-emulation-accuracy %}), all but a cycle-accurate emulator would make it impossible for another value to be read. After all, if the instruction is uninterruptable no value could be loaded onto the bus between the last byte of the instruction and the load performed by the instruction.

Enter cycle accuracy. When you drop the assumption that instructions cannot be interrupted and allow other events to occur in the middle of an instruction a lot of things change. In this case, something interesting happens: suddenly the value on the bus can, in rare circumstances, change. After some false starts byuu realized that an HDMA—a specialized kind of hardware load that is performed in between scanlines—could trigger between the load of the instruction itself and the load performed by the execution. Thus, in the case where the HDMA fired at *just* the right time, the value read off the bus would *not* be `$18`. As it turns out if it read back a 0 the new address would contain a value that *did* let the loop exit. Indeed, this was the solution. After hundreds of person-hours of investigation, that was one Holy Grail bug down.

### Magical Drop

{: .article-aside }
![Magical Drop "game over"](/assets/magical-drop.png)
You're stuck here forever

So what happens when you have a bug in an emulator that…sometimes reproduces on hardware, but not always? Even worse, the probability of it occurring on hardware varies from unit to unit? It's a difficult situation and it's one that plagues the Japan-only game Magical Drop.

The Super Nintendo era had a wave of block-matching puzzle games like Panel de Pon (Tetris Attack in the US), Puyo Puyo, or Columns. In this same vein, Magical Drop features a slight variation on the match-3 formula. The game has several modes, including a versus mode and an endless mode. The game is also notable for having voice clips that play in some emulators but not others. Also in some emulators, endless mode really is endless.

Usually endless or marathon modes function by giving you a variant of the gameplay that does not have a set win condition. Rather, you keep playing until you max out the score counter (and you can keep playing after that too) or you hit a failure condition and get a game over. Like many match-3 games, the failure condition is when you can no longer fit blocks on your screen. The player character then exclaims in disappointment and you can enter your initials for the high score screen.

But in higan, and *sometimes* on consoles too, the game over screen never appears. The player character keeps hanging their head in shame, and the game softlocks. You can't enter your initials. The words "Game Over" never appear. Buttons do nothing. So what triggers this softlock? Well, that remains partially unsolved, but investigation into the issue opened a whole can of worms.

Much like the issues with voice clips never playing the issue appears to be related to the SNES's sound chip. The S-DSP handles processing audio samples into the final output audio. It allows for making various adjustments to the audio in the process, including pitch, echo, [ADSR](https://en.wikipedia.org/wiki/Synthesizer#ADSR_envelope), and more. The game adjusts the state of S-DSP to tune how things are to be output.

So…what happens if a game fails to initialize the S-DSP before it uses it? Well, the current hypothesis is that…we don't know. Or at least we don't definitively know. A common failure case for older game consoles is that when hardware isn't properly initialized its state is actually somewhat *random*. The exact details of this randomization vary from device to device, but this can and has led to games whose behavior will change dramatically based on some random state. Magical Drop happens to be one of those games. Though it's not entirely clear which part of the S-DSP is causing the hang. A hypothesis is the state of the pitch adjustment, but this has yet to be properly confirmed.

One of the issues with fixing the S-DSP randomness is that much of the internal state is opaque beyond the S-DSP itself, making it difficult to tell what should be random and what shouldn't be. Another issue is due to a lack of proper test cases to ensure nothing breaks. In fact, byuu attempted to randomize the entire S-DSP state, and sure enough, it broke things that previously worked. For example, King of Dragons would randomly drop sound effects and channels. Further, which ones were missing varied every boot. So clearly some things weren't random…but some things are. The issue has not yet been fully investigated on hardware, but research is ongoing and will hopefully yield a further understanding as it continues. Until then, well, you're resigned to using less accurate emulators for this specific game.

## That's four consoles down

So what's next? Well I've only addressed Holy Grail bugs in four consoles so far. There are many consoles out there more complex and less well understood than the SNES or the GB, and with them their own sets of issues. We've barely even scratched the surface of older consoles, and newer ones not at all.

As you get newer they get far more difficult to understand. They have more moving parts and more complex rendering pipelines. Understanding such large systems as a whole is difficult enough, but understanding ways they can break is far worse. In part 3, I plan to interview emulator developers for newer systems their takes on the difficulties in emulation. I don't know about you but I can't wait to find out what horrors lie buried in the Sega Saturn.
