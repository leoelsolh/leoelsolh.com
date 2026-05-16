---
title: "Chinese PhaaS Operation: DHL Sweden Smishing Investigation"
date: 2026-05-15T20:30:00+02:00
draft: False
description: "Looking into a Chinese Phishing-as-a-Service platform delivering DHL-branded smishing to Swedish targets, with a live operator WebSocket dashboard and a parallel Taiwan-targeted campaign on shared infrastructure."
tags: ["threat intelligence", "phishing", "PhaaS", "smishing", "DHL", "investigation"]
categories: ["threat intel"]
ShowToc: true
TocOpen: false
---

## Summary

A DHL-branded smishing campaign targeting Swedish users was traced to a Chinese-operated Phishing-as-a-Service platform hosted on Alibaba Cloud Hong Kong. Looking into the infrastructure revealed a multi layer architecture with bot filtering, keystroke level card exfiltration via a Vue.js SPA skimmer, real time operator review over WebSocket, and live BIN intelligence. A separate health scam targeting Taiwan elderly was identified running on the same server, which means were looking at a multi-tenant commercial operation.

At time of publication the infrastructure remains partially active.

## Initial Vector

The campaign delivery is SMS impersonating DHL Sweden:

> Your DHL package requires a customs fee. Pay here: https://tecyuse[.]top/...

The localisation is unusually accurate. Landing pages render in Swedish, address fields match Swedish postal code formats, and the phone number country is targeted. Compared to mass blasted smishing, targeting is deliberate.

The destination domain was registered the same day as campaign launch.

## First-Stage Filter

The first domain (`tecyuse[.]top`) serves an almost perfect clone of a Cloudflare bot detection captcha. The page does not communicate with Cloudflare. JavaScript on the page calls a local `/eat` endpoint, which returns a session token which contains an embedded timestamp:

```
valid_<YYYYMMDDHHMMSS>_<md5_hash>
```

The timestamp is in CST (UTC+8)... the first signal our operator was Chinese based. The captcha functions as a researcher filter. Automated scanners such as URLScan are redirected to the legitimate DHL site. Real users from in-region residential IPs are passed to the next page.

## Infrastructure

All malicious domains resolve to a single Alibaba Cloud server in Hong Kong. Shodan historical data shows the server has been active since at least 2026-02-27, showing us that domain rotation is the actor's primary evasion strategy. 

```
IP:      47.82.74.37
ASN:     AS45102 (ALIBABA-CN-NET)
GeoIP:   Hong Kong (HK)
Server:  GoFrame HTTP Server (Chinese Go web framework)
Proxy:   Caddy 1.1
OS:      Ubuntu (OpenSSH 9.6p1)
```

Port 111 (portmapper/RPC) was exposed, consistent with a standard unmanaged VPS rather than a hardened server. Operational security is concentrated at the application layer, not the OS layer.

## Domain Map

| Domain | Role | Registered |
|---|---|---|
| `tecyuse.top` | SMS redirector and captcha filter | 2026-03-23 09:17 UTC |
| `fvuyce.top` | SMS redirector, same campaign | 2026-03-23 09:19 UTC |
| `yciaycb.top` | Full PhaaS kit, card exfiltration | Unknown |
| `dopckshin.top` | Separate Taiwan health scam campaign | 2025-12-16 |

`tecyuse.top` and `fvuyce.top` were registered two minutes apart on the day of discovery. Let's Encrypt SSL certificates were auto-provisioned within the hour. Both domains use Alibaba DNS (`ns1.alidns.com`, `ns2.alidns.com`). Domain deployment is fully automated.

`dopckshin.top` uses a TrustAsia SSL certificate, a Chinese CertAuthority, and serves Traditional Chinese content targeting Taiwanese users aged 45 and over.

## The Skimmer

The card collection domain (`yciaycb[.]top`) loads a Vue.js 3 single page application from `/svweee/`. The cloned DHL Sweden page proxies legit DHL assets like their logo, fonts, and header, all live from `www.dhl.com/se-sv`. Its only the malicious components that are hosted locally.

The kit is versioned and maintained. A build comment is even visable in the page source:

```html
<!-- Date Compiled - - Mon Mar 23 11:29:35 UTC 2026 | Mask - - 32 | Application Version - - 2026.2.0 -->
```

The exfiltration method is unusual. Rather than waiting for form submission, the skimmer POSTs every keystroke individually to a collection API as the victim types

```http
POST /sLxurrwZio/api/input?token=<operator_token>
Content-Type: application/json

{"content":{"type":"input_card","key":"cardNumber","text":"2889"},"timestamp":<ts>}
{"content":{"type":"input_card","key":"cardNumber","text":"2889 XXX"},"timestamp":<ts>}
{"content":{"type":"input_card","key":"cardNumber","text":"2889 XXXX XXXX XXXX"},"timestamp":<ts>}
{"content":{"type":"input_card","key":"expires","text":"06/27"},"timestamp":<ts>}
{"content":{"type":"input_card","key":"cvv","text":"998"},"timestamp":<ts>}
```

The server responds `{"code":0,"message":"","data":{}}` to every keystroke. Partial card data is captured before the victim submits anything. A grandma who recognises the scam mid entry has already lost the digits she typed before that point.

## Live Operator Dashboard

The most significant finding of this investigation. The skimmer maintains a persistent WebSocket connection while the victim is on the page:

```
wss://yciaycb.top/ws?token={operator_token}
```

Connecting with the session token streams all collected data to the operator in real time:

