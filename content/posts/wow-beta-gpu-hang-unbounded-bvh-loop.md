+++
title = "Finding an infinite loop in a game shader without running the game"
date = 2026-09-18T20:00:00Z
description = "The World of Warcraft beta hung an NVIDIA GPU a few frames after every world load on Linux: Xid 109, device lost, client dead. Recording the Vulkan stream once and replaying it offline turned a hang that needed a logged-in game into one that could be bisected in seventeen-second runs, down to one compute shader with an unbounded tree traversal."
[taxonomies]
tags = ["nvidia", "vulkan", "proton", "linux", "debugging", "gaming"]
+++

Log in, pick a character, press Enter World. The loading bar finishes, the world
appears for an instant, and the picture freezes. Twenty seconds later the game
reports that a thread has become unresponsive and exits. Every time.

This is the story of how that turned out to be two loops in one compute shader,
and of the four wrong explanations that came first.

<!-- more -->

## Where this happened

- AMD Ryzen 7 7745HX laptop with an NVIDIA RTX 4060 Laptop GPU (Ada), 8 GB
- Arch Linux, kernel 7.2, nvidia-open 615.71.09, Hyprland 0.56.2 via Omarchy
- Battle.net under umu-launcher with GE-Proton11-7, which ships vkd3d-proton 3.1.0
- World of Warcraft beta client `WowB.exe` 1.60.1.69913, D3D12, 3840x2160,
  graphics preset at maximum

A finding on one machine may not carry over to another. What follows was
measured here; claims about other systems are marked as hearsay.

## The symptom

The game's own message is `ERROR #109: A thread has become unresponsive`, which
is its watchdog talking, not the cause. The kernel had the real event a few
seconds earlier:

```text
NVRM: Xid (PCI:0000:01:00): 109, name=WowB.exe, errorString CTX SWITCH TIMEOUT
```

Xid 109 means the driver asked the GPU to switch away from a context and the
GPU never did. vkd3d-proton then saw `VK_TIMEOUT` followed by
`VK_ERROR_DEVICE_LOST`, and the game's graphics log said
`Device Removed Reason: GPU Hung. Timeout when waiting for queue: Graphics`.
The frozen thread's top frame was in `dxgi.dll`, waiting for a GPU that was
not coming back.

## Four wrong turns

**Dirty driver state.** Earlier that day the same GPU had been through hours of
deliberate crash reproduction for an unrelated compositor bug, with a pile of
driver errors in the log. Blaming leftovers was reasonable and wrong: after a
clean reboot, with nothing else on the GPU, the hang came back two minutes into
the session.

**The graphics API.** Switching the game from D3D12 to D3D11 seemed to make it
worse: it now died at startup with `BC_ASSERT(dt >= bcDuration_Zero)`. That
assertion is a clock running backwards, and it led to a real but separate
discovery. The kernel had been saying this on every boot for weeks:

```text
Measured 9113637366 cycles TSC warp between CPUs, turning off TSC clock.
```

Measuring per core showed that CPU 0's timestamp counter is 2.5363 seconds
behind the other fifteen, which agree with each other exactly. Linux falls back
to HPET and carries on, but the CPU still advertises an invariant TSC, so
Windows code under Wine reads it directly and sees time jump whenever a thread
crosses CPU 0. Pinning the whole Wine session to CPUs 1-15 cured the assertion.
It did nothing for the hang. Once D3D11 could actually reach the world, it hung
exactly like D3D12.

**Optional Vulkan features.** vkd3d-proton turns on a lot of recent machinery
on this driver: descriptor buffers, descriptor heaps, mesh shaders,
variable-rate shading, low-latency mode, ray tracing. Disabling all of them at
once changed nothing.

**The driver's special buffer access path.** More on that below; it looked
very promising for about ten minutes.

A search turned up other people with the same signature on the same beta build,
across several NVIDIA generations and driver versions, on both APIs, with no
diagnosis. The one reported workaround was to lower the lighting and global
illumination settings. That is a clue, not a fix. This GPU has no business
hanging on a shader, at any preset.

## Making the bug portable

Every experiment so far had cost a full manual cycle: start the launcher, log
in, enter the world, wait for the freeze, kill everything. An online game is
also a poor thing to automate, so a person had to press the buttons. That does
not scale to a bisect.

The way out was to stop needing the game. GFXReconstruct is a Vulkan layer that
records every API call an application makes, with the data, into a file that
`gfxrecon-replay` can play back later. If the recording reproduces the hang,
the game, the account and the login are out of the loop.

Three things got in the way:

1. **The game exited silently with the layer loaded.** The Proton log showed a
   null call inside `vkGetPhysicalDeviceDescriptorSizeEXT`. That entry point
   belongs to `VK_EXT_descriptor_heap`, which the capture layer (1.0.5) does not
   know, so its dispatch table had a hole. The caller was DXVK, which the game
   also loads at startup, so both layers had to be told to stay away from it:
   `VKD3D_DISABLE_EXTENSIONS=VK_EXT_descriptor_heap` and
   `DXVK_CONFIG=dxvk.enableDescriptorHeap=False`.
2. **The first good recording would not replay.** It failed early with
   `vkCreateImageView returned VK_ERROR_INVALID_OPAQUE_CAPTURE_ADDRESS`.
   With `VK_EXT_descriptor_buffer`, image views carry driver-assigned opaque
   capture data, and the driver declined to hand the same addresses out again.
   Disabling that extension too makes vkd3d-proton fall back to ordinary
   descriptor sets, and those replay fine.
