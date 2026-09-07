+++
title = "Who keeps killing gnome-keyring?"
date = 2026-09-07T19:00:00Z
description = "gnome-keyring-daemon 50.0 started aborting every ten minutes, hours after a password manager was installed. The password manager was innocent. A status bar widget was spawning several copies of a mail CLI at once, and each one hit a client-registration race in the daemon."
[taxonomies]
tags = ["linux", "gnome-keyring", "dbus", "omarchy", "debugging"]
+++

At 01:45 one night gnome-keyring-daemon started dying. Not once: every ten to
thirty minutes, SIGABRT, a new core dump, a crash notification on the desktop,
and everything that had a keyring session open getting "not logged in" errors
until it retried. It kept going for the rest of the day, 43 crashes in total.

A password manager had been installed a few hours earlier. It integrates with
the system keyring. Case closed, surely.

<!-- more -->

## The setup

- Arch Linux, gnome-keyring 50.0
- Hyprland via Omarchy, with the Omarchy shell and its bar
- systemd-coredump catching the crashes, with debuginfod available for symbols

## Where it died

The core dumps were fully symbolized thanks to debuginfod, and every one of
them had the same stack. The daemon was inside
`service_method_open_session` in `daemon/dbus/gkd-secret-service.c`, finishing
a D-Bus `OpenSession` call. Session negotiation had failed, so it tried to
complete the call with a NULL output and a NULL result, and
`g_variant_new("(@vo)", NULL, ...)` is a fatal GLib error. Abort.

Why did negotiation fail? Two assertions earlier in the same crash, in
`gkd_secret_service_get_pkcs11_session` and
`gkd_secret_service_publish_dispatch`, said the daemon had no client record for
the D-Bus sender. Client records are created by `ensure_client_for_sender`,
which runs as an idle callback scheduled from the message filter at
`G_PRIORITY_HIGH`. Method dispatch can beat it. A client that connects, sends
`OpenSession` as its first message, and possibly disconnects quickly can have
its method handled before the daemon has admitted it exists.

This matched an existing Ubuntu report,
[bug 2162595](https://bugs.launchpad.net/ubuntu/+source/gnome-keyring/+bug/2162595),
"gnome-keyring 50.0 crashes during concurrent Secret Service access", where
the trigger was a Python package manager running parallel installs. Same
version, same abort. So the bug was upstream and known. The question was who
on this machine was hammering the Secret Service.

## Catching the caller

The password manager was the suspect, but the crashes had a rhythm that did not
fit it: 15:00:26, 15:10:26, on the ten-minute mark. That is a poll, not a user.

`dbus-monitor` on the session bus, filtered to the Secret Service, was running
when the 15:10:26 crash landed. The last calls before the abort came from a
burst of new connections, each of which opened a fresh bus connection and sent
`OpenSession("plain")` as its very first call. The sender names resolved to
processes running the `hey` command line client for the HEY mail service.

Nobody was running it by hand. The Omarchy bar had a widget installed that
morning, the
[official HEY plugin](https://github.com/basecamp/omarchy-hey-plugin), added
to the shell configuration at 01:36. Nine minutes before the first crash. Its
service component refreshes every ten minutes and, on each refresh, spawns
several `hey` processes at once for different queries. Each process probes the
keyring on startup. Several simultaneous first-call `OpenSession` requests on
brand new connections is precisely the race the daemon loses.

The password manager was installed at about the same time by coincidence. It
was never involved.

## Fixing it

Locally: remove the widget from the bar configuration. Crashes stopped at once.

Upstream, two things. The keyring race is a GNOME bug and was already tracked
in gnome-keyring issues
[#183](https://gitlab.gnome.org/GNOME/gnome-keyring/-/issues/183),
[#190](https://gitlab.gnome.org/GNOME/gnome-keyring/-/issues/190) and
[#194](https://gitlab.gnome.org/GNOME/gnome-keyring/-/issues/194), one of
which has a candidate patch. The trigger was worth reporting to the CLI's
authors, since a tool that talks to the keyring should not need to create a
session just to check that the keyring exists. That went to
[basecamp/hey-cli#352](https://github.com/basecamp/hey-cli/issues/352), an
existing issue about the same symptom.

The response there was fast. According to the maintainer, the next release
changed the startup probe from a create-and-delete test entry to a read-only
lookup, tested with two dozen concurrent invocations against gnome-keyring
50.0 without a single daemon restart. That shipped in hey 1.3.1 the following
day.

With the CLI updated to 1.4.0 the widget went back into the bar on September 3.
Four days later, zero crashes. The daemon is still the same vulnerable 50.0
build, and anything else that opens many concurrent Secret Service sessions
would still kill it, but nothing on this machine does anymore.

## What I would do differently

Look at the timestamps before looking at the suspects. "It started right after
X was installed" is a real signal, but so is "it happens at :00 and :10", and
the second one pointed at the right answer in one step. Both things were
installed that night. Only one of them had a ten-minute timer.
