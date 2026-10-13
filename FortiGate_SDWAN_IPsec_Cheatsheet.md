---
title: FortiGate SD-WAN + IPsec Cheatsheet
tags: [fortigate, sdwan, ipsec, cheatsheet]
---

# FortiGate SD-WAN + IPsec Cheatsheet (HQ: 701G, Branch: 50G)

> [!info] Тайлбар
> - Бүх IP, нэр нь **placeholder эсвэл ЖИШЭЭ**. Бодит утгаа өөрийн сүлжээнээс орлуулна.
> - Команд нь **FortiOS 7.0 – 7.6**-д зориулсан. Хувилбараас хамаарах газрыг `[Хувилбар]` гэж тэмдэглэв. Эхлээд `get system status`-аар хувилбараа шалга.
> - Config доторх `#` мөр нь тайлбар. CLI-д paste хийхдээ устга.
> - Interface нэр (`wan1`, `port1`, `lan` г.м.) загвар болон хувилбараас хамаарна. `get system interface physical` эсвэл GUI-аас баталгаажуул.

---

## 1. Network Plan

| Зүйл | HQ (701G) | Branch (50G) |
|---|---|---|
| Hostname | `<HQ_HOSTNAME>` | `<BR1_HOSTNAME>` |
| LAN subnet | `<HQ_LAN_SUBNET>` (жишээ: 10.10.0.0/24) | `<BR1_LAN_SUBNET>` (жишээ: 10.20.1.0/24) |
| LAN IP / interface | `<HQ_LAN_IP>` / `<HQ_LAN_IF>` | `<BR1_LAN_IP>` / `<BR1_LAN_IF>` |
| WAN1 IP / ISP GW | `<HQ_WAN1_IP>` / `<HQ_ISP1_GW>` | `<BR1_WAN1_IP>` / `<BR1_ISP1_GW>` |
| WAN2 IP / ISP GW | `<HQ_WAN2_IP>` / `<HQ_ISP2_GW>` | `<BR1_WAN2_IP>` / `<BR1_ISP2_GW>` |
| Tunnel нэр (WAN1) | `HQ-BR1-W1` | `BR1-HQ-W1` |
| Tunnel нэр (WAN2) | `HQ-BR1-W2` | `BR1-HQ-W2` |
| Tunnel IP (`[Шаардлагатай үед]`) | `<TUN1_HQ_IP>` (жишээ: 10.255.1.1) | `<TUN1_BR1_IP>` (жишээ: 10.255.1.2) |

> [!warning] MikroTik-тэй зөрчилдөхөөс сэргийл
> - `<BR1_LAN_SUBNET>`, tunnel subnet нь одоо MikroTik сүлжээнд ашиглагдаж буй subnet-тэй **давхцаж болохгүй**.
> - HQ FortiGate-ийг MikroTik-ийн ард (NAT-тай) тавих бол MikroTik дээр **UDP 500, UDP 4500**-ийг FortiGate WAN IP руу forward хийнэ.

---

## 2. SD-WAN

### 2.1 Эхлэхийн өмнө (чухал)

> [!danger] Interface-д reference байвал SD-WAN member болгож болохгүй
> Interface дээр static route, firewall policy, DHCP г.м. reference байвал member болгохгүй. Эхлээд reference-ийг арилгах эсвэл шинэ policy/route-ыг zone-оор дахин үүсгэнэ.
> Шалгах команд: `show system interface <WAN_IF>`, `diagnose sys checkused system.interface.name <WAN_IF>`

### 2.2 Zone, Member, Health Check, SLA, Rule (HQ болон Branch-д ижил бүтэц)

