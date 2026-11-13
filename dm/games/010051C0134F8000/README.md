# DRAGON BALL Z: KAKAROT + A NEW POWER AWAKENS SET (`010051C0134F8000`)

The base game 1.00 (v0), SDK 9.3.1 (its own `sdk` build). CyberConnect2, Unreal Engine 4.22 to 4.25 (version 8 paks, `pakchunk0-Switch.pak` 4.5 GB and `pakchunk1-Switch.pak` 1.9 GB), NVN, movies through the SDK's movie module (`subsdk0`), CRI audio (`.awb`). Modules `main`, `subsdk0`, `sdk`. The 17 add-on content packs of the New Power Awakens set are packed with it.

Status: on an iPhone 17 (iOS 27.2, 8 GB, in LiveContainer), from a Windows build: the boot, the logos and the opening movies, the title, a new game and the opening fight in play, at 60 fps (30 on the Switch). At the opening movie about 2.5 GB resident, 1.2 GB of it Metal allocations, and 5.8 GB left before the system's memory limit. Not run on the Mac.

## What it needs

| Setting | Where | Why |
|---|---|---|
| No `game_args` | the profile | with `-SwitchVirtualAddressSpaceSize=` this `nninitStartup` never reserves the engine region (below) |
| Engine region given what the alias region has room for: gate `OBS_UE_REGION` on the store of its size, `main+0x1aa9d40` | the rewriter's engine-wide rule (`dm/tools/ue_region.py`), no build ID; the host's room (`hle_svc.c`) | Unreal's `nninitStartup` reserves 62 GB by default |
| FMallocBinned3 constructor call cut, `main+0x1aaa368` `bl 0x1952980` → `nop`, with gate `OBS_UE_MALLOC` on it | the rewriter's engine-wide rule (`dm/tools/ue_malloc.py`), no build ID | its pools would take 37 GB of the engine region; the host runs Unreal's heap (`hle_uemalloc.c`, the layout `UE4_24`: the name getter at slot 17, no TryMalloc / TryRealloc) |
| Engage's 5 GB block (`mem_reserve 0x140000000`, `mem_alias 0x118000000`), `app_pool 0x80000000` | the profile | room for the engine region beside the SDK's node area; Unreal sizes its page cache from the 2 GB pool (`Total 2012610560 bytes available for the page cache`) |
| `fps_target 60` | the profile; `t.MaxFPS` and `rhi.SyncInterval` found by name (`dm/tools/ue_cvar.py`) | 60 fps instead of the Switch's 30 |
| `pad_layout full` | the profile | a 3D action game with a free camera on the right stick |

## Frame rate

