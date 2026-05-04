# Ocarina of Time N64 Anti-Piracy Analysis

A reverse engineering analysis of three CIC-based anti-piracy mechanisms in the N64 edition of Ocarina of Time, and the minimal byte-level patches that disable each one.

The N64 cartridge format includes a security chip (the CIC) that the console verifies at boot. A second layer of protection lives inside the game itself: routines scattered throughout the code read CIC-derived state at runtime and sabotage gameplay if the expected values are not present. Unlike the boot check, these routines trigger deep into the game.

This document covers three of those routines: the fishing pond fish-release sabotage, the Zelda hair deformation and the castle escape gate sabotage. This text explains how each one works at the MIPS instruction level along with the patch that disables it.

These mechanisms are specific to the N64 cartridge release of the game. The GameCube bonus disc editions that contain Ocarina of Time do not have these checks at all. The relevant code paths are compiled out in those builds.

## Background

The running game does not read the CIC chip directly in these routines. The fishing and castle-escape routines validate residue left in RDRAM by the CIC-6105/IPL3 boot process. The game preserves one of those words before clearing RAM during startup, and later checks it in the fishing minigame. The castle gate routine reads another word directly from low RDRAM, where the IPL3 boot process leaves copied boot/RSP code.

Zelda's hair check is slightly different: it reads the CIC identifier from the boot-parameter area at `0x80000310`. On a correctly booted CIC-6105 cartridge, this value is `6105`.

Historically, the most common cause of these checks tripping was period backup loaders such as the Bung Doctor V64, which required a real cartridge to be inserted as a CIC source. If the inserted cartridge used a different CIC variant than the loaded ROM expected, the checks would fail. Early flashcarts without proper CIC handling and inaccurate emulators of the era could trip the checks for the same reason. Most modern accurate emulators reproduce the CIC behavior correctly and do not trigger these routines.

The protection routines below all follow the same general pattern: load CIC/IPL3-derived state, compare it against a hardcoded constant, and branch into sabotage behavior on mismatch. Because the constants and sabotage paths differ between routines, each one needs its own patch.

The behavior of each routine was identified by observing in-game effects in conditions where the checks trip and tracing the responsible code paths back through the ROM.

## Patches

### 1. Fishing pond

In the fishing minigame, a routine compares a value loaded from a fixed memory address against the constant `0xAD090010`. The result of the comparison is written to a byte in memory that controls whether hooked fish stay on the line. On a genuine cartridge the values match and the byte is set to `0`, and fishing works normally. On a mismatch the byte is set to `1`, and any fish that bites breaks free of the line almost immediately, making it impossible to land a catch.

The expected value `0xAD090010` is the machine code for `sw t1, 0x10(t0)` inside the cartridge IPL3 bootcode. In context, IPL3 has just loaded `6105` into `t1`, and `t0` points at `0xA0000300`, so this instruction writes the CIC id to the boot parameter area at `0xA0000310`. The game captures this boot residue from physical RDRAM before clearing memory and later checks it in the fishing pond.

The relevant instruction window in the ROM:

```mips
3C 01 AD 09   lui   $at, 0xAD09
34 21 00 10   ori   $at, 0x0010        ; $at = 0xAD090010
00 41 10 26   xor   $v0, $v0, $at      ; compare against loaded value
2C 42 00 01   sltiu $v0, $v0, 1        ; == check
2C 42 00 01   sltiu $v0, $v0, 1        ; logical NOT
3C 01 80 A5   lui   $at, 0x80A5        ; upper bits of target address
A0 22 8D 04   sb    $v0, -0x72FC($at)  ; store result to flag byte
```

The cleanest patch changes the source register of the final `sb` from `$v0` to `$zero`, so the value written to the flag is always `0` regardless of the comparison result. Only one byte changes:

```
A0 22 8D 04   ->   A0 20 8D 04
```

The unique signature `3C 01 80 A5 A0 22 8D 04` can be used to locate the patch site in the ROM.

