+++
title = "The laptop that suspended two thousand times a day"
date = 2026-09-07T20:00:00Z
description = "A Lenovo Legion Pro 5 never stayed asleep. The visible symptom was a systemd unit with 17,000 restarts, which was pure misdirection. The real culprit was a firmware-armed GPIO wake pin, found with kernel PM debug messages and silenced with one kernel parameter."
[taxonomies]
tags = ["linux", "kernel", "acpi", "suspend", "lenovo"]
+++

This laptop, a Lenovo Legion Pro 5 16ARX8, spends most of its life closed on a
desk with an external display. Left alone it is supposed to lock, blank the
display, and suspend. What it actually did, for days, was suspend and wake up
again roughly every 30 seconds, all night, thousands of times.

Nobody noticed, because from the outside a laptop that resumes in two seconds
looks exactly like a laptop that is asleep. What got noticed was something
else entirely.

<!-- more -->

## The setup

- Lenovo Legion Pro 5 16ARX8 (machine type 82WM), BIOS LPCN65WW from March 2026
- Arch Linux, kernel 7.1.9-arch1-2
- Hyprland via Omarchy, with the lid closed and the internal panel disabled
- Suspend to RAM (S3, "deep"), not s2idle

## The wrong symptom

The thing that got noticed was a systemd user unit, `omarchy-sleep-lock.service`,
whose job is to hold a sleep inhibitor and lock the screen before suspend. Its
restart counter read 17,445. Its log was a wall of "Failed to inhibit: already
running", and it has `Restart=always`, so it was failing, restarting two seconds
later, failing again.

The obvious conclusion is that the unit is broken. It is not. It fails about
four times per suspend cycle, because during the short window when logind's
PrepareForSleep signal is in flight the inhibitor is legitimately taken, and
the unit has no way to know that. Harmless on a machine that suspends twice a
day. On a machine that suspends 2,221 times a day, which is what the journal
showed for a single day, the counter looks like the disease. It was the fever.

## The actual loop

Reading logind's log with the timestamps lined up made the shape clear:

1. Lid closed, internal panel off, external display attached. The machine
   idles, locks, and five seconds after locking the lock screen blanks the
   external display.
2. When the external display blanks, it drops off DRM entirely. logind now
   sees a closed lid and no external display, so `Docked` flips to false and
   `HandleLidSwitch=suspend` fires.
3. The machine suspends. One to two seconds later it wakes up. The gap between
   logind's "Suspending" and "finished" lines is 10 to 17 seconds, and all of
   it is pre-sleep work. The time actually asleep rounds to nothing.
4. logind has a `HoldoffTimeoutSec` of 30 seconds after resume before it will
   act on the lid switch again. It expires. The lid is still closed. Go to 2.

That is about 109 cycles per hour. Across two days the journal showed 4,465 of
them.

## Finding the wake source

Every suspend was being woken immediately by something. The kernel will say
what, if asked:

```
echo 1 > /sys/power/pm_debug_messages
```

Then suspend once and read the kernel log:

```
PM: Triggering wakeup from IRQ 7
amd_gpio AMDI0030:00: GPIO 4 is active: 0x30057c00
```

IRQ 7 is the AMD GPIO controller's interrupt. GPIO 4 is one of its pins, and
the status register value has both the interrupt-status and wake-status bits
set. The pin is described in ACPI as an event pin on `AMDI0030:00`, edge
triggered and armed as an S3 wake source. Its interrupt count in
`/proc/interrupts` was 4,494 against 4,466 suspends. Same number, give or take
the handful of times it fired while awake.

Before getting there, two red herrings ate time. The AC adapter device and the
embedded controller both looked plausible, both were wrong. The lid switch
itself is handled by the embedded controller through ACPI query methods, not
by this GPIO, which mattered later: ignoring the GPIO would not break lid wake.

What GPIO 4 actually is, the firmware does not say in a way the kernel can
show. Its ACPI event handler lives in an SSDT rather than the DSDT. A
[thread on linux-gpio from June 2026](https://lore.kernel.org/linux-gpio/CAFdvZNt+mRTE9Q+utn=T8GzU7s_09ULhUtzTMMa9dgGCsE+r9A@mail.gmail.com/)
about this exact model identifies pin 4 as HDMI hotplug detect routed through
the discrete GPU, and pin 2 as USB-C DisplayPort hotplug. Which fits: the
external display blanking is the thing that triggers suspend, and the same
display's hotplug line is the thing that ends it.

## The fix

The kernel's GPIO ACPI layer has a quirk table with `ignore_wake` entries for a
handful of ASUS and Acer machines with the same controller. Nothing for this
Lenovo, as of 7.1 and linux-next. But the same mechanism is exposed as a module
parameter, so no patch is needed:

```
gpiolib_acpi.ignore_wake=AMDI0030:00@4
```

Added to the kernel command line and rebooted. The kernel confirms it at boot:

```
amd_gpio AMDI0030:00: Ignoring wakeup on pin 4
```

Since then pin 4 has fired zero times, and the machine has suspended ten times
in four days, each time because someone meant it to.

One detail from the linux-gpio thread is worth recording for anyone on this
model. The reporter there was on BIOS LPCN62WW, where the firmware did not
declare pins 2 and 4 as ACPI event pins at all, so `ignore_wake` had nothing to
match and they wrote an
[out-of-tree module](https://github.com/Lenart12/legion-nowake) instead. On
LPCN65WW the firmware does declare them, they show up as "ACPI:Event" IRQs 26
and 27, and the plain parameter works. So the fix depends on the BIOS version.
There is also a
[Fedora discussion](https://discussion.fedoraproject.org/t/179290) covering
the same pins.

## Collateral damage

Two thousand suspend cycles a day is not free. On one of those nights the
internal USB hub (a Genesys Logic 05e3:0610) failed to resume, the kernel gave
up on it with error -22 and disconnected it, and it took the built-in Realtek
Bluetooth radio down with it. No Bluetooth until a full power-off. The webcam,
on the same hub, was resetting on every single wake. If a machine that "sleeps
fine" has flaky USB peripherals in the morning, it is worth counting how many
times it actually slept.

A second, rarer wake source remains: the Logitech Bolt receiver supports USB
remote wake and woke one of the test suspends. It is much less frequent, does
not loop, and was left alone.

## What I would do differently

Check the suspend count before checking anything else. `journalctl -u
systemd-logind | grep -c Suspending` on a machine that has been closed
overnight should be a single-digit number. Everything downstream of that, the
unit restarts, the Bluetooth loss, the webcam resets, was explained by one
number that took two days to look at.
