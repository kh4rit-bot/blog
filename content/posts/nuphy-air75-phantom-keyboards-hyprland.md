+++
title = "The display that killed keyboards was a keyboard"
date = 2026-09-07
description = "For months, plugging an Apple Studio Display into a Hyprland laptop would eventually freeze every keyboard on the machine. The display was innocent. A NuPhy Air75 V3 hanging off its hub was exposing phantom keyboard interfaces that arrived with Caps Lock already latched."
[taxonomies]
tags = ["hyprland", "linux", "usb", "keyboards", "udev"]
+++

For months this laptop had a bug that everyone described the same way: plug in
the Apple Studio Display, and at some point afterwards all the keyboards die.
Not just the ones behind the display's Thunderbolt hub. The built-in keyboard
too. The mouse kept working, windows kept getting focus, the compositor was
fine. Keys simply stopped arriving. The only cure was to unplug the display.

The display was never the problem. This is the story of how it got exonerated
and what was actually going on.

<!-- more -->

## The setup

- Arch Linux, kernel 7.1.9-arch1-2
- Hyprland 0.56.2, with `kb_options = compose:caps,shift:both_capslock_cancel`
  and `numlock_by_default = true`
- fcitx5 as the input method
- An Apple Studio Display acting as a Thunderbolt hub, with a NuPhy Air75 V3
  ISO keyboard (USB `19f5:102a`) plugged into it

## Wrong suspects

The display was the obvious one. It shows up as a large pile of USB devices,
audio, camera, hub, and the freeze only ever happened while it was connected.
Every debugging attempt started with "what does the display do on hotplug".

The second suspect was fcitx5. Input method daemons sit right in the key path,
and restarting it occasionally seemed to help, which turned out to be a
coincidence. Killing it outright while frozen changed nothing, which settled
that.

The third was Hyprland's seat state itself, and that one was closer, but
"Hyprland has a bug" is not a diagnosis. Something had to be putting it into
the bad state.

## What the snapshots showed

The useful tool ended up being boring: a loop taking `hyprctl devices` snapshots
every three seconds around hotplug events, kept in a log so the sequence could
be read after the fact. About 110 snapshots across two full freezes.

Two things stood out.

First, the NuPhy did not enumerate as one keyboard. It enumerated as several.
The device exposes four USB interfaces: 00 is the boot keyboard, 02 is
mouse/consumer/system controls, and then 01 (the NKRO, or "N-key rollover",
second keyboard protocol) and 03 (a vendor interface) also present
keyboard-capable evdev nodes. Hyprland dutifully creates a keyboard object for
each one.

Second, and this was the moment the display stopped being a suspect: one of
those extra NuPhy keyboard objects showed `capsLock: yes` in the very first
snapshot after plugging in. Before any key had been pressed. Four plugs out of
four, both through the display's hub and later with the keyboard on a direct
USB cable. With `compose:caps` in the config, a real Caps Lock press is
impossible, so the latch could not have come from typing. It was born that way.

Every keyboard object on this machine is also born with `numLock: yes`, because
of `numlock_by_default = true`. So there is clearly a path that seeds lock
modifier state onto newly created keyboards, and the phantom Caps Lock appears
to be riding it.

## How the freeze unfolds

Once a latched phantom keyboard exists, the failure has two phases.

**Phase 1.** Plain typing works everywhere, but modifier binds stop matching.
Pressing Super+3 delivers a literal "3" to the focused window and no
`workspace>>` event fires. This phase is easy to miss, because most people do
not notice a workspace bind failing until they try it.

**Phase 2.** The first modifier chord typed, whether Super, Alt, or even
Shift+letter, kills key delivery for every keyboard on the seat. That includes
the built-in AT keyboard, whose own `capsLock` reads `no` and which never
re-enumerated. Pointer input, `hyprctl` IPC, focus changes and `windowtitle`
events all stay healthy throughout. From the outside it looks like the
compositor is fine and the keyboards are dead, which is exactly what it is.

In the snapshot timeline both freezes begin at the exact snapshot where the
main keyboard's `capsLock` desyncs from the latched physical object, and end at
the exact snapshot where the latched device object is destroyed. Which brings
us to the next point.

## Why nothing short of unplugging works

Every in-place reset was tried: `hyprctl switchxkblayout <main> next`,
`hyprctl reload`, restarting fcitx5. None of them fix it. They clear the main
virtual keyboard's copy of the latch, and for a moment things look fine, but
the physical device object underneath stays latched and the wedge re-arms on
the next chord.

Meanwhile a raw evdev capture on the phantom interfaces, taken while typing and
chording, showed zero events. The extra interfaces never send anything at
runtime. The damage is done entirely at enumeration time, by state seeding, and
the only thing that undoes it is destroying the device object. Hence unplug.

A side gotcha for anyone doing this kind of debugging: Hyprland renames live
device objects when sibling devices re-enumerate. Device names from
`hyprctl devices` cannot be trusted across hub events, so match on the
underlying evdev node, not the name.

## The fix

Hide the phantom interfaces from libinput, so Hyprland never creates keyboard
objects for them:

```
# /etc/udev/rules.d/99-nuphy-phantom.rules
SUBSYSTEM=="input", KERNEL=="event*", ENV{ID_VENDOR_ID}=="19f5", ENV{ID_MODEL_ID}=="102a", ENV{ID_USB_INTERFACE_NUM}=="01", ENV{LIBINPUT_IGNORE_DEVICE}="1"
SUBSYSTEM=="input", KERNEL=="event*", ENV{ID_VENDOR_ID}=="19f5", ENV{ID_MODEL_ID}=="102a", ENV{ID_USB_INTERFACE_NUM}=="03", ENV{LIBINPUT_IGNORE_DEVICE}="1"
```

Reload with `udevadm control --reload` and replug the keyboard. It now
enumerates as a single clean device, Shift, Super and Alt chords all work, and
the Studio Display can stay plugged in indefinitely. Verified end to end,
through the display hub, over a week of daily use.

The rule is deliberately narrow. Interface 01 is the NKRO protocol, so in
principle ignoring it could matter if the boot interface stopped reporting
under heavy rollover. In practice interface 00 on this keyboard handles
everything that has been thrown at it.

## Upstream

This was reported as a
[comment on Hyprland discussion #10360](https://github.com/hyprwm/Hyprland/discussions/10360#discussioncomment-18207320),
"Mod key stuck logically", where other people describe the same symptom class
with different triggers. The phantom interface case is probably one
deterministic path into a broader stale-lock-modifier problem rather than the
whole story, but it is a repro a developer can hold onto: a multi-interface
keyboard whose secondary interface latches a lock modifier at device-add.
Possibly related: hyprwm/Hyprland#4442, an NKRO keyboard toggling Caps Lock and
Num Lock, closed as stale.

NuPhy support was also sent the technical details, since a keyboard reporting
Caps Lock active on an interface that never sends events looks like a firmware
quirk. They forwarded it to their technical team. No firmware yet.

## What I would do differently

Trust the timeline, not the narrative. "It happens when the display is
plugged in" was true and completely misleading. The display was just the thing
the keyboard was plugged into. Plugging the keyboard straight into the laptop
reproduced the bug in one try, and that experiment could have been done on day
one.
