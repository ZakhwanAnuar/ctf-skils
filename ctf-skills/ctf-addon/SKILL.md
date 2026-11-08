---
name: ctf-addon
description: Add-on skill for the ljagiello/ctf-skills suite. Covers gaps the upstream repo leaves: Attack-Defense play, Windows privilege escalation, cloud/AWS exploitation, Android mobile quick-reference, Kali environment setup, and competition speed strategy. Use alongside the ctf-* skills, not instead of them.
license: MIT
compatibility: Kali Linux (rolling), filesystem agent with bash + Python 3 + internet.
allowed-tools: Bash Read Write Edit Glob Grep Task WebFetch WebSearch
---

# CTF Add-On — Gap Coverage

Upstream `ctf-skills` is deep on technique but light in five areas. These files fill them.

- [kali-setup.md](kali-setup.md) — One-shot Kali environment build: full tool install script, wordlist prep, gdb/pwndbg, Ghidra, python venv, service checks. Run at competition start.
- [attack-defense.md](attack-defense.md) — A/D playbook: round-0 flow, patch strategy, sploit automation, flag rotation tracking, infrastructure defense. Upstream is jeopardy-only.
- [windows-privesc.md](windows-privesc.md) — Windows privilege escalation quick reference: service abuse (registry, unquoted paths, DLL hijack), AlwaysInstallElevated, tokens/impersonation, UAC bypass list, SeXxxPrivilege abuse table. Upstream covers Linux privesc only.
- [cloud-aws.md](cloud-aws.md) — AWS/GCP/Azure CTF patterns beyond metadata SSRF: IAM privilege-escalation policy paths, S3 enumeration, ECR/lambda abuse, gopher-to-RCE chains, per-cloud CLI cheat lines.
- [mobile-android.md](mobile-android.md) — Android quick reference: APK teardown flow, Frida hook cookbook (SSL pinning, root detection bypass, secret dump), objection one-liners, network capture setup. Upstream covers iOS RE and Android forensics but not the exploitation flow.
- [speed-playbook.md](speed-playbook.md) — Competition strategy: category triage order by solve-count economics, time-boxing rules, parallelization with subagents, CTFd API flag spam, team note discipline.

## Routing

| Situation | File |
|---|---|
| Competition just started, fresh box | kali-setup.md first |
| A/D event announced | attack-defense.md |
| Got a shell as low-priv user on Windows box | windows-privesc.md |
| SSRF gave cloud creds, now what | cloud-aws.md |
| APK given, need dynamic analysis | mobile-android.md |
| 6 hours left, 9 unsolved challenges | speed-playbook.md |