```
config system sdwan
    set status enable
    config zone
        edit "underlay"          # ISP WAN-уудын zone
        next
        edit "overlay"           # IPsec tunnel-уудын zone
        next
    end
    config members
        edit 1
            set interface "<WAN1_IF>"      # WAN1 member
            set zone "underlay"
            set gateway <ISP1_GW>          # ISP1 gateway
        next
        edit 2
            set interface "<WAN2_IF>"      # WAN2 member
            set zone "underlay"
            set gateway <ISP2_GW>          # ISP2 gateway
        next
        edit 10
            set interface "<TUNNEL1_NAME>" # IPsec tunnel (via WAN1)
            set zone "overlay"
        next
        edit 11
            set interface "<TUNNEL2_NAME>" # IPsec tunnel (via WAN2)
            set zone "overlay"
        next
    end
    config health-check
        edit "HC_INTERNET"
            set server "<HC_TARGET_IP>"    # Internet-ийн найдвартай target (ISP DNS гэх мэт)
            set protocol ping
            set interval 1000              # ms
            set failtime 5                 # 5 удаа алдвал member down
            set recoverytime 5
            set members 1 2                # Зөвхөн WAN member-үүд
            config sla
                edit 1
                    set latency-threshold 150
                    set jitter-threshold 30
                    set packetloss-threshold 2
                next
            end
        next
        edit "HC_HQ"
            set server "<HQ_LAN_IP>"       # Branch-д: HQ-ийн LAN IP. HQ-д: <BR1_LAN_IP>
            set protocol ping
            set interval 1000
            set failtime 5
            set recoverytime 5
            set members 10 11              # Зөвхөн tunnel member-үүд
            config sla
                edit 1
                    set latency-threshold 150
                    set jitter-threshold 30
                    set packetloss-threshold 2
                next
            end
        next
    end
    config service
        edit 1
            set name "Internet"
            set mode sla                   # SLA-д тулгуурласан сонголт
            set dst "all"
            config sla
                edit "HC_INTERNET"
                    set id 1
                next
            end
            set priority-members 1 2       # 1 нь primary, 2 нь backup
        next
        edit 2
            set name "To-HQ"
            set mode sla
            set dst "<HQ_LAN_ADDR_OBJ>"    # firewall address object
            config sla
                edit "HC_HQ"
                    set id 1
                next
            end
            set priority-members 10 11
        next
    end
end
```

- `[Хувилбар]` `config system sdwan` нь 6.4+-д ажиллана. 6.2 ба түүнээс өмнө `virtual-wan-link` байсан.
- `[Шаардлагатай үед]` Load-balance хэрэгтэй бол `set mode load-balance` ашиглана.
- `[Шаардлагатай үед]` Tunnel-д health-check хийхийн тулд tunnel interface-д IP байх ёстой (3-р хэсгийн Tunnel IP).
- Source IP-ээр зорилтот rule хийх бол service дотор `set src "<ADDR_OBJ>"` нэм.

---

## 3. IPsec VPN (Route-based, IKEv2)

### 3.1 HQ (701G): Phase1 + Phase2, Branch бүрд tunnel тусдаа

```
config vpn ipsec phase1-interface
    edit "HQ-BR1-W1"
        set interface "<HQ_WAN1_IF>"       # Tunnel-ийн гарах WAN
        set ike-version 2
        set peertype any
        set net-device disable             # SD-WAN member болгоход зөвлөдөг
        set proposal aes256-sha256
        set dhgrp 14                       # Хоёр талд ижил байх ёстой
        set remote-gw <BR1_WAN1_IP>
        set psksecret <PSK_PLACEHOLDER>    # Бодит PSK-г өөрөө үүсгэ (урт, санамсаргүй)
        set dpd on-idle
        set dpd-retrycount 3
        set dpd-retryinterval 5
        set keylife 86400
    next
end
config vpn ipsec phase2-interface
    edit "HQ-BR1-W1"
        set phase1name "HQ-BR1-W1"
        set proposal aes256-sha256
        set dhgrp 14
        set src-subnet 0.0.0.0 0.0.0.0     # Route-based: selector нээлттэй, чиглүүлэлтийг route шийднэ
        set dst-subnet 0.0.0.0 0.0.0.0
        set keylifeseconds 43200
    next
end
```