```json
{
  "event": "result_type",
  "content": {
    "type": "reject",
    "value": {
      "data": {
        "addressPageData": {
          "Your Name": "Johan Doö",
          "Address": "Drottningsgatan 11",
          "City": "Stockholm",
          "Zip Code": "1121"
        },
        "cardData": {
          "cardBIN": {
            "bank": "NORD BANK, LTD.",
            "country": "SWE",
            "schema": "MASTERCARD",
            "type": "DEBIT",
            "level": "PLATINUM"
          },
          "cardNumber": "2889 5060 XXXX 6092",
          "expires": "05/29",
          "cvv": "555"
        },
        "cardHistory": [...]
      }
    }
  }
}
```

The `result_type` field is controlled by the operator. During testing, the session was initially set to `reject`, then changed to `""`. The change occurred while the session was idle, indicating a human operator reviewed the submission and modified the victim flow in real time.

The WebSocket sends `heartbeat` events every five seconds and `pong` responses, keeping the operator console alive. The `cardHistory` array tracks all card attempts per session, indicating operators expect victims to attempt a second card after a fake decline.

This is human operated fraud with decision loops, not scripted exfiltration.

## PhaaS Platform Architecture

The initial API call on page load returns the operator's campaign configuration:

```json
{
  "Token": "{operator_token}",
  "custom": {
    "deny_c_msg": "",
    "pay_amount": "",
    "address_msg": "",
    "customOp": []
  },
  "isBlock": false,
  "country": "SE",
  "mode": 1
}
```

Each operator holds a token. Each token has a tenant scoped configuration: country targeting, custom victim facing messages, IP block lists, mode flags. Campaigns are spun up per country from a shared backend.

Live BIN lookups are performed on every card submission, returning bank, country, schema, type, and level. This allows operators to prioritise high-value cards (PLATINUM and CREDIT over standard DEBIT) and decide per victim whether to accept, reject, or request additional verification.

## Parallel Campaign on Shared Infrastructure

Pivoting on the IP (`47.82.74.37`) identified a second domain on the same server: `dopckshin[.]top`. The domain uses a TrustAsia SSL certificate and serves Traditional Chinese content targeting Taiwanese users aged 45+ with a fake health products scam. Google Ads tracking ID `AW-17991545097` is embedded in the page, indicating paid traffic acquisition.

Same infrastructure, different campaign, different language, different target demographic. The server hosts at least two independent phishing operations in parallel. The platform is the constant. Campaigns swap in and out.

## Attribution

| Indicator | Assessment |
|---|---|
| GoFrame HTTP Server | Chinese Go web framework, dominant in mainland China |
| Alibaba Cloud (Aliyun) | Preferred infrastructure for Chinese threat actors |
| `ns1.alidns.com` / `ns2.alidns.com` | Alibaba DNS |
| Token timestamp timezone | CST (UTC+8) confirmed |
| TrustAsia SSL CA | Chinese certificate authority |
| `dopckshin.top` content | Traditional Chinese, Taiwan targeting |
| Registrant Country (secondary domain) | Hong Kong |
| Versioned application build comment | Maintained product, not one off |

No single indicator is conclusive. Together they describe a Chinese operator running a commercial PhaaS platform out of Alibaba Cloud Hong Kong with multi-tenant operator access. The infrastructure sophistication and versioned build pipeline are consistent with an organised group rather than a lone actor. The platform is likely rented or sold to multiple operators running independent campaigns.

## Disclosure

The infrastructure was active during write up. Reports submitted to:

- `phishing-dpdhl@dhl.com` (DHL Fraud Team)
- `abuse@alibaba-inc.com` (Alibaba Cloud Abuse)
- `abuse-contact@publicdomainregistry.com` (PDR Ltd, domain registrar)
- `cert.se` (Swedish CERT)
- Google Safe Browsing
- ThreatFox (abuse.ch)
- URLScan.io (public submission, verdict marked malicious)

## Indicators of Compromise

```
# IP Addresses
47.82.74.37

# Domains
tecyuse.top
fvuyce.top
yciaycb.top
dopckshin.top
xx.dopckshin.top

# ASN
AS45102 (ALIBABA-CN-NET)

# SSH Key Fingerprint
[REDACTED]
# Page Hash (tecyuse.top/post)
f19133a3407d991bc68bfa9f72fa10886299c9cf4e4d72035a0782e6ec04ea23

# SSL Certificate Serials
067dea91dda4ce324030555c2e4d7360625b  (fvuyce.top)
060a65556223771fac3b38cf4bf99ba45561  (tecyuse.top)
245b5e325f55dfbd08bad4d195195e1898c108b3  (dopckshin.top)

# API Endpoints
/sLxurrwZio/api
/sLxurrwZio/api/input
/ws

# Kit Paths
/svweee/
/assets/index-1a74d72d.js
/assets/CardView-e269a034.js
/assets/AddressView-95dda592.js
/assets/HomeView-735b3a01.js

# Google Ads Tracking ID (dopckshin campaign)
AW-17991545097

# Server Stack Fingerprint
GoFrame HTTP Server + Caddy 1.1 + Ubuntu + Aliyun
```

## Notes

Three observations i think are worth bringing up again.

The gap between application and OS layers is sharp. Application layer hardening is professional, with code splitting, randomised paths, custom WebSocket protocol, and a multi-tenant operator console. OS layer is a stock VPS with port 111 exposed. The operators concentrate on evasion at the layer victims interact with, not the layer scanners enumerate.

Time from credential capture, to operator decision is measured in seconds instead of hours. Fraud workflows relying on a delay between exfiltration and use will not catch this kit.

The PhaaS commercial model means takedowns of individual operators do not kill the platform. The Sweden DHL scam and the Taiwan health products scam share a host. They do not share a tenant. Removing one tenant does not affect the others. Platform level disruption is what matters.