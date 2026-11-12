# Reskinning WindowsiOS as Windows 3.1 through Windows 10

A planning document. No code has been changed.

## 1. The premise needs one correction

This repo is not a Windows 11 clone. It is a virtual machine that boots the real, retail
Windows 11 24H2 ARM64 from Microsoft's own installer image. Nothing in the repo draws a
taskbar, a Start menu or a window frame. Windows draws all of it.

That changes what "reskin" means. There is no UI code here to restyle. The work splits into
three separate jobs:

| Layer | What it is | Where it lives | Share of the work |
| :--- | :--- | :--- | :--- |
| **A. Guest skin** | Make the real Windows 11 inside the VM look like an older Windows | The disk image, built by `scripts/` | About 75% |
| **B. Host chrome** | The iOS app's fake "Surface Laptop" bezel, keyboard deck, boot overlay | `ios/Sources/*.swift` | About 15% |
| **C. Plumbing** | Choosing a skin, building it, switching it, testing it | `scripts/`, `host/share.c`, `Makefile` | About 10% |

Your goal of "Windows 11 functionality, skinned to look like 98 or XP" matches Layer A
exactly: it is what the Windows theming community already does to real Windows 11.

## 2. What the repo gives you to work with

Read from the source, not assumed:

- **An offline image-customisation pipeline already exists.** `scripts/optimize_windows_lite.py`
  boots a throwaway WinPE disk next to the Windows disk, loads the `SYSTEM`, `SOFTWARE` and
  user registry hives, imports `.reg` files, renames system files, and shuts down. A skin
  installer is the same shape of job.
- **A first-logon hook exists.** `scripts/provision_windows_arm64.py` writes an `unattend.xml`
  with `FirstLogonCommands`. Tools that must be installed in a live session can run from there.
- **Headless boot with screenshots exists.** `build/winvm --headless --cmd-file … --screenshot …`
  is how both scripts watch progress. That is a ready-made way to check each skin.
- **A Mac runner exists.** `host/main_mac.m` and `build/winvm` run the same VM on an Apple
  Silicon Mac. All of Layer A can be built and checked there with no iPhone.
- **A guest-to-host message channel exists.** `host/share.c` serves `http://10.0.2.4/.henge/…`
  endpoints (`open-url`, `expose-port`, `oauth-done`). The guest can use the same channel to
  tell the iOS app which skin is active.

And the constraints the repo imposes:

| Constraint | Evidence | Why it matters for skins |
| :--- | :--- | :--- |
| No GPU. The display is a plain framebuffer (`ramfb`) and the desktop compositor renders on the CPU | `devices/fb_display.c`; the optimiser's own comment "CPU-rendered `dwm.exe` Mica/Acrylic" | Blur is expensive. The optimiser turns transparency off for this reason. Vista and 7 "glass" needs blur |
| The guest is ARM64 | README, provisioning script | Any tool that injects into `explorer.exe` needs a native ARM64 build. Stand-alone x86/x64 programs run under Windows' built-in emulation |
| The image is pinned to build 26100 and Windows Update is disabled | `OPT_SYS_REG` in the optimiser | Good: theming patches break on updates, and this image never updates |
| Defender is disabled | Same | Good: shell patchers are often flagged |
| The optimiser sets "best performance" visuals for every user | `make_user_reg()`: `VisualFXSetting=3`, `EnableTransparency=0`, `UserPreferencesMask`, `MinAnimate=0` | Skins must be applied after these, and some need to override them |
| Provisioning runs with `--no-net` | Both scripts | Every skin file must be downloaded beforehand into `work/` and injected offline |
| No binaries live in git | `.gitignore`, README note | Skin assets need a fetch list with checksums, not committed files |
| The user name `Ammaar` and machine name `IPHONE-DUO` are hardcoded | `unattend.xml`, `startnet.cmd` (`Users\Ammaar\NTUSER.DAT`) | Should become parameters before more scripts copy them |
| Host colours are hardcoded | Roughly 70 inline `Color(red:…)` literals across `SessionView.swift` and `LaptopDeckView.swift`; no theme object | Layer B needs a refactor before it can have more than one look |
| The 1-tap launchers type into the Run dialog | `VMViewModel.swift`: `launchGuestEdgeBrowser`, `launchGuestAntigravity`, `launchGuestPaint` send Win+R then keystrokes | Win+R is provided by `explorer.exe`. Any skin that replaces the shell breaks these buttons and the "Start" pill |

