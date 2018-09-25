---
layout: post
title:  "mGBA Ports: Nintendo Switch"
date: 2018-09-16
authors:
- endrift
tags:
- announcement
- development
- release
---
When homebrew hit the Wii U in 2015 I decided to not create an official port of mGBA. There were several reasons for this, but the primary reason was the fact that Nintendo was releasing GBA games in the Wii U VC. It already ran well in vWii mode, and the writing was already on the wall that the Wii U was on its way out. With the hype surrounding the next system, then only known as the NX, I decided to wait out the Wii U and see how the next hardware held up.

When the Switch eventually did launch, with little news on a Virtual Console, I decided that I would port mGBA to the Switch if Nintendo had in fact abandoned VC as the rumors claimed. As the fate of VC got more and more grim, I decided that I would release a Switch port as soon as hardware accelerated graphics arrived.

With the recent addition of 3D graphics support to libnx I decided to jump in. Porting to the Switch was relatively painless, and I'm glad to announce that early builds of mGBA for Switch are [now available on the downloads page]({{ site.baseurl }}/downloads.html#homebrew-2) and the upcoming mGBA 0.7.0 will include official Switch support.<!--more-->

The Switch port is still missing features, such as rumble and controller sensor support, which will arrive in upcoming builds. Compiling mGBA for Switch currently requires building [libnx](https://github.com/switchbrew/libnx) and [libdrm\_nouveau](https://github.com/devkitPro/libdrm_nouveau) from source, as it requires fixes in both libraries that have not yet made it into releases. Please be patient as the libraries and the port improves.
