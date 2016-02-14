---
layout: post
title:  "A Special Surprise: Game Boy Support"
date:   2016-02-13
authors:
- endrift
tags:
- announcement
- development
---
{% hero pkmn-red.png mGBA playing Pokémon Red %}

For the past month, I've been working on a special surprise to be released with mGBA 0.5: Game Boy support. Although secondary to Game Boy Advance support, mGBA aims to have extremely accurate Game Boy support, eventually cycle accurate. Although still early in development, it already passes a high percentage of many GB accuracy test suites. However, it's still quite rough around the edges and is not recommended for everyday usage: it's quite slow due to the aim of cycle-accuracy, and still very buggy. Support will be available in upcoming [nightlies]({ site.baseurl}}/downloads.html#development-downloads), starting tonight.

Please note that this involved a gigantic amount of refactoring, so if you find a bug or regression, please make sure to report it [on GitHub](https://github.com/mgba-emu/mgba/issues). Some features, such as rewinding, have been temporarily removed, but will return before 0.5.0 is released. Furthermore, many features are not yet supported for Game Boy, such as savestates.