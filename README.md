# Reverse Engineering Writeups

Static and dynamic analysis writeups of crackmes and binary challenges, mostly solved with Radare2. Each writeup documents the full analysis process — disassembly, control-flow tracing, and any hidden logic uncovered along the way — rather than just the final answer.

## Writeups

| Writeup | Techniques |
|---|---|
| [Base64 Crackme + Backdoor](writeups/crackme-base64-backdoor.md) | Base64 decoding, hidden `strstr` auth bypass |

## Tools

- [Radare2](https://github.com/radareorg/radare2) — primary disassembler/analysis framework
- Python — used occasionally to script and verify solutions

## Structure

```
writeups/     individual challenge writeups (Markdown)
assets/       supporting screenshots, where relevant
```

## About

Maintained by [Arshia Alikhani](https://github.com/arshiaalikhani) as part of ongoing reverse engineering and ICS/OT security practice.
