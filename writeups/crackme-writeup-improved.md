# Reverse Engineering Write-up: Base64-Obfuscated Crackme with a Hidden Backdoor

**Author:** Arshia
**Date:** 2026-09-04
**Source:** crackmes.one
**Binary:** `main.exe` — Windows PE, x86-64, MinGW-compiled
**Tools:** Radare2

---

## Table of Contents
1. [Overview](#1-overview)
2. [Reconnaissance](#2-reconnaissance)
3. [Analyzing `main`](#3-analyzing-main)
4. [Core Logic: `check_password`](#4-core-logic-check_password)
5. [The Backdoor](#5-the-backdoor)
6. [Post-Validation Flow](#6-post-validation-flow)
7. [Results & Skills Demonstrated](#7-results--skills-demonstrated)

---

## 1. Overview

This write-up documents the static analysis of a Windows crackme challenge from **crackmes.one**. The goal was to bypass the login mechanism and recover a valid password. Using Radare2, I traced the control flow, decoded an obfuscated string, and — as a bonus — uncovered an unintended backdoor left in the binary by the challenge author.

**Result:** two independent ways to pass authentication — the intended password and a hidden bypass condition.

---

## 2. Reconnaissance

The binary was loaded with full auto-analysis:

```
r2 -A main.exe
```

Listing functions (`afl`) surfaced three routines of interest:

```
0x140001690    6    115  sym.check_password
0x1400017cb   14    518  sym.main
0x1400014f5   19    411  sym.base64_decode
```

- `sym.main` — user-facing input loop
- `sym.check_password` — validation routine
- `sym.base64_decode` — custom decoder, used to hide the real password

---

## 3. Analyzing `main`

Disassembling `main` (`pdf @ sym.main`) revealed the program's flow:

1. **Setup** — clears the console and prints a banner.
2. **Attempt limit** — exactly 3 tries (`mov r8d, 3`, `cmp dword [var_4h], 2`).
3. **Input loop** — prompts `[Attempt %d/%d] Enter password:`, reads up to 50 bytes via `fgets`.
4. **Sanitization** — checks for `fgets` failure, computes length with `strlen`, strips the trailing newline.
5. **Validation** — passes the cleaned input to `sym.check_password`.
6. **Branching:**
   - **Success:** prints `ACCESS GRANTED` plus a `rand()`-generated `CR4CK3R_XXXX` ID, then exits the loop.
   - **Failure:** prints `ACCESS DENIED` and increments the attempt counter.
7. **Lockout** — after 3 failures, triggers a fake `SECURITY ALERT`, logs a fabricated IP, and exits.

---

## 4. Core Logic: `check_password`

### 4.1 Obfuscated password

The function loads a hardcoded Base64 string:

```
lea rax, str.Y3JhY2ttZTIwMjQ=
```

This is passed to `sym.base64_decode`, and the result is stored for comparison.

### 4.2 Primary check

```
call sym.strcmp
test eax, eax
jne  0x1400016d9
mov  eax, 1        ; success
```

If the decoded string matches user input, `check_password` returns `1`.

### 4.3 Decoding

- Encoded: `Y3JhY2ttZTIwMjQ=`
- Decoded: **`crackme2024`**

This is the intended, primary valid password.

---

## 5. The Backdoor

If `strcmp` fails, execution doesn't stop — it falls through to a second, undocumented check:

```
lea rdx, str.hack       ; "hack"
call sym.strstr
test rax, rax
je   0x1400016f8
mov  eax, 1              ; success
```

`strstr` scans the user's input for the substring `"hack"`. If found anywhere in the input (`hack`, `123hack456`, `hackme`, etc.), authentication succeeds regardless of the actual password.

This is a classic developer-left backdoor: a debug/testing shortcut that never got removed before the challenge was published.

---

## 6. Post-Validation Flow

**On success:** prints a congratulatory message, generates a random `CR4CK3R_XXXX` ID via `rand()`, prints it, exits cleanly.

**On lockout (3 failed attempts):** fires a fake `SECURITY ALERT`, claims to log the user's IP, forces exit. (Cosmetic — no real logging occurs.)

---

## 7. Results & Skills Demonstrated

| Solution | Input |
| :--- | :--- |
| Primary (intended) | `crackme2024` |
| Backdoor (unintended) | Any string containing `hack` |

**Takeaway:** obfuscation (Base64) is not encryption — a few minutes of static analysis fully defeats it. More importantly, this challenge is a good illustration of how leftover debug/backdoor logic (`strstr` fallback checks, hardcoded bypass strings) can silently undermine even a "working" authentication routine — a pattern worth checking for in real ICS/OT and embedded auth code, not just CTF binaries.

**Techniques used in this write-up:**
- Static disassembly and control-flow analysis with Radare2 (`afl`, `pdf`)
- Manual and automated Base64 decoding
- Identifying secondary/hidden authentication branches (`strstr`-based backdoors)
- Windows x86-64 PE / MinGW calling convention reading
