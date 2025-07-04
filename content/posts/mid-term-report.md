+++
title = "Mid-Term GSoC Report"
date = "2025-07-04T10:38:39+02:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = ""
authorTwitter = "" #do not include @
cover = ""
keywords = ["", ""]
description = "The mid-term report for my Google Summer of Code @ Waycrate"
showFullContent = false
readingTime = false
hideComments = false
+++

I began contributing to Waycrate at the beginning of June, and since next week will mark halfway of the program, I am writing this brief report about my activities in the last few weeks. 

The first week was spent on understanding the codebase, especially the areas that were relevant for the issues I was about to work on, as well as useful tools used for debugging, such as [`d-feet`](https://wiki.gnome.org/Apps(2f)DFeet.html)/[`d-spy`](https://apps.gnome.org/Dspy/), utilities that allows to simulate single D-Bus calls. They are very useful to test changes on single methods, although they are not always the best way to test changes, since some of the issues I fixed appeared on sequences of function calls.

The goal of the project is to fix several issues about the current implementation of `xdg-desktop-portal`:
- Key combinations including modifier keys such as `Shift` or `Alt` are not working
- Wrong cursor position: the simulated pointer movements are performed in a different position from the one passed to [`org.freedesktop.portal.RemoteDesktop.NotifyPointerMotionAbsolute`](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop#org-freedesktop-portal-remotedesktop-notifypointermotionabsolute) (See [Issue #82](https://github.com/waycrate/xdg-desktop-portal-luminous/issues/82) for a video demonstration)
- Remote desktop control is always on even if a remote connection with RustDesk is closed

I firstly tested and reproduced all the issues, and the latter appeared to be a problem in the implementation of RustDesk. To understand this, I tested the issue on a Fedora 42 virtual machine running `xdg-desktop-gnome`, and I was able to reproduce the issue.

The first issue I studied is the one about the keyboard simulation. It turns out that RustDesk uses [`org.freedesktop.portal.RemoteDesktop.NotifyKeyboardKeycode`](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop#org-freedesktop-portal-remotedesktop-notifykeyboardkeycode) to notify keys, and the internal implementation just forwarded these calls to a [`ZwpVirtualKeyboardV1`](https://smithay.github.io/wayland-rs/wayland_protocols_misc/zwp_virtual_keyboard_v1/server/zwp_virtual_keyboard_v1/struct.ZwpVirtualKeyboardV1.html) object. However, key combinations are not obtained through setting the state of two keys to "pressed", but rather to the pressure of a key with a modifier: for example, the `A` character is not produced by simulating the pressure of `Shift` followed by the pressure of `a`, but rather the pressure of `a` with the `Shift` modifier. I added a `Modifiers` enum to map each modifier key to its bitflag, and I obtained the keycodes representing the modifier keys from the header file `/usr/include/linux/input-event-codes.h`. Pressing a modifier key now sets its bit in the modifiers bitflag to 1, while releasing it sets the bit back to 0. The CapsLock modifier is handled separately since it is the only modifier key that remains active after being released: each time it is pressed, its bit in the modifiers bitflag is switched.

To understand these concepts I built and studied a project called [`wvkbd`](https://github.com/jjsullivan5196/wvkbd), a virtual keyboard for `wlroots` compositors implemented in C. 

[`org.freedesktop.portal.RemoteDesktop.NotifyKeyboardKeysym`](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop#org-freedesktop-portal-remotedesktop-notifykeyboardkeysym) still needs to be fixed: keysyms, which are related to X11 key management, have a different semantic meaning compared to Keycodes, since a keysym can represent a keycode with a modifier (i.e. there is a keysym that represents `a` and another that represents `a`+`AltGr` or `a`+`Shift`). This means a keycode is mapped to different keysysms depending on the modifers active, and this mapping can be seen running `xmodmap -pke`.

I then fixed the cursor issue: the error is related to the backend implementation of [`org.freedesktop.portal.RemoteDesktop.NotifyPointerMotionAbsolute`](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.RemoteDesktop#org-freedesktop-portal-remotedesktop-notifypointermotionabsolute), which is called by RustDesk to update the cursor position during remote connections. To simulate the cursor movements, the [`motion_absolute`](https://traffloat.github.io/api/master/wayland_protocols/wlr/unstable/virtual_pointer/v1/client/zwlr_virtual_pointer_v1/struct.ZwlrVirtualPointerV1.html#method.motion_absolute) method of [`ZwlrVirtualPointerV1`](https://traffloat.github.io/api/master/wayland_protocols/wlr/unstable/virtual_pointer/v1/client/zwlr_virtual_pointer_v1/struct.ZwlrVirtualPointerV1.html) is used, which accepts two unsigned integers as parameters, `x_extent` and `y_extent`, that represent the dimension of the `WlOutput`. These parameters were hardcoded, and this produced an error because they are internally used for [scaling purposes](https://github.com/swaywm/wlroots/blob/51f8c22f4d72f9e18657baccfff3db544c1c0660/types/wlr_virtual_pointer_v1.c#L64-L69). I refactored the code in order to pass correct values to the function, that were obtained from [`libwaysip`](https://github.com/waycrate/waysip/tree/master).