### 2. Zelda's hair

In cutscenes featuring adult Zelda, first seen in the cutscene where Sheik's identity is revealed in the Temple of Time, a routine loads the CIC identifier from the boot-parameter area and compares it against `6105`. On match, the code branches *past* a matrix scaling call. On mismatch, the branch is not taken and the matrix is multiplied by `2.0f` (the constant `0x40000000` loaded via `lui at, 0x4000`), which inflates the geometry of Zelda's hair into the well-known oversized golden pentagon shape that Nintendo used as a visible piracy tell.

The relevant instruction window:

```mips
3C 19 00 00   lui   $t9, 0x0
8F 39 00 00   lw    $t9, 0($t9)        ; load boot-parameter CIC identifier
24 01 17 D9   li    $at, 6105          ; expected value
8F A2 00 94   lw    $v0, 0x94($sp)
13 21 00 08   beq   $t9, $at, +8       ; skip scaling on match
3C 01 40 00   lui   $at, 0x4000        ; 2.0f scale factor (sabotage path)
```

Replacing the conditional `beq` with an unconditional branch causes the scaling call to always be skipped:

```
13 21 00 08   ->   10 00 00 08
```

The unique signature `24 01 17 D9 8F A2 00 94 13 21 00 08` can be used to locate the patch site in the ROM.

### 3. Castle escape gates

After defeating Ganon's first form, Zelda and Link must flee the collapsing tower in a timed escape sequence. Along the way, gates block the corridor. Zelda is supposed to use her magic to open each gate so the player can pass through. The routine that controls these gates performs a low-level boot-residue check by reading physical RDRAM through uncached KSEG1 at `0xA00002E8` and comparing the result against `0xC86E2000`. On mismatch, the routine returns early without raising the gate. Zelda still plays the animation of opening it and runs straight through the closed bars, leaving the player stuck on the wrong side with no way to progress.

The expected value `0xC86E2000` is also a word from the CIC-6105 IPL3 bootcode, specifically from embedded RSP code copied into RDRAM during boot. The gate routine reads physical address `0x000002E8` through uncached KSEG1 and only opens the bars if that IPL3 residue is present.

The relevant instruction window:

```mips
3C 0E A0 00   lui   $t6, 0xA000
8D CF 02 E8   lw    $t7, 0x02E8($t6)   ; load value from 0xA00002E8
3C 01 C8 6E   lui   $at, 0xC86E
34 21 20 00   ori   $at, 0x2000        ; $at = 0xC86E2000
55 E1 00 09   bnel  $t7, $at, +9       ; branch to early return on mismatch
8F BF 00 14   lw    $ra, 0x14($sp)     ; delay slot
```

The simplest patch replaces the `bnel` with a `nop`:

```
55 E1 00 09   ->   00 00 00 00
```

A more elegant single-byte alternative changes the second register of the comparison from `$at` to `$t7`, causing the branch to compare `$t7` with itself. Since the values are always equal, the `bnel` condition can never be true.

```
55 E1 00 09   ->   55 EF 00 09
```

The unique signature `3C 01 C8 6E 34 21 20 00` can be used to locate the patch site in the ROM.

## Summary

| Routine | Trigger | Sabotage | Patch |
|---|---|---|---|
| Fishing pond | comparison value `!= 0xAD090010` | hooked fish always escape the line | `sb $v0` → `sb $zero` (1 byte) |
| Zelda's hair | CIC identifier `!= 6105` | hair geometry scaled by 2.0f into a pentagon | `beq` → unconditional `b` (2 bytes) |
| Castle escape gates | RDRAM boot residue `!= 0xC86E2000` | routine returns early, gates never open | second register changed (1 byte) |

All three routines share the same design: a runtime comparison against CIC/IPL3-derived state, with sabotage behavior on mismatch placed far enough into the game that casual testing would not surface it. All three can be neutralized with a minimal patch, one to two bytes per routine.
