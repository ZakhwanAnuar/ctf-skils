# Speed Playbook — Competition Strategy

Technique is upstream. This is the meta-game: how to score more points per hour.

## Triage at event start (first 15 min)

1. Open every challenge. Read every description. Note flag format, given files, point value.
2. Sort into three buckets:
   - **Fast wins** (low points, obvious category, files present) — do immediately, in parallel.
   - **Category-fit** (your strongest category) — start these within the first hour.
   - **Deep/chainy** (high points, multi-step) — time-box, revisit.
3. Start the slow background stuff first — directory bruteforce, wordlist cracks, large scans run while you think. Idle CPU = wasted points.
4. Rule of thumb: solve-value density = points / expected-hours. Work down that list, not the category list.

## Time-boxing

- 25 minutes no progress → write down the current hypothesis, switch challenges. Subconscious keeps working; the block usually dissolves on return.
- 3 dead hypotheses on the same challenge → the premise is wrong. Re-read the description. Something given is unused.
- Never "just try one more thing" past the box. That's the sunk-cost trap.

## Parallelization

- Every long-running scan/crack/brute in `tmux` — name the panes.
- Run independent technique branches concurrently (e.g., while sqlmap runs, manually read source).
- Subagent pattern: delegate enumeration tasks ("run this checklist, report findings") while you hold the exploit-writing context.
- Two-person split when playing as a team: one enumerates/reads, one exploits whatever comes back.

## The stuck loop (in order, every time)

1. Re-read the challenge description and title — slowly, twice.
2. What has the author given me that I haven't used? Every artifact is load-bearing.
3. Walk the category checklist (upstream SKILL.md files) — what haven't I run?
4. Check the obvious: flag-format grep, base64 layers, admin/admin, port 8080/8443, UDP not TCP.
5. Point-value sanity: a 100-point web challenge is one bug. A 500-point one is a chain — look for two primitives, not a harder single one.
6. Ask: what does the intended solution look like from the author's chair? They wrote it to be solvable.

## CTFd API — flag submission without the browser

```bash
# (upstream ctfd-navigation.md has the full client — the fast version:)
curl -s -X POST https://ctf.example.com/api/v1/challenges/attempt \
  -H "Content-Type: application/json" \
  -b "session=<cookie>" \
  -d '{"challenge_id": 5, "submission": "flag{...}"}'
# Batch script every candidate flag from notes — seconds, not minutes
```

## Notes discipline

- One file: `~/ctf/notes.md`. Every flag, every hypothesis, every dead end with WHY.
- Dead ends are the most valuable entries — teammates (and future-you) skip them.
- When a challenge breaks open, write the full solve steps immediately — writeups are often required, and memory fades fast.

## Endgame (last hour)

- Re-grep everything for the flag format — usually one near-miss is sitting in an old output file.
- Revisit the "almost" challenges — one missing step beats a fresh start.
- Submit every partial-format candidate; flag checkers are cheap, typing is expensive.
- Never leave an unsolved-but-solved-in-head challenge unsubmitted at time.
