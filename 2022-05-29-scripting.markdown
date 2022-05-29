---
layout: post
title:  "Scripting support now available in dev builds"
date:   2022-05-29
authors:
- endrift
tags:
- announcement
---
The highly-anticipated scripting feature, which has been in development for the past several months, has now been merged and is available in development builds.
With this merged, users can now write and run scripts in Lua, as is possible in some other emulators.
Currently, there is only preliminary support and many features are not yet exposed.
These builds include an example script that shows how to interact with the emulator, and can pull information about the party from the US releases of the first three Pokémon generations.
There is also documentation on the current API available [on its own page]({{ site.baseurl }}/docs/scripting.html).

As this feature is still nascent, we're looking for feedback on how to improve it and what should be added in the future.
For this, there is now a #scripting channel in the Discord server (linked on the sidebar) to discuss writing scripts or requesting support or new features.
If you've wanted to write scripts for mGBA, now's the time to start looking into it!

The current features are currently implemented:

- Read/write access to emulator memory (via the full address space, or via memory domains) and registers
- Save state saving and loading
- Getting and updating the currently pressed buttons
- Getting various informational state about the emulated game
- Taking a screenshot to file
- Various callbacks, such as per-frame, when the core is reset, right before keys are read, and more
- Instruction advance, frame advance, and resetting the emulation state
- A console for logging, and buffers for displaying textual information to the user
<!--more-->

There are several features that are currently notably absent, however:

- Overlays or similar
- Debugger integration
- Interaction with state that is managed by the front-end, such as fast-forwarding or pausing
- Support on the homebrew ports
- "Headless" mode, for running scripts in the background without the emulator being visible to the user
- Support for languages other than Lua, such as Python

While these features will be added in the future, these may not make it into the first release with scripting.

As per usual, development builds can be downloaded at the [bottom of Downloads page]({{ site.baseurl }}/downloads.html#development-downloads)
