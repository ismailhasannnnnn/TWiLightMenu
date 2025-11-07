# TWiLight Menu++ Development Guide

## Project Overview
TWiLight Menu++ is a Nintendo DS/DSi/3DS menu replacement composed of **multiple independent sub-projects** that compile to `.srldr` files. Each subfolder (`booter/`, `romsel_dsimenutheme/`, `settings/`, etc.) is a standalone NDS application compiled separately and loaded dynamically by the booter.

### Architecture: Multi-Binary System
- **`booter/`** → `BOOT.NDS` - Main entrypoint for SD card/CFW users
- **`booter_fc/`** → `_DS_MENU.DAT`, `akmenu4.nds`, etc. - Flashcard entrypoints
- **`romsel_dsimenutheme/`** → `dsimenu.srldr` - DSi/3DS/Saturn/Homebrew UI themes
- **`romsel_aktheme/`** → `akmenu.srldr` - Wood UI theme
- **`romsel_r4theme/`** → `r4menu.srldr` - R4/GBC UI themes
- **`settings/`** → `settings.srldr` - Settings menu
- **`title/`** → `main.srldr` - Boot splash screen
- **`quickmenu/`** → `mainmenu.srldr` - DS Lite classic UI
- **`slot1launch/`**, **`manual/`**, **`imageview/`**, **`gbapatcher/`** - Additional utilities

Each `.srldr` is loaded via `runNdsFile()` from the booter - these are NOT linked together at compile time.

## Build System

### Standard Build Commands
```powershell
# Full build (all sub-projects + packaging)
make package

# Build specific component
cd romsel_dsimenutheme
make dist

# Clean everything
make clean
```

### Docker Build (Recommended)
```powershell
# Build everything
.\compile_docker.ps1 package

# Clean with Docker
.\compile_docker.ps1 clean

# Single component
cd settings
..\compile_docker.ps1 dist
```

**Critical**: Docker builds use `devkitpro/devkitarm:20241104`. Do NOT mix native Windows builds with Docker builds - run `.\compile_docker.ps1 clean` before switching.

### Build Output Structure
All builds output to `7zfile/` matching the distribution structure:
- `7zfile/_nds/TWiLightMenu/*.srldr` - UI modules
- `7zfile/DSi&3DS - SD card users/BOOT.NDS` - Main booter
- `7zfile/Flashcard users/` - Flashcard variants
- `7zfile/debug/` - ELF files for debugging

## devkitARM-Specific Patterns

### Each Sub-Project Has Dual ARM Cores
All sub-projects compile **both ARM7 and ARM9 binaries**:
- `arm9/` - Main application logic (32 MB RAM, graphics, UI)
- `arm7/` - Sound processor (4 MB RAM, audio via maxmod9)
- `ndstool` links them into one `.nds` file

### Graphics Pipeline
Graphics use **grit** for conversion:
```makefile
%.s %.h : %.png %.grit
    grit $< -fts -o$*
```
`.grit` files define palette/tile settings. Converted to `.s` assembly, then linked as binary blobs.

### Common Code Sharing
Shared code lives in `universal/`:
- `universal/include/common/` - Headers for settings, bootstrapping, system detection
- `universal/source/` - Shared implementations, lodepng, font rendering
- `universal/bootloader_*/` - Shared bootloader binaries compiled once, used by multiple sub-projects

Sub-projects include universal headers via `-iquote` and compile bootloaders into `data/load.bin` or `data/bootstub.bin`.

## Critical File Transformations

### NDS Header Patching (Post-Build)
After `ndstool` creates `.nds` files, **`patch_ndsheader_dsiware.py`** patches headers for DSi/3DS compatibility:
```makefile
$(PYTHON) ../patch_ndsheader_dsiware.py $(TARGET).nds --twlTouch --structParam
```
This sets TWL-mode flags, touch support, and device lists. Never skip this step for DSi targets.

### Animated Banner Patching
Booter uses custom animated banners:
```makefile
$(PYTHON) animatedbannerpatch.py $@ twl_banner.bin
```

## Platform-Specific Conventions

### File Path Prefixes
- `fat:/` - FAT filesystem (SD card or flashcard)
- `sdmc:/` - SD card specifically (DSi/3DS)
- Booter uses `isRunFromSD()` to detect execution context