### 3.2 Branch (50G)

HQ-тэй ижил, зөвхөн дараах утгууд өөр:

| Параметр | Утга |
|---|---|
| `edit` нэр | `BR1-HQ-W1` |
| `set interface` | `<BR1_WAN1_IF>` |
| `set remote-gw` | `<HQ_WAN1_IP>` |
| `set psksecret` | HQ-тэй **яг ижил** |
| `proposal`, `dhgrp`, `keylife` | HQ-тэй **яг ижил** |

- `[Шаардлагатай үед]` WAN2 tunnel-д мөн адил хоёр дахь phase1/phase2 үүсгэ (`HQ-BR1-W2`: `<HQ_WAN2_IF>` ↔ `<BR1_WAN2_IP>`).
- `[Шаардлагатай үед]` Branch WAN IP динамик бол HQ-д `set type dynamic` ашиглана. Branch талд `remote-gw <HQ_WAN1_IP>`, HQ талд `set remote-gw` оронд `set type dynamic` тавина. Tunnel-ийг зөвхөн branch эхлүүлнэ.
- `[Шаардлагатай үед]` Tunnel IP тохируулах (HQ жишээ; Branch-д `ip`/`remote-ip`-ийг сольж тавина):

```
config system interface
    edit "HQ-BR1-W1"
        set ip <TUN1_HQ_IP> 255.255.255.255        # Tunnel-ийн local IP
        set remote-ip <TUN1_BR1_IP> 255.255.255.252 # Tunnel-ийн remote IP
    next
end
```

> [!warning] Хоёр талд заавал таарах зүйлс
> IKE version, proposal, DH group, PSK, Phase2 proposal/selector. Нэг нь зөрвөл tunnel босохгүй.
> Tunnel-ээр том packet алдагдвал firewall policy дотор `set tcp-mss-sender 1350` / `set tcp-mss-receiver 1350` ашиглаж MSS-ийг багасга (MTU/MSS-ийн утгыг ISP-ийн замаас хамаарч туршиж тохируул).

---

## 4. Routing

### 4.1 Branch

```
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set sdwan-zone "underlay"                  # Internet-ийн default route (SD-WAN zone-оор)
    next
    edit 10
        set dst <HQ_LAN_SUBNET>
        set sdwan-zone "overlay"                   # HQ LAN руу tunnel-ээр
    next
    edit 11
        set dst <HQ_LAN_SUBNET>
        set blackhole enable
        set distance 254                           # Tunnel унавал packet internet руу гарахаас сэргийлнэ
    next
end
```

### 4.2 HQ

```
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set sdwan-zone "underlay"                  # Internet default route
    next
    edit 10
        set dst <BR1_LAN_SUBNET>
        set sdwan-zone "overlay"                   # Branch LAN руу tunnel-ээр
    next
    edit 11
        set dst <BR1_LAN_SUBNET>
        set blackhole enable
        set distance 254                           # Tunnel унавал blackhole
    next
end
```

- `[Хувилбар]` `set sdwan-zone` нь 6.4+-д байна. Хуучин хувилбарт `set device "virtual-wan-link"`.
- `[Шаардлагатай үед]` Branch олон бол static route-ийн оронд BGP/OSPF over tunnel ашиглана (tunnel IP шаардлагатай).
- HQ-д одоо ажиллаж буй MikroTik-ийн ард байгаа subnet-үүд рүү route хэрэгтэй бол тусад нь static route нэм. MikroTik дээрх чиглүүлэлтийг **өөрчлөхгүй**.

---

## 5. Firewall Policy

### 5.1 Address object (HQ-д; Branch-д харгалзуулан сольж)

```
config firewall address
    edit "BR1_LAN"
        set subnet <BR1_LAN_SUBNET>      # Жишээ: 10.20.1.0 255.255.255.0
    next
    edit "HQ_LAN"
        set subnet <HQ_LAN_SUBNET>
    next
end
```

