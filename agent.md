# SILICON GAUNTLET // Retro Computing Challenge Agent & Architecture Guide

## 1. Project Overview
**Silicon Gauntlet** is an interactive, self-contained single-page web dashboard designed for retro computing enthusiasts running authentic vintage iron and cycle-accurate emulators (e.g., true 8088 XT / Turbo XT, ESP32 8088 cycle-accurate emulation, 486 DX/DX2, and Socket 7 Pentium P75–P200).

The application randomly dispenses historically authentic software challenges that push, stress, or break specific vintage microprocessor architectures.

---

## 2. Hardware Fleet Profiles

### Tier 1: 8088 / XT ("The Crawl")
* **Clock Speeds**: 4.77 MHz (IBM 5150/5160 baseline) to 8–10 MHz (Turbo XT).
* **Architecture**: 16-bit internal, 8-bit external data bus, real mode only, non-pipelined instruction execution.
* **Memory Limits**: $\le 640\text{ KB}$ conventional base RAM (plus UMB / expanded memory tricks on later chipsets).
* **Display Hardware**: CGA (320x200 4-color palettes, 640x200 monochrome, NTSC composite color artifacting), early EGA, MDA (80x25 text).
* **Audio**: Intel 8253 Programmable Interval Timer (PIT) driving the 1-bit PC Speaker (square waves & PWM bit-banging), Tandy 3-Voice SN76496.
* **Key Pitfalls**:
  * Uncalibrated CPU timing loops causing physics runaways (e.g. *Sopwith*, *Digger*).
  * Video memory bus contention causing "CGA snow" on unblanked reads/writes.
  * Absence of math coprocessor (8087) requiring heavy 16-bit fixed-point trigonometry.

### Tier 2: 486 DX / DX2 ("The Sweet Spot")
* **Clock Speeds**: 33 MHz to 66 MHz (DX2-66 baseline) up to 100 MHz (DX4).
* **Architecture**: 32-bit internal/external bus, 8 KB unified L1 cache (Write-Through / Write-Back), 32-bit protected mode.
* **Memory Management**: DOS/4GW flat memory model (DPMI), `HIMEM.SYS` (XMS 3.0), `EMM386.EXE` / `QEMM` (EMS 4.0 64 KB page frames).
* **Display Hardware**: VGA Mode 13h ($320 \times 200$ 256 colors), Mode X/Mode Y unchained planar modes ($320 \times 240$ / $320 \times 200$), VESA Local Bus (VLB).
* **Audio**: Sound Blaster Pro / 16 / AWE32, Gravis UltraSound (GUS hardware mixing), Roland MT-32 / LAPC-I, General MIDI via MPU-401 UART.
* **Key Pitfalls**:
  * Unthrottled 286/386 loops running too fast (*Wing Commander*).
  * Origin Systems' "Voodoo Memory Manager" (*Ultima VII*) failing under any standard XMS/EMS TSR.
  * Squeezing $\ge 590\text{–}600\text{ KB}$ conventional free RAM while loading mouse, CD-ROM (`MSCDEX`), and sound card drivers.
  * ISA bus DMA channel conflicts and L2 cache wait-state tuning.

### Tier 3: Pentium P75–P200 ("Raw Crunch")
* **Clock Speeds**: 75 MHz to 200+ MHz (Socket 5 / Socket 7, Classic & MMX).
* **Architecture**: Superscalar execution (U-pipe and V-pipe instruction pairing), pipelined floating-point unit (FPU), branch prediction, 64-bit external data bus, PCI bus.
* **Memory & OS**: 16 MB to 64 MB EDO/SDRAM, Windows 3.11 for Workgroups / WinG, Windows 95, DOS32A / DOS/4GW extenders.
* **Display Hardware**: VESA VBE 2.0 Linear Frame Buffers (LFB) via UniVBE / Scitech Display Doctor, $640 \times 480$ / $800 \times 600$ SVGA 256 colors.
* **Audio**: Sound Blaster AWE32/64, General MIDI wavetable daughterboards, DirectSound 3D.
* **Key Pitfalls**:
  * Borland Pascal/C++ `Runtime Error 200` (division by zero in 55ms timer calibration loops at $>200\text{ MHz}$).
  * Carmack's pipelined FPU loop in *Quake* (`FXCH` overlap) exposing 486 and Cyrix 6x86 FPU stalls.
  * MTRR (Memory Type Range Register) write-combining configuration in DOS for high SVGA frame throughput.

---

## 3. Application Architecture

Silicon Gauntlet is designed as a **zero-dependency, single-file HTML/CSS/JS** application located at `index.html`.

