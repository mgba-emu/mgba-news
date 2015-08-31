---
layout: post
title:  "mGBA Ports: Nintendo 3DS, Wii and PS Vita"
date:   2015-08-30
authors:
- endrift
tags:
- announcement
- development
- release
---
Two of the early goals of mGBA were to make it fast, and to make it portable. A common problem with older emulators is that they tend to be very entrenched in the architecture for which they were originally built. For example, ZSNES and the NO$ emulators have massive amount of x86 assembly directly in the core. Other emulators, like Project 64 and SSF only work on Windows due to their heavy reliance on the operating system's API. mGBA always sought to be free from any such restrictions by using easily-replaceable, lightweight abstraction layers for any platform-specific code.

With mGBA being portable, and as fast as it is, naturally a question arose of what platforms could it be ported to reasonably well? As it turns out, homebrew on game systems were an excellent target, as there is an already-existing userbase, and the platforms are different enough to stretch the possibilities of how well mGBA can be ported.

mGBA has now been ported to three homebrew platforms: the Nintendo 3DS, the Wii, and the PlayStation Vita. All of these ports are relatively young, but have been merged onto the master branch. As such, nightly builds have been set up, and new builds of mGBA for all three platforms will be available for download every day!<!--more-->

Alphas were already available for the Wii and the PlayStation Vita, but so much progress has been made in the meantime that it seems like a good time for a new alpha, as well as an alpha of the 3DS version.

- Nintendo 3DS: [alpha 1](https://s3.amazonaws.com/mgba/mGBA-3DS-alpha-01.7z)
- Wii: [alpha 3](https://s3.amazonaws.com/mgba/mGBA-Wii-alpha03.7z)
- PlayStation Vita: [alpha 4](https://s3.amazonaws.com/mgba/mGBA-Vita-alpha04.7z)
- Source: [ef9bb6a](https://github.com/mgba-emu/mgba/archive/ef9bb6ac5c0014b30f32a654e41955c1c74ea545.zip)

A [thread](https://forums.mgba.io/showthread.php?tid=25) on the forums contains more information and support for these ports.
