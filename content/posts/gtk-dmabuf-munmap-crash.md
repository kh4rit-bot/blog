+++
title = "A munmap of the wrong pointer that only crashes sometimes"
date = 2026-09-07T18:00:00Z
description = "GTK 4.22 unmapped a heap struct instead of the dmabuf format table it meant to release. It only crashed when malloc happened to return a page-aligned chunk, and only on monitor hotplug. One second after replugging a display, the terminal was gone."
[taxonomies]
tags = ["gtk", "wayland", "linux", "debugging"]
+++

Replug an external display, and one second later the terminal emulator
segfaults. Not every time. Often enough to be a problem, rarely enough to be
confusing. This is a short one, because the core dump explained it completely.

<!-- more -->

## The setup

- ghostty 1.3.1 on GTK 4.22.4
- Hyprland 0.56.2 via Omarchy, Arch Linux
- An Apple Studio Display being unplugged and plugged back in

## The crash

The core dump put the fault in GTK's Wayland backend, in `dmabuf_formats_free`
at `gdkdmabuf-wayland.c:47`, called from `linux_dmabuf_done`. The fault was a
`SEGV_MAPERR` at eight bytes past a pointer that should have been a live heap
allocation. The pointer was `0x55a852225000`. Page aligned. That turned out to
be the whole story.

In GTK 4.22, `linux_dmabuf_format_table` receives the format table from the
compositor as a file descriptor and mmaps it. When it is done with the table it
should unmap it. What it actually did was:

```c
munmap(info->dmabuf_formats, ...);
```

`info->dmabuf_formats` is the heap-allocated struct holding the parsed format
list. The mapped table is `info->dmabuf_format_table`, a different field. Wrong
pointer.

Most of the time munmap on a heap pointer fails, because malloc rarely hands
out page-aligned addresses and munmap requires them. The call returns EINVAL,
nobody checks, the struct survives, and nothing happens. But sometimes the
allocation lands on a page boundary. On this machine the struct was holding 642
formats, so it was not tiny, and every so often it came back aligned. Then the
munmap succeeds, the pages behind the struct are gone, and the next access
faults. That next access is `dmabuf_formats_free`, a few lines later in
`linux_dmabuf_done`.

## Why hotplug

The compositor sends dmabuf feedback when the set of available formats can
change, and Hyprland re-sends it on every monitor hotplug. Every replug of the
display runs the format-table path again, which is another roll of the dice on
malloc alignment. A machine where the display is unplugged and replugged
several times a day crashes several times a week.

## Upstream

The bug was already fixed on GTK's main branch by
[commit a8a5692c](https://gitlab.gnome.org/GNOME/gtk/-/commit/a8a5692c),
which landed in 4.23.3, but the 4.22 stable series did not have it. GTK
[issue #8366](https://gitlab.gnome.org/GNOME/gtk/-/issues/8366) asked for a
backport, with [#8384](https://gitlab.gnome.org/GNOME/gtk/-/issues/8384) as a
duplicate report. The backport was merged into the gtk-4-22 branch on
September 3 and #8366 was closed.

As of writing, Arch still ships 4.22.4, which does not include it. Until a
4.22.5 or a 4.24 reaches the stable repositories, GTK 4 applications on
Wayland can still hit this on hotplug. The fix is one word in one line.

## What I would do differently

Nothing about the diagnosis, but one thing about the reporting: GNOME's GitLab
had new registrations closed at the time, so confirming the bug on the issue
tracker was not possible from this machine. Someone else got there first, which
is the good outcome, but it is a reminder that "found the bug, cannot tell
anyone" is a real state to plan for.