3. **Does it still hang in that configuration?** It had to, or the recording
   would be of a different program. It did.

With `GFXRECON_CAPTURE_FILE_FLUSH=true`, so the file is complete up to the
moment the GPU stops, one more manual world load produced a 475 MB capture of
1122 frames. Replaying it, with no game running:

```text
NVRM: Xid (PCI:0000:01:00): 109, name=gfxrecon-replay, errorString CTX SWITCH TIMEOUT
```

Same error, same info word, different process name. From here on, each
experiment was one command and about a minute.

## Bisecting a frame

Converting the capture to JSON lines and counting work per frame showed how
little of the world ever gets drawn. Frames up to 1119 are the loading screen,
19 draws each. Then:

| frame | draws | compute dispatches | queue submits |
|-------|-------|--------------------|---------------|
| 1120  | 1972  | 153                | 164           |
| 1121  | 1079  | 62                 | 21            |
| 1122  | 1087  | 60                 | 34            |

and nothing after that completes. `--quit-after-frame` narrowed it further:
stopping after the loading screen is clean, stopping after the *first* world
frame already hangs. With `--sync`, the device loss lands right after one
graphics-queue submit carrying 17 command buffers.

Draws or dispatches? `gfxrecon-replay --replace-shaders` swaps the SPIR-V of
chosen shader modules at replay time, so the 19 compute shaders used around
that submit were replaced with an empty `main`. Clean. Halving from there:

- a group of six, including the GPU culling shader behind the frame's indirect
  dispatches (my first suspect): still hangs
- the other thirteen: clean
- three sub-groups: only one matters
- five shaders, one at a time: exactly one

The culprit is a compute shader with a 4x4x4 local size, dispatched as 8x8x8,
so a 32³ volume, eight times in the frame. It samples a cube map, reads three
structured buffers and writes a storage image. That shape, and the known
workaround, both say global illumination probe update.

## The loops

Disassembled, the shader has two loops that walk a tree stored in a structured
buffer of 32-byte nodes, using a small explicit stack. Interior nodes push a
child and descend, leaves have the sign bit set in their first word and pop.
The continue condition of both loops is:

```text
%615 = OpINotEqual %bool %node %uint_536870911     ; node != 0x1FFFFFFF
%616 = OpIAdd %uint %sp %uint_4294967295           ; sp - 1
%617 = OpULessThan %bool %616 %uint_15
%619 = OpLogicalAnd %bool %617 %615
       OpBranchConditional %619 %loop_header %loop_exit
```

Keep going while the node is not the sentinel and the stack pointer is in
range. There is no iteration bound. A well-formed tree terminates. An all-zero
buffer terminates too, because node 0 keeps pushing itself until the stack
guard trips. A tree with a cycle in it never terminates, and a GPU has no way
to preempt a shader that will not end.

Here came the fourth wrong turn. Every buffer read in that shader goes through
`OpRawAccessChainNV`, from `SPV_NV_raw_access_chains`, an NVIDIA-only extension
that both vkd3d-proton and DXVK use when available. A driver bug there would
explain identical hangs on both APIs. So I rewrote the shader to use plain
`OpAccessChain` plus bitcasts, 38 accesses, validated it and replayed. It still
hung. Not the extension.

The decisive test was smaller. Keep the shader exactly as it is, add one
counter, and AND `counter < 4096` into those two branch conditions. With that:

- the first world frame replays clean
- the whole capture replays clean, in 13 seconds instead of never

Memory barriers are present between every dispatch in that command buffer, so
the ordering inside it is correct. The bad node data comes from somewhere
earlier. Why the tree contains a cycle on the first frame after a cold load is
the part I could not determine from outside the game.

## Fixing the live game

vkd3d-proton has a developer feature for this. With `VKD3D_SHADER_DUMP_PATH`
set it writes every shader as `<hash>.dxil` and `<hash>.spv`; with
`VKD3D_SHADER_OVERRIDE` it loads `<hash>.spv` from a directory in place of its
own translation. Nearly all of this game's compute pipelines are created at
startup, so a dump from the login screen was enough to find the shader by
comparing SPIR-V with the one from the capture. Its hash is
`2d79ed5755446f2f`.

One trap: the SPIR-V that vkd3d-proton generates depends on the descriptor
model in use, so the shader from the capture configuration is not the shader
the normal configuration runs. The override has to be made from a dump taken in
the configuration it will be used in. A forty-line script finds the two branch
conditions by shape rather than by id, adds the cap, and `spirv-as` reassembles
it.

Result: D3D12, every setting back at maximum, a cold world load, and the
character standing in a village with the GPU at full load and an empty kernel
log. The override is keyed to the shader's hash, so it will quietly stop
applying the day the client ships a rebuilt shader. If the loops are bounded by
then, good.

## What I take from it

- **Record once, replay forever.** The expensive part of this bug was reaching
  it. A capture moved it from "needs a person, a login and a minute of
  clicking" to a shell loop. Everything conclusive happened after that.
- **`--replace-shaders` is a bisect tool.** Swapping shaders for no-ops in
  halves found one module out of 2500 in a dozen replays.
- **An instrumented copy beats a theory.** The raw-access-chain idea was
  plausible, specific and wrong. The capped copy of the same shader could only
  come out one way if the loop was the problem.
- **Translation layers were not at fault.** Both of them passed the shader
  through faithfully. The report belongs with the people who wrote the shader.

And CPU 0 on this laptop is still two and a half seconds behind everyone else.
That one is for another day.
