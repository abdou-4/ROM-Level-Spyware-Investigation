# 📡 Infinix X6532  ROM-Level Spyware Investigation

> **A passive network traffic analysis revealing a factory-installed browser hijack and persistent tracking infrastructure baked into XOS 14.0.1 all happen with "user experience improvement" OFF**

---

## ⚠️ TL;DR

| Finding | Severity | Fixable Without Root? |
|---|---|---|
| Chrome homepage hijacked to `page.portals.mobi` → leaks device hardware ID over plain HTTP | 🔴 Critical | ✅ Yes |
| Base `android` process phones home to `gslb.shalltry.com` at every boot | 🔴 Critical | ⚠️ Partial (DNS block) |
| TPMS (`com.hoffnung`) beacons to `ire-dsu.shalltry.com` (Alibaba Cloud) | 🔴 High | ⚠️ Partial |
| Dynamic Bar builds a "One-ID" cross-device profile via `ire-oneid.shalltry.com` | 🟠 Medium | ⚠️ Partial |
| 10 plain HTTP connections (no TLS) including device-ID transmission | 🟠 Medium | ⚠️ Partial |

---

## 📋 Device Information

| Field | Value |
|---|---|
| **Model** | Infinix X6532 |
| **OS** | XOS 14.0.1 (Android 14) |
| **Manufacturer** | Transsion Holdings |
| **Capture tool** | PCAPdroid |
| **Capture date** | 2026-05-28 |
| **Capture duration** | ~16 minutes (17:10 → 17:26 UTC+1) |
| **Total connections logged** | 251 |

---

## 🔬 Methodology

Traffic was captured using **PCAPdroid** (no root required) running in VPN-mode on the device itself, which intercepts all outgoing connections at the application layer. No external hardware was used.

```
[Android App] → [PCAPdroid VPN Interface] → [Real Internet]
                        ↓
                  Captured PCAP/CSV
                  Analyzed offline
```

The capture was done on a **fresh boot + idle session + one manual Chrome open** to trigger the hijack.

---

## 🔍 Finding 1 — Chrome Browser Hijack (ROM-level)

### What happens

Every time Chrome is opened, the factory-configured new-tab/homepage silently fires a request to `page.portals.mobi` **before** loading Google. The redirect completes in ~1 seconds — the "blink" users notice.

### The URL

```
http://page.portals.mobi/sp/m/Infinix%20X6532
  ?p=X6532-OP        ← Model + operator code
  &uid=8188[...] ← Persistent hardware device ID ⚠️
  &gaid=00000000-[...] ← Google Ad ID (zeroed out = tracking disabled)
```

### Traffic timeline (from PCAP, 17:25:33)

| Time (UTC+1) | Event | Bytes |
|---|---|---|
| `17:25:33.082` | Chrome DNS query #1 → `page.portals.mobi` | 63 sent / 211 rcvd |
| `17:25:33.088` | Chrome DNS query #2 (redundant) | 63 sent / 165 rcvd |
| `17:25:33.139` | First HTTP GET → `13.207.64.198:80` | 100 sent / 88 rcvd |
| `17:25:33.307–.373` | 3× redirect probes (100 bytes each) | — |
| `17:25:33.442` | Full HTTP exchange (device ID transmitted) | 767 sent / 684 rcvd |

### Why it's dangerous

- Transmitted over **plain HTTP (port 80)** — zero encryption
- Anyone on the same Wi-Fi (café, hotel, university) can intercept the `uid=` parameter
- The UID `8188[...]` is a **persistent hardware identifier** — it doesn't reset with a factory reset
- Host resolves to **Microsoft Azure** (`13.207.64.198`) — Transsion pays to run this infrastructure commercially

### Root cause

Transsion/Infinix pre-programs Chrome's homepage or new-tab URL in the factory ROM. This is **not a virus or a user action** — it ships this way.

---

## 🔍 Finding 2 — Transsion `shalltry.com` Tracking Network

Three system processes call back to Transsion's own identity/ad backend from the moment the device boots. You cannot stop them without disabling or uninstalling the apps using USB debugger (they don't exist on app list)  or blocking at DNS level.

### Connection table