### File Organization
```
RetroChallenge/
├── index.html       # Complete application (UI, CSS themes, Web Audio synth, database, logic)
└── agent.md         # Architecture, hardware instructions, and maintenance guide
```

### Key Subsystems in `index.html`

1. **CSS Theme Engine**:
   * Uses CSS variables `--bg-color`, `--text-main`, `--panel-border`, `--border-bright`.
   * Four switchable themes:
     * `amber`: P31 Amber CRT phosphor
     * `green`: P1 Green CRT phosphor
     * `turbovision`: Borland / Norton Commander blue & cyan IDE aesthetic
     * `bios`: Classic American Megatrends / Phoenix monochrome BIOS
   * Visual filters: CRT scanline gradient, chromatic aberration tint, and radial vignette.

2. **Synthesized Web Audio Engine (`RetroAudioEngine`)**:
   * Generates all sound effects dynamically via `AudioContext` without any external audio files.
   * `beep(freq, duration, type, volume)`: PIT 8253 square-wave emulation.
   * `rollChatter()`: Multi-frequency bursts simulating mechanical floppy head seek stepper chatter during dispension.
   * `dispenseComplete()`: IBM PC POST power-on double beep.
   * `conquerFanfare()`: 4-note ascending C-major arpeggio (C5, E5, G5, C6) in 8-bit square waves.

3. **Data Model**:
   * Stored in `state.challenges` and synchronized with `localStorage` (`SILICON_GAUNTLET_DB_V1`).
   * Schema:
     ```json
     {
       "id": "doom-1993",
       "title": "Doom",
       "year": 1993,
       "tier": "486",
       "tierLabel": "486 (The Sweet Spot)",
       "platform": "MS-DOS 5.0 - 6.22 (DOS/4GW 32-bit Protected Mode)",
       "specs": {
         "ram": "4 MB RAM (8 MB Recommended)",
         "video": "VGA 320x200 Mode Y (Unchained Planar)",
         "sound": "Sound Blaster 16 (OPL3 FM / General MIDI) or GUS",
         "storage": "IDE / SCSI HDD (Approx 15 MB Free)"
       },
       "hardwareChallenge": "Achieve locked 35 FPS in timedemo benchmark: doom -timedemo demo3...",
       "whyItTests": "Carmack's BSP tree and Mode Y planar VGA rendering demand...",
       "tags": ["DOS/4GW", "Mode Y Planar", "35 FPS Cap"],
       "conquered": false,
       "conqueredDate": null,
       "notes": ""
     }
     ```

4. **Persistence & Data Management**:
   * `localStorage` keys:
     * `SILICON_GAUNTLET_DB_V1`: Full challenge dataset + notes + conquest records.
     * `SILICON_GAUNTLET_THEME`: Active color scheme.
     * `SILICON_GAUNTLET_SOUND`: Audio toggle state.
     * `SILICON_GAUNTLET_CRT`: Scanline overlay toggle state.
   * JSON Export/Import: Allows exporting the database to a `.json` file or restoring from a pasted backup buffer.
   * Factory Reset: Reverts to the built-in 32 curated titles.

---

## 4. Curated Challenge Library Reference

### 8088 / XT Titles (11)
1. **Sopwith (1984)**: 4.77 MHz timing loop & NTSC CGA composite color burst artifacting.
2. **Digger (1983)**: INT 08h timer interrupt hijack & PC Speaker PWM polyphony.
3. **Prince of Persia (1989)**: 512 KB conventional memory bank swapping and rotoscoped frame overlays.
4. **Alley Cat (1984)**: PC Booter, direct memory segment `0xB800` writes, NTSC color burst phase shift.
5. **King's Quest: Quest for the Crown (1984)**: AGI vector line-art polygon rasterization and flood fills.
6. **Microsoft Flight Simulator 2.0 (1984)**: 16-bit fixed-point trigonometry without an 8087 math coprocessor.
7. **Commander Keen 1: Marooned on Mars (1990)**: Carmack's Adaptive Tile Refresh on 8-bit ISA EGA hardware.
8. **The Ancient Art of War (1984)**: Real-time tactical squad AI loop saturating 8088 registers.
9. **California Games (1987)**: High-frequency timer interrupt digitized PCM audio over 1-bit PC Speaker.
10. **Planet X3 (2019)**: Modern 8088 stress test with CGA/EGA/VGA/Tandy autosensing.
11. **Castle Adventure (1984)**: CGA video bus contention ("snow") suppression during text-mode page writes.