## 3. What a "skin" is made of

Each version needs the same list of parts. The left column is the checklist for every skin.

| Part | How it is changed on Windows 11 | Difficulty |
| :--- | :--- | :--- |
| Wallpaper, colours, cursors, sounds | Registry values and files. Fully supported by Windows | Easy |
| Icons (My Computer, folders, Recycle Bin) | Registry icon overrides, or patched icon libraries | Easy to medium |
| System font (MS Sans Serif, Tahoma, Segoe UI) | Registry font substitution | Medium. Many modern apps ignore it |
| Window frames and buttons | A custom visual style (`.msstyles`). Windows refuses unsigned ones, so a signature patch is required | Medium to hard |
| "Classic" grey 3D look (3.1, 95, 98, 2000) | Windows 8 removed the option but the drawing code is still there. A small tool switches it back on | Hard. Many side effects on Windows 11 |
| Taskbar, Start menu, tray | A replacement or a patch to `explorer.exe` | Medium to hard |
| File Explorer layout | Shell-patching tools | Medium |
| Logon screen, boot screen | Mostly out of reach. See section 7 | Hard or skip |
| The iOS frame around the screen | Layer B | Medium |

**A fidelity ceiling applies to every skin.** Theming reaches classic desktop programs and the
shell. It does not reach Settings, Edge, the new Task Manager, the new Notepad and Paint, or
any Store app. Those keep the Windows 11 look inside whatever window frame the skin provides.
Options are to accept it, or to swap a few of them for older-style equivalents (the optimiser
already removes many Store apps).

## 4. The versions, grouped into six families

Ten-plus versions collapse into six pieces of engineering. Within a family, versions differ
only by assets (wallpaper, colours, logo, sounds, Start-menu banner).

| Family | Versions | Window look | Shell | Relative effort | Main risk |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1. Windows 10** | 10 | Stock Windows 11 frames with square corners | Windows 10 taskbar and Start via a shell patcher | Lowest | Windows 24H2 removed the old taskbar code, so the patcher must supply its own |
| **2. Metro** | 8, 8.1 | Flat single-colour frames (visual style) | Taskbar without Start button (8) or with one (8.1); full-screen Start screen replacement | Medium | A convincing full-screen Start screen is the hard part |
| **3. Aero** | Vista, 7 | Glass frames (visual style plus a blur tool) | Windows 7 taskbar and Start menu replacement | Medium to high | Blur cost on a CPU-rendered desktop. May need a "glass without blur" variant |
| **4. Luna** | XP (blue, olive, silver; optionally Media Center "Royale") | XP visual style | XP taskbar and two-column Start menu | Medium | Finding a visual style that works on build 26100 |
| **5. Classic** | 95, NT 4, 98, Me, 2000 | Classic grey 3D theme | Classic taskbar and single-column Start menu | High | Classic theme breaks parts of Windows 11 (see below) |
| **6. Program Manager** | 3.1, NT 3.51 | Classic theme with 3.1 colours and 3.1-style title-bar buttons | No taskbar. Program Manager and File Manager replace Explorer as the shell | Highest | Shell replacement breaks the repo's launchers and removes the tray |

Notes on the two hard families:

- **Classic (95 to 2000).** Turning the classic theme back on is a known trick, but on
  Windows 11 it produces glitches: black or transparent title bars in some apps, a broken
  modern Task Manager, missing borders, odd behaviour in anything built on the newer UI
  toolkits. A pile of small follow-up fixes is expected. An alternative is a visual style that
  imitates the classic look, which is more stable but less exact. Decide by trying both.
- **Program Manager (3.1).** Windows 3.1 has no taskbar, no Start menu, no desktop icons and
  no right-click menus, so "Windows 11 functionality" has to give somewhere. Minimised windows
  need to become icons at the bottom of the screen, and something has to stand in for the
  tray (network, volume, clock). Microsoft has published the original File Manager source
  under an open licence, which helps. Program Manager needs a third-party recreation.
  3.1 also puts the title-bar buttons in different places, which needs its own visual style
  or acceptance of 95-style buttons.

## 5. Guest-side tooling (not in the repo; must be verified)

The repo contains none of this. The table is from general knowledge of the Windows theming
community as of my training data. **Every row needs checking against Windows 11 24H2
build 26100 on ARM64 before the plan depends on it.** This is the largest unknown in the
project.

| Need | Candidate tools | What to verify |
| :--- | :--- | :--- |
| Allow unsigned visual styles | SecureUxTheme, UltraUXThemePatcher | ARM64 build; works on 26100 |
| Re-enable classic theme | Windhawk "classic theme" mods, ClassicThemeTray | ARM64; severity of 24H2 glitches |
| General Explorer patching | Windhawk (mod platform), ExplorerPatcher | ARM64 builds exist for both, I believe; confirm versions |
| Classic and XP taskbar | RetroBar (stand-alone, themes for 95 through XP and Vista) | ARM64 build; behaviour when the real taskbar is hidden |
| Start menus (95 through 7) | Open-Shell (free), StartAllBack (paid, no redistribution) | ARM64 support for Open-Shell is uncertain |
| Windows 10 taskbar and Start | ExplorerPatcher | Its replacement taskbar on 24H2 ARM64 |
| Aero glass blur | DWMBlurGlass or similar | ARM64; CPU cost under this VM |
| Visual styles themselves | Community `.msstyles` for XP, Vista, 7, 8 | Each must target Windows 11 24H2. Styles for older Windows 11 builds often fail |
| Program Manager, File Manager | Microsoft's open-source WinFile; community Program Manager recreations | ARM64 or runs acceptably under emulation |

Two rules follow from the ARM64 constraint. Tools that run as their own program (RetroBar,
a Program Manager clone) can be x64 and run emulated. Tools that load inside `explorer.exe`
or `dwm.exe` must be native ARM64, with no workaround.

## 6. Changes required in this repo

### 6.1 Image pipeline (`scripts/`, `tools/`)

1. **New build stage, for example `scripts/apply_skin.py`,** run after
   `optimize_windows_lite.py`. Reuse its pattern: clone the image with `cp -c`, build a WinPE
   helper disk, boot it headless, apply changes, shut down.