### 5.2 Policy (Branch дээрх жишээ; HQ-д `HQ_LAN`/`BR1_LAN`-ийг сэлгэнэ)

```
config firewall policy
    edit 1
        set name "LAN-to-Internet"
        set srcintf "<BR1_LAN_IF>"
        set dstintf "underlay"           # SD-WAN zone
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable                   # Internet-д гарахад NAT заавал
        set logtraffic all
    next
    edit 2
        set name "LAN-to-HQ-VPN"
        set srcintf "<BR1_LAN_IF>"
        set dstintf "overlay"            # IPsec tunnel zone
        set srcaddr "BR1_LAN"
        set dstaddr "HQ_LAN"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable                  # VPN-д NAT ХИЙХГҮЙ
        set logtraffic all
    next
    edit 3
        set name "VPN-to-LAN"
        set srcintf "overlay"
        set dstintf "<BR1_LAN_IF>"
        set srcaddr "HQ_LAN"
        set dstaddr "BR1_LAN"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
        set logtraffic all
    next
end
```

- Policy-д `service "ALL"`-ийг ажил дээр шаардлагатай service-үүдээр хязгаарла.
- Policy дарааллыг `show firewall policy`-аар шалга. Дээрээс доош таарсан эхний policy ажиллана.
- `[Шаардлагатай үед]` Branch-ээс HQ-ээр дамжуулан Internet гаргах бол Branch → overlay → HQ → underlay policy тусад нь хэрэгтэй.

---

## 6. Troubleshooting

### 6.1 VPN

| Команд | Зориулалт |
|---|---|
| `get vpn ipsec tunnel summary` | Tunnel-ийн төлөв, SA тоо |
| `diagnose vpn ike gateway list name <TUNNEL>` | IKE (Phase1) төлөв |
| `diagnose vpn tunnel list name <TUNNEL>` | Phase2 SA, packet encrypt/decrypt тоо |
| `diagnose vpn ike log filter dst-addr4 <PEER_IP>` | IKE debug-ийг peer-ээр шүүх `[Хувилбар]` (хуучин: `log-filter`) |
| `diagnose debug application ike -1` | IKE debug асаах |
| `diagnose debug enable` | Debug эхлүүлэх |
| `diagnose debug disable` + `diagnose debug reset` | **Debug-ийг заавал унтрааж reset хий** |

### 6.2 Routing

| Команд | Зориулалт |
|---|---|
| `get router info routing-table all` | Бүх route |
| `get router info routing-table details <IP>` | Тухайн IP руу ямар route ашиглагдахыг харах |
| `diagnose firewall proute list` | Policy route (SD-WAN rule-ууд) |
| `diagnose ip route list` | Kernel route table |

### 6.3 SD-WAN

| Команд | Зориулалт |
|---|---|
| `diagnose sys sdwan member` | Member төлөв |
| `diagnose sys sdwan health-check` | SLA: latency, jitter, loss, alive/dead |
| `diagnose sys sdwan service` | Rule бүрийн сонгосон member |
| `diagnose sys sdwan zone` | Zone-ийн member-үүд |

`[Хувилбар]` 6.2-оос өмнө `diagnose sys virtual-wan-link ...`.

### 6.4 Interface / Network

| Команд | Зориулалт |
|---|---|
| `get system interface physical` | Port-ын link, speed, IP |
| `diagnose hardware deviceinfo nic <PORT>` | NIC error/drop |
| `get system arp` | ARP table |
| `execute ping-options source <SRC_IP>` + `execute ping <DST_IP>` | Source IP заасан ping |
| `execute traceroute <DST_IP>` | Замын шалгалт |

### 6.5 Firewall / Packet