### 486 Titles (11)
12. **Wing Commander (1990)**: Uncalibrated sprite scaling loop throttling via Turbo button / cycle governors.
13. **Doom (1993)**: DOS/4GW Mode Y planar VGA rendering, 35 FPS cap, and Sound Blaster DMA IRQ routing.
14. **Ultima VII: The Black Gate (1992)**: Voodoo Memory Manager clean boot ($\ge 595\text{ KB}$ conventional RAM, no `HIMEM`/`EMM386`).
15. **Star Wars: TIE Fighter (1994)**: 64 KB EMS 4.0 page frame bank-switching with iMUSE interactive MIDI.
16. **Duke Nukem 3D (1996)**: Ken Silverman Build engine 16-bit 44.1 kHz DMA mixing on GUS / SB16.
17. **Strike Commander (1993)**: The ultimate 486 killer—Gouraud-shaded textured 3D cockpits on 8 MB RAM.
18. **System Shock (1994)**: 6-DOF physics engine in SVGA $640 \times 480$ stressing VLB/PCI bus bandwidth.
19. **Descent (1995)**: Full polygonal 3D matrix math with BIOS L1 Write-Back cache optimization.
20. **The 7th Guest (1993)**: Double-speed CD-ROM streaming, `MSCDEX`, and VESA VBE 1.2 SVGA compatibility.
21. **Wing Commander: Privateer (1993)**: 590 KB conventional memory squeeze with joystick calibration.
22. **SimCity 2000 (1993)**: Win 3.11 WinG GDI blitting and cellular automata simulation ticks.

### Pentium Titles (10)
23. **Quake (1996)**: Pipelined FPU assembly (`FXCH` overlap) timedemo benchmark (`quake -nosound -timedemo demo1`).
24. **The Need for Speed: SE (1996)**: VESA VBE 2.0 Linear Frame Buffer (LFB) with UniVBE TSR.
25. **Blood (1997)**: SVGA $800 \times 600$ Build engine memory bus saturation and Q-Sound 3D audio.
26. **Tomb Raider (1996)**: Pure software 3D superscalar instruction pairing (U/V pipe) at $640 \times 480$.
27. **Interstate '76 (1997)**: Heavy software physics and vehicle deformation demanding P166+.
28. **Screamer 2 (1996)**: High-resolution VESA $640 \times 480$ software rally racing stressing L2 pipeline latency.
29. **MechWarrior 2 (Pentium Edition, 1995)**: Intel OEM branch-prediction-optimized assembly routines.
30. **Unreal (1998, Software Fallback)**: Tim Sweeney's `-nohardware` MMX software renderer.
31. **Fallout (1997, DOS Port)**: DOS32A extender replacement and high-resolution isometric page swapping.
32. **Jazz Jackrabbit 2 (1998)**: Borland CRT `Runtime Error 200` division-by-zero calibration loop patch.

---

## 5. Controls & Keyboard Shortcuts

| Shortcut | Action |
|:---|:---|
| `Space` | Dispense / roll a random hardware challenge matching active filters |
| `C` | Toggle "Mark as Conquered" state on currently active challenge |
| `T` | Cycle color theme (Amber $\rightarrow$ Green $\rightarrow$ Turbo Vision $\rightarrow$ BIOS) |
| `M` | Mute / Unmute synthesized audio effects |
| `L` | Toggle between Challenge Dispenser and Full Archival Codex view |

---

## 6. How to Extend & Maintain

### Adding New Challenges to the Factory ROM
Edit `index.html` and append an object to the `FACTORY_CHALLENGES` array:
```javascript
{
  id: "game-slug-year",
  title: "Game Title",
  year: 1994,
  tier: "486", // "8088", "486", or "pentium"
  tierLabel: "486 (The Sweet Spot)",
  platform: "MS-DOS 6.22",
  specs: {
    ram: "4 MB Conventional / XMS",
    video: "VGA Mode 13h 320x200",
    sound: "Sound Blaster 16",
    storage: "IDE HDD"
  },
  hardwareChallenge: "Describe the specific configuration or survival feat...",
  whyItTests: "Explain the architectural bottleneck...",
  tags: ["Tag1", "Tag2"]
}
```

### Adding New Sound Effects
Use the `sound.beep()` primitive on the `RetroAudioEngine` instance:
```javascript
sound.beep(frequencyInHz, durationInSeconds, waveformType, volume);
// Example: sound.beep(440, 0.1, "square", 0.15);
```
Waveform types supported: `"square"` (authentic 8253 PIT tone), `"sawtooth"`, `"triangle"`, `"sine"`.
