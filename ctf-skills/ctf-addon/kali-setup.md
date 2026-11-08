# Kali Setup — Competition Start

Run this block at round start on a fresh Kali. Idempotent — safe to re-run.

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y seclists wordlists gdb gdb-multiarch qemu-user-static \
  binwalk foremost exiftool outguess steghide volatility3 jq file android-tools \
  sqlmap nikto dirsearch feroxbuster gobuster ffuf hashcat john hashid \
  testdisk photorec foremost sleuthkit tcpdump tshark yara radare2

# Wordlists
sudo gunzip -f /usr/share/wordlists/rockyou.txt.gz
ln -sf /usr/share/wordlists/rockyou.txt ~/rockyou.txt

# Python toolchain (dedicated venv to avoid pip/system conflicts)
python3 -m venv ~/ctfvenv && source ~/ctfvenv/bin/activate
pip install pwntools ROPgadget ropper angr z3-solver frida-tools \
  PyCryptodome gmpy2 sympy requests beautifulsoup4

# Ruby gems
sudo gem install one_gadget seccomp-tools

# pwndbg (pick ONE debugger plugin)
git clone https://github.com/pwndbg/pwndbg ~/tools/pwndbg && cd ~/tools/pwndbg && ./setup.sh
echo "source ~/tools/pwndbg/gdbinit.py" >> ~/.gdbinit

# Ghidra (skip if already installed)
cd ~/tools && wget -q https://github.com/NationalSecurityAgency/ghidra/releases/latest/download/ghidra_11.3.2_PUBLIC_20250415.zip
# check ghidra-sre.org for current version, then: unzip && ./ghidraRun

# Tools needing git clone
cd ~/tools
git clone https://github.com/ticarpi/jwt_tool
git clone https://github.com/Ganapati/RsaCtfTool && cd RsaCtfTool && pip3 install -r requirements.txt && cd ..
git clone https://github.com/ambionics/phpggc
git clone https://github.com/frohoff/ysoserial  # needs: mvn package or download release jar
git clone https://github.com/niklasb/libc-database && cd libc-database && ./get ubuntu 2>/dev/null &

# stegsolve
cd ~/tools && wget -q https://github.com/eugenekolo/sec-tools/raw/master/stego/stegsolve/stegsolve/Stegsolve.jar -O stegsolve.jar

# Workspace + notes
mkdir -p ~/ctf/{recon,loot,notes} && echo "# CTF notes — $(date)" > ~/ctf/notes.md
```

## Sanity checks (30 seconds)

```bash
source ~/ctfvenv/bin/activate
python3 -c "import pwn; print('pwntools ok')"
gdb -q -ex quit 2>&1 | grep -i pwndbg        # pwndbg banner = ok
checksec --file=/bin/ls                       # pwntools checksec
ffuf -h >/dev/null && echo ffuf ok
ls /usr/share/seclists/Discovery/Web-Content/common.txt
java -version 2>&1 | head -1                  # Ghidra needs 17+
```

## Burp setup

- Install cert in Firefox, set proxy 127.0.0.1:8080 (FoxyProxy).
- Load extensions: J2EEScan, JWT Editor, Turbo Intruder (for race conditions).
- Project file per challenge: `/root/ctf/loot/burp-<chal>.burp`.

## Network defaults

```bash
# VPN check
ip a show tun0 2>/dev/null || echo "no vpn"
# Attack box IP for reverse shells
export ATTACKIP=$(ip -4 addr show tun0 2>/dev/null | grep -oP '(?<=inet )[\d.]+' || ip -4 addr show eth0 | grep -oP '(?<=inet )[\d.]+')
echo $ATTACKIP
# Persistent listeners — one per common port, run in tmux
sudo apt install -y tmux
```