2. **A way to carry files into the image.** Today only a handful of small files are added to
   WinPE's `boot.wim`. Skins need hundreds of megabytes of themes, icons, sounds and
   installers. Put a payload folder on the WinPE helper disk (it is already a 2.2 GB FAT32
   volume) and copy it to `C:\Skins\` during the offline pass.
3. **A skin manifest format.** One folder per skin with: asset list, registry files for the
   machine hive and the user hive, an optional first-logon script, and which host chrome it
   pairs with.
4. **A fetch script with checksums** for third-party tools and assets, saving into `work/`,
   matching how the README already treats the Windows ESD, firmware and drivers.
5. **A live-session install step.** Signature patchers and Explorer patchers generally need
   to run inside a logged-in Windows. Either extend `FirstLogonCommands` in the provisioning
   script or add a `RunOnce` entry during the offline pass, then boot once headless, wait for
   a "done" marker file, and shut down. The existing phase-2 OOBE boot is the model.
6. **Reconcile with the optimiser's visual settings.** Each skin's user-hive registry file
   overrides the optimiser's values where needed (animations, font smoothing, transparency).
   ClearType should be off for the 3.1 through 2000 skins to look right.
7. **Parameterise the user name** now hardcoded as `Ammaar` in two scripts.
8. **Possibly extend `tools/opt_hive_arm64.c`.** It exists because some registry keys are
   locked against even administrators. Theme-related keys may hit the same wall.
9. **Makefile targets** such as `make skin SKIN=xp`.

### 6.2 Runtime skin switching (optional, recommended later)

Two models:

- **One image per skin.** Simple and isolated. Costs several gigabytes of phone storage per
  skin, and installed apps and files are not shared.
- **One image, all skins installed, a switcher inside Windows.** A small script per skin
  swaps the theme, starts or stops the shell tools, and restarts Explorer. This matches your
  "same Windows 11, different costume" idea and is what I recommend once two or three skins
  work on their own. Classic-theme and Program Manager skins may need a sign-out to switch.

For the second model, add one endpoint to `host/share.c` (for example
`PUT /.henge/skin`) so the guest reports the active skin and the iOS app changes its frame to
match. `tools/winios_browser_arm64.c` already shows how a tiny guest program calls these
endpoints.

### 6.3 Host chrome (`ios/Sources/`)

1. **Introduce a theme object.** Replace the inline colour literals in `SessionView.swift`
   and `LaptopDeckView.swift` with named values (deck top, deck bottom, key face, key legend,
   bezel, accent, pill background). This is a mechanical refactor and the prerequisite for
   everything else in this layer.
2. **Per-era logo.** `WindowsLogoMark` draws the flat four-pane Windows 11 logo. It appears
   on the Win key, the lid, the command pill and the boot overlay. It needs variants: the
   waving flag (3.1 through 2000), the XP flag, the Vista and 7 orb, the tilted Windows 8
   logo, the Windows 10 logo.
3. **Per-era text.** Hardcoded strings to make skin-dependent: `SURFACE`,
   `SURFACE LAPTOP ULTRA`, `DEVICE_NAME = SURFACE_DUO_ULTRA`, `WIN11 ARM64 · 4.5 GB`, the
   `Start` pill label, and `CFBundleDisplayName: Windows 11` in `ios/project.yml`.
4. **Per-era hardware look, as far as you want to take it.** The deck is currently a dark
   blue aluminium laptop. Natural pairings: beige plastic with a CRT-style bezel for 3.1 and
   95, grey or black business laptop for 98 and 2000, silver and blue for XP, glossy black
   for Vista and 7. This is drawing work in SwiftUI with no technical risk.
5. **Keyboard details.** The function row has a Copilot key and the layout is a 2020s
   Surface. Keyboards had no Windows key before 1995. For the 3.1 skin, decide whether to
   hide the key for looks or keep it for function.
6. **Host boot overlay.** `surfaceHeroBootOverlay` shows while the VM starts. It can become
   a period splash (the 95 clouds, the XP progress bar). This is the only "boot screen" that
   is cheap to change.
7. **Launch buttons.** For skins that replace Explorer, the Win+R typing trick stops working.
   Replace it with a small guest-side agent that polls a `/.henge/` endpoint for "launch this"
   requests, or keep a hidden Explorer-independent hotkey helper in the guest.
8. **Safari sheet tint** in `VMViewModel.swift` is also hardcoded to the blue palette.

### 6.4 Things to measure before designing assets

- **Screen resolution and scaling.** I did not find where the guest resolution is set; it
  comes from the firmware's framebuffer. Old skins use fixed-size bitmaps made for 96 DPI.
  If Windows is running at 150% or 200% scaling, classic elements will be either tiny or
  blurry. This needs to be measured on the Mac runner first, and may call for a lower guest
  resolution for the oldest skins.
- **Idle memory and CPU per skin.** The README quotes 1.46 GB idle. Each added shell tool
  costs memory inside a 4.5 GB guest.

## 7. What is out of reach or not worth it

- **Windows' own boot screen** (the spinner and logo before logon). Changing it means
  patching boot files. One repo-specific possibility: Windows shows a firmware-supplied logo
  if the firmware's ACPI tables include one, and `devices/acpi.c` builds those tables by
  hand, so a per-skin boot logo could be supplied from the host. Treat as a stretch goal.
- **The logon screen.** Auto-logon is already configured, so it is barely seen. Skip.
- **Modern apps** (Settings, Edge, Store apps). See the fidelity ceiling in section 3.
- **Period-correct behaviour.** Out of scope by your own definition.

## 8. Legal and licensing

- Original Microsoft wallpapers, sounds, icons, fonts and visual styles are copyrighted. Do
  not commit them. Either fetch them at build time from a copy you own, or use community
  recreations and check each one's licence.
- The Windows logos are trademarks. Drawing them in the iOS chrome is the same exposure the
  upstream repo already has with its Windows 11 logo, but worth knowing if you publish.
- The repo is GPL-2.0-or-later. Your changes inherit that.
- Third-party tools have their own terms. StartAllBack is paid and cannot be bundled.
  A fetch-at-build approach avoids redistribution questions for the rest.

## 9. Suggested order of work

| Phase | Goal | Output |
| :--- | :--- | :--- |
| 0 | Get the stock image building and booting on an Apple Silicon Mac | A working `build/winvm` session |
| 1 | Compatibility spike: try each tool in section 5 by hand in the running VM | A table of what works on 26100 ARM64. This decides which families are feasible |
| 2 | Skin pipeline: payload transport, manifest, `apply_skin.py`, screenshot check | One command produces a skinned image |
| 3 | First skin: Windows 10 (lowest risk, proves the pipeline) | `make skin SKIN=win10` |
| 4 | XP, then Windows 7 | Proves unsigned visual styles and a replacement Start menu |
| 5 | Windows 8 and 8.1 | |
| 6 | Classic family: 98 first, then 95, 2000, Me, NT 4 as asset swaps | Proves classic theme |
| 7 | Windows 3.1 | Proves shell replacement and the launch agent |
| 8 | Host chrome refactor and per-era frames | Can run in parallel from phase 2 onward |
| 9 | Runtime switcher and `/.henge/skin` endpoint | One image, all skins |

Phase 1 is the gate. If the signature patcher or the classic-theme switch does not work on
this build and architecture, families 3 through 6 change shape or drop out.

## 10. What you need that you do not have yet

- **An Apple Silicon Mac with Xcode and Homebrew.** The provisioning scripts call `hdiutil`,
  `diskutil` and Homebrew's `wimlib-imagex`, and the VM engine itself is ARM64-on-ARM64. None
  of this builds or runs on Windows, Linux or an Intel Mac.
- **A Windows 11 24H2 ARM64 ESD** from Microsoft, placed where the README says.
- **For Layer B only: an iOS device the app runs on.** The app targets iOS 17 and later on
  iPhone and iPad and has a manual posture toggle for devices without a hinge, but the
  launch scripts and the JIT setup are written for and tested on the iPhone Duo only.
  Whether it runs on another device is unknown from the source. Without one, Layer B can be
  written but not seen, and the project's result is a skinned Windows VM on your Mac.

## 11. Decisions for you

1. **Is a Mac-only result acceptable,** or is running on an iOS device part of the goal?
   This decides whether Layer B is worth doing at all.
2. **Which versions matter most?** Windows 10, XP and 7 are a realistic first milestone.
   3.1 costs about as much as all the others combined.
3. **One image per skin, or a runtime switcher?**
4. **How strict on fidelity?** "Recognisable at a glance" is achievable for every family.
   "Pixel-accurate" is not, because of the modern-app ceiling.
5. **Paid tools allowed?** StartAllBack is the most polished option for several families.