- **Current:** 60 fps (30 on the Switch). The host writes `t.MaxFPS` 60 and `rhi.SyncInterval` 1 into the engine's console variables, found by name ([Unreal's frame limit and present interval, found by name](../../../docs/reference/frame-rate.md#unreals-frame-limit-and-present-interval-found-by-name)): `t.MaxFPS` at `main+0x88177d8` (registered at `main+0x348359c`, float, default 0), `rhi.SyncInterval` at `main+0x871f080` (`main+0x201d5e4`, int, default 1). Unreal steps the game by the time that really passed. The engine looks both up again at `main+0x1a7d6d4` / `+0x1a7d740` (`FindConsoleVariable`, cached at `main+0x7cbaec8` / `+0x7cbaed8`).
- **Checked:** on the iPhone 17 the user played the opening fight at 60 and found it right. During the opening movie (30 fps by the movie) the frame time was p50 33.45 ms, the CPU 69 % of one core, the GPU p50 2 ms.

## Memory

- **Resolved: `LowLevelFatalError [Line: 919] nn::os::AllocateMemoryPages failed with 0x7`, then a SIGSEGV at the allocator gate (`x19` = 0), with `-SwitchVirtualAddressSpaceSize=2048` given.** This `nninitStartup` (`main+0xc6d0` → `main+0x1aa9490`) reads the option into `main+0x7cbea00` and reserves the region only when the variable is 0: `ldr x8, [main+0x7cbea00]; cbz x8` to `mov x8, #0xf80000000` (62 GB), `str x8, [x9, #0xa00]`, `mov x1, #0xf80000000` and `nn::os::AllocateAddressRegion(main+0x7cbe9f8, 62 GB)` (`main+0x1aa9d2c` to `+0x1aa9d44`; a second path at `main+0x1aaa600` does the same). With the option the base at `main+0x7cbe9f8` stayed 0, and the first commit, `AllocateMemoryPages(0 + offset)` (`main+0x1aaa9e0`, from `main+0x1ab1378`), failed into Unreal's fatal error (`mov w1, #0x397`, line 919, at `main+0x1aaaab4`). The fatal error's own `malloc` then found no `GMalloc` (`main+0x195cad4`) and entered `GCreateMalloc` again, whose `malloc(0x1c38)` returned 0, so the allocator gate stored through a null object. The same trap as Pikmin 4's ([Unreal's engine address region and its options](../../../docs/reference/memory.md#unreals-engine-address-region-and-its-options)); without the option the default path runs and the rewriter's gate sizes it.
- **Current: the allocator is the host's** ([Unreal's FMallocBinned3 replaced by the host allocator](../../../docs/reference/memory.md#unreals-fmallocbinned3-replaced-by-the-host-allocator)). `GCreateMalloc` (`main+0x1aaa350`) mallocs 0x1c38 bytes into `x19` and calls the constructor (`main+0x1aaa368` → `main+0x1952980`, the one store of the vtable `main+0x760a010` outside its destructors). The vtable has 18 slots and an older FMalloc than UE 4.26's: 0-1 destructors, 2 Exec, 3 Malloc, 4 Realloc, 5 Free, 6 QuantizeSize, 7 GetAllocationSize, 8 Trim, 9 / 10 the TLS cache setup and teardown, 11 InitializeStatsMetadata, 12 UpdateStats, 13 GetAllocatorStats, 14 DumpAllocatorStats, 15 IsInternallyThreadSafe and 16 ValidateHeap (both return 1), 17 GetDescriptiveName ("Binned3"). On the iPhone the Unreal heap reached 1.01 GB by the title.

## Files and shaders

- **Current:** two version 8 paks with the shaders inline in the cooked packages ([Unreal Engine 4.22 to 4.25 paks (version 8)](../../../docs/reference/shaders.md#unreal-engine-422-to-425-paks-version-8)): 43,113 shaders from 3,776 files.
- **Current: movies into a BGRA8 texture.** The logos and the opening are 1920×1080 movies the game decodes into a client texture of format 0x9a (BGRA8); they play right with red and blue swapped into place ([A BGRA8 client texture gets its movie with red and blue swapped](../../../docs/reference/movies.md#a-bgra8-client-texture-gets-its-movie-with-red-and-blue-swapped)). The first movie: 702 packets, 702 frames, 4.22 ms per packet on the iPhone.

## Found on this game

- **The FMalloc layout of Unreal Engine 4.22 to 4.25** (the name getter at slot 17, no TryMalloc / TryRealloc): `ue_malloc.py` `LAYOUTS`, `hle_uemalloc.c` `UE4_24` ([Unreal's FMallocBinned3 replaced by the host allocator](../../../docs/reference/memory.md#unreals-fmallocbinned3-replaced-by-the-host-allocator)).
- **Version 8 paks** ([Unreal Engine 4.22 to 4.25 paks (version 8)](../../../docs/reference/shaders.md#unreal-engine-422-to-425-paks-version-8)).

## Open

- Not run on the Mac; no smoke route.
- Played on the iPhone through the opening fight only: later areas (the open world, the training grounds) and their memory are not checked.
- The run that played was a Windows build of an older tree, not of this profile on this tree: the same cuts as build-gated rows (the allocator call at `main+0x1aaa368` and its gate, the engine region's default cut to 2048 MB at `main+0x1aa9d2c` / `+0x1aa9d3c` / `+0x1aaa62c` / `+0x1aaa638`), the frame-rate variables at the addresses `ue_cvar.py` finds, a 5.75 GB alias region and a 2 GB pool (`DM_ALIAS_SIZE`, `DM_APP_POOL`). Here the engine-wide rules find the same allocator call, region store and variables in this `main` (`ue_malloc.py`, `ue_region.py`, `ue_cvar.py` run on it), and Engage's 5 GB block gives the engine region about 4.06 GB instead of 2 GB. A build of this tree is still to be run.
- Only the base game 1.00 is registered; the updates are packed from a derived profile.