| Process | Package | Domain | Resolved IP | Cloud Provider |
|---|---|---|---|---|
| Android (base OS) | `android` | `gslb.shalltry.com` | `154.85.94.35` | Unknown |
| TPMS | `com.hoffnung` | `ire-dsu.shalltry.com` | `47.254.156.191` | Alibaba Cloud |
| Dynamic Bar | `com.transsion.dynamicbar` | `ire-oneid.shalltry.com` | `3.160.188.22` | AWS (Ireland) |

### What `ire-oneid` means

`ire` = Ireland (EU data center), `oneid` = One-ID program. This is Transsion's attempt to build a **persistent cross-device identity** tied to hardware — similar to Apple's IDFA or Google's GAID but entirely proprietary and non-resettable.

### Data volumes (16-minute capture)

- `shalltry.com` total: **6,129 bytes sent / 11,480 bytes received**
- 6 connections across 3 subdomains
- All connections begin **within 1 second of capture start** (i.e., at boot)

---

## 📊 Traffic Breakdown

```
Total connections : 251
├── Chrome         : 69   (Google services, hijack traffic)
├── Firefox        : 67   (Mozilla + uBlock updates — clean)
├── Instagram      : 42   (Meta CDN/QUIC — expected)
├── Google Play    : 22   (update checks — expected)
├── Android (OS)   : 14   ⚠️ includes shalltry.com
├── Play Services  : 13   (FCM push — expected)
├── WhatsApp       :  6   (expected)
├── TPMS           :  2   ⚠️ shalltry.com
├── Dynamic Bar    :  2   ⚠️ shalltry.com
└── Other          : 14

Protocol split:
DNS   : 118  (47%)
HTTPS :  59  (24%)
QUIC  :  30  (12%)
HTTP  :  10  (4%) ← unencrypted ⚠️
TLS   :  12  (5%)
Other :  22  (9%)
```

---

## 🛡️ Mitigations

### Fix 1 — Stop the Chrome Hijack (Easy, ~2 minutes)

```
Chrome → ⋮ → Settings → Homepage
  → Toggle OFF or set to: https://www.google.com

Chrome → ⋮ → Settings → New tab page
  → Set to: Default Google or Blank
```

Alternatively, use **Firefox** as your default browser — it has no such hijack in this ROM.

### Fix 2 — Block shalltry.com at DNS Level (Recommended)

Set your device or router DNS to a filtering resolver:

**Option A — AdGuard DNS (free)**
```
DNS-over-HTTPS: https://dns.adguard-dns.com/dns-query
IPv4: 94.140.14.14 / 94.140.15.15
```

**Option B — NextDNS (recommended for visibility)**
```
1. Create a free account at nextdns.io
2. Add custom block rule: *.shalltry.com
3. Apply profile DNS to your Wi-Fi
```

On Android: `Settings → Wi-Fi → [Your network] → Advanced → Private DNS`
Enter: `dns.adguard-dns.com` (or your NextDNS hostname)

### Fix 3 — Disable TPMS and Dynamic Bar

```
needs to enable devloper option → usb debbugger → disable it using Universal Android Debloater
```

> ⚠️ These will likely be re-enabled after an OTA update. Check after every system update.

### Fix 4 — Nuclear Option (Advanced)

Flash a clean AOSP-based ROM. LineageOS support for X6532 is **not officially available** as of this writing — check [LineageOS Devices](https://wiki.lineageos.org/devices/) or XDA Developers for community builds.

---

## 🔗 References & Related Work

- [Transsion Holdings — Wikipedia](https://en.wikipedia.org/wiki/Transsion)
- [shalltry.com WHOIS](https://whois.domaintools.com/shalltry.com)
- [PCAPdroid — Open Source Android Traffic Capture](https://github.com/emanuele-f/PCAPdroid)
- [Prior art: Transsion malware reports (2019, BuzzFeed News)](https://www.buzzfeednews.com/article/nicolenguyen/tecno-phones-preinstalled-malware-transsion)
- [XDA Developers — Infinix X6532](https://xdaforum.com)

---

## 📌 Disclosure Notes

- No proprietary keys, tokens, or user credentials are included in this repository.
- The `uid=` parameter is a hardware device identifier belonging to the researcher's own device.
- This research was conducted on a personally-owned device for educational and portfolio purposes.
- Transsion Holdings was not contacted prior to publication (behavior is ROM-level by design, not a vulnerability).

---

## 👤 Author

Security research conducted as part of SOC analyst portfolio development.  
Tools used: PCAPdroid · Wireshark-compatible PCAP analysis