| Команд | Зориулалт |
|---|---|
| `diagnose sniffer packet any 'host <IP> and icmp' 4 0 l` | Packet барих (interface, чиглэл харагдана) |
| `diagnose debug flow filter addr <IP>` | Flow debug-ийн filter |
| `diagnose debug flow show function-name enable` | Ямар policy/route шийдэв гэдгийг харуулах |
| `diagnose debug flow trace start 20` + `diagnose debug enable` | 20 packet-ийн flow trace |
| `diagnose sys session filter dst <IP>` + `diagnose sys session list` | Session шалгах |

### 6.6 Түгээмэл шалтгаан

- Phase1 унахгүй мөртлөө traffic явахгүй: **route эсвэл policy байхгүй** (ихэнх тохиолдол).
- Phase1 босохгүй: PSK, proposal, DH group зөрсөн эсвэл UDP 500/4500 хаалттай (MikroTik/ISP).
- Tunnel тогтмол унах: DPD, NAT-T, ISP-ийн NAT timeout.
- Tunnel доторх том файл гацах: MTU/MSS (3-р хэсгийн анхааруулга).

---

## 7. Backup ба аюулгүй өөрчлөлт

### 7.1 Backup

| Команд | Зориулалт |
|---|---|
| `execute backup config tftp <FILENAME> <TFTP_SERVER_IP>` | Config-ийг TFTP рүү хадгалах |
| `execute backup config usb <FILENAME>` | USB рүү хадгалах |
| `show full-configuration` | Бүтэн config харах (terminal log-д хадгалж болно) |
| GUI: админ цэс → Configuration → Backup | Файл болгон татах |

Backup-ийг хийхдээ файлд `<HOSTNAME>_<YYYYMMDD>` гэх мэтээр нэр өг. Өөрчлөлт бүрийн өмнө шинэ backup хий.

### 7.2 Config revert mode (алдаа гарвал автомат буцаах)

```
config system global
    set cfg-save revert                # Config-ийг гараар хадгалах горим
    set cfg-revert-timeout 600         # 600 сек дотор хадгалахгүй бол өмнөх төлөв рүү буцна
end
```

- Өөрчлөлт хийсний дараа хэвийн байвал `execute cfg save`-ээр баталгаажуул. Ажиллаагүй бол timeout дуустал хүлээхэд буцна.
- `[Хувилбар]` `cfg-save revert` нь хувилбарт хамаарч ажиллагаа нь өөр байж болно. Production-д ашиглахаасаа өмнө lab дээр шалга.

### 7.3 MikroTik сүлжээг тасалдуулахаас сэргийлэх

> [!danger] Эрсдэлтэй үйлдлүүд
> - Одоо ашиглагдаж буй WAN/LAN interface-ийг SD-WAN member болгох, IP солих, `config system interface` засах.
> - HQ дээр `0.0.0.0/0` default route солих. MikroTik-ийн default gateway-аас өөр зам руу traffic гарч магадгүй.
> - MikroTik-тэй ижил subnet эсвэл ижил gateway IP FortiGate-д оноох (IP conflict).
> - Production LAN-д DHCP server асаах (MikroTik-ийн DHCP-тэй зөрчилдөнө).

Зөвлөмж:
1. Эхлээд **lab эсвэл тусдаа VLAN/subnet**-д FortiGate-ийг турш.
2. Өөрчлөлтийг **ажлын цагаас гадуур**, console/out-of-band хандалттайгаар хий.
3. Management access: өөрчлөлт хийхээсээ өмнө `config system interface` дээр mgmt interface-ийн `allowaccess`-ийг шалга (өөрийгөө түгжихгүйн тулд).
4. Нэг дор нэг Branch-ийг cut-over хий. Tunnel + SLA ажиллаж байгааг баталсны дараа дараагийнхыг хий.
5. Rollback төлөвлөгөө: backup файл, MikroTik-ийн хуучин config export (`/export file=<NAME>`), ISP-ийн холболтын мэдээлэл.