### Hardware Detection Patterns
```cpp
#include "common/systemdetails.h"
sys().isDSiMode()  // DSi/3DS vs DS Phat/Lite
sys().isRunFromSD() // SD card vs flashcard
dsiFeatures() // DSi-enhanced mode available
```

### Bootstrap Integration
TWiLight Menu++ launches ROMs via **nds-bootstrap** (separate project). Configuration written to `sd:/_nds/nds-bootstrap.ini`:
```cpp
bootstrapini.SetInt("NDS-BOOTSTRAP", "BOOST_VRAM", perGameSettings_boostVram);
```
See `common/bootstrapsettings.h` and donor/save/compatibility maps in `universal/include/`.

## Development Workflow

### Adding a New UI Feature
1. Identify which sub-project(s) need changes (usually `romsel_*` for UI)
2. Modify ARM9 code in `<subproject>/arm9/source/`
3. Update graphics in `<subproject>/gfx/` with `.grit` descriptors
4. Test build: `cd <subproject> && make clean && make dist`
5. Full integration test: `make clean && make package` from root

### Localization
Translations managed via **Crowdin** (see `crowdin.yml`):
- Source strings: `*/nitrofiles/languages/en/*.txt`
- Never edit non-English files directly - they're auto-generated
- Test with different languages via settings menu

### Maintaining Custom Modifications

**Custom Save Directory Structure** (this fork):
- Saves stored at `/saves/romname/savefile.sav` (root of SD card)
- Modified in `quickmenu/arm9/source/main.cpp` and `romsel_dsimenutheme/arm9/source/main.cpp`
- Upstream uses `ms().saveLocation` enum (TWLMFolder/GamesFolder/SDRoot) - we bypass this
- When merging upstream: Look for changes to save path logic around line ~1700-2550 in these files
- Key variables: `savepath`, `savename`, `saveNameFc` - ensure directory structure preserved

**Removed Dependencies**:
- `ms().fcSaveOnSd` setting was removed from upstream - don't reference it
- Old code had flashcard SD save toggle; current implementation doesn't need this check

### Common Pitfalls
- **FIFO communication**: ARM7 ↔ ARM9 use FIFO for IPC (`fifoSendValue32(FIFO_USER_02, 1)`)
- **VRAM banking**: Background/sprite VRAM must be mapped correctly via `BG_VRAM` macros
- **FAT timing**: SD operations are slow; show progress indicators for file copies
- **Submodule updates**: This repo uses submodules - always clone with `--recursive`
- **Docker vs Native**: Don't mix builds - run `.\compile_docker.ps1 clean` before switching
- **Build warnings**: `__sync_synchronize` and RWX segment warnings are normal for NDS homebrew

## Testing & Debugging
- Use `.elf` files in `7zfile/debug/` with no$gba or melonDS debugger
- Check console-specific builds: DSi vs 3DS vs Flashcard distributions differ
- Test bootstrap launch: Most ROM loading bugs are in bootstrap config, not TWiLight Menu++

## Troubleshooting Build Issues

### Docker Build Fails Early
**Symptom**: Build stops at `booter_fc` with DLDI file errors
**Cause**: Git submodule `booter_fc/flashcart_specifics/DLDI` not initialized
**Fix**: 
```powershell
git submodule update --init --recursive
```

### Compilation Errors After Upstream Merge
**Common Issues**:
1. **Save path variables changed**: Check for `saveLocation`, `savepath`, `savename` modifications
2. **Settings class members removed**: Grep for removed properties like `fcSaveOnSd`
3. **Bootstrap integration changes**: Look for `bootstrapini.SetInt/SetString` parameter changes

**Strategy**: 
- Compare upstream `main.cpp` changes in `romsel_*` and `quickmenu` folders
- Preserve custom directory creation logic: `mkdir("/saves/" + savename)`
- Ensure `savepath = "sd:/saves/" + savename + "/" + saveNameFc` pattern maintained

### Clean Build Required
If you see linking errors or "undefined reference" errors:
```powershell
.\compile_docker.ps1 clean
.\compile_docker.ps1 package
```

### Mixed Docker/Native Builds
**Symptom**: Linker errors about incompatible object files
**Cause**: Object files from different toolchain versions
**Fix**: Always clean before switching build methods

## Reference Files
- `README.md` - Component descriptions and credit list
- `Makefile` - Root build orchestration
- `universal/include/defaultSettings.h` - Default configuration values
- `.github/workflows/nightly.yml` - CI/CD packaging logic
