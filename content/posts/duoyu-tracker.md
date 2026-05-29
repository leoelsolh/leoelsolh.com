---
title: "A phishing kit operated live across 22 countries"
date: 2026-05-29T13:00:00+02:00
draft: false
slug: "duoyu-tracker"
tags: ["threat-intel", "phishing", "phaas"]
summary: "A friend in Sweden forwarded a domain he'd received in a text message. It turned out to be one of thirty active campaigns served from the same shared backend, targeting victims across twenty-two countries."
---


In late May, a friend in Sweden forwarded me a domain he'd received in a text message: transportstyrelsen-biljett.top. The SMS claimed he had an unpaid traffic fine and that he needed to settle it through the linked page, which presented itself as the Swedish Transport Agency. The page was a competent mobile-only imitation, one of thirty active campaigns served from the same shared backend, targeting victims across twenty-two countries.

Fetching the URL with curl returns what looks like a Cloudflare timeout error: "Error 524, a timeout occurred." Now here's what you wouldn't see from the browser: the HTTP status is 200 OK, the error page is what the server returns to non-mobile clients. so anything that's not a phone is served a fake Cloudflare error. Sending the same curl with a spoofed mobile 'Safari' User-Agent header gets us through to the actual kit, a Swedish Transportstyrelsen page asking the visitor to pay an unpaid traffic fine of 346 kr, about €32.

The kit pulls its content from an API at /getApp?app_id=N. The Swedish deployment is ID 27. Step through the other IDs and the picture gets bigger fast. Lloyds Bank UK, American Express in five countries, Vodafone Albania, Dutch traffic fines, DPD Austria parcel pickup. In total, 30 active deployments across 22 countries.

When a victim loads the page, the kit opens a WebSocket back to the same server. The important traffic flows from the server to the kit, telling it what to show next: card form, SMS code, email code, banking app prompt, PIN entry, balance check, or any custom screen the operator types up on the fly. This means what we're seeing is not a script firing on a timer. Someone is sitting at a panel, watching the session, deciding what to ask the victim for next.

The panel gives the operator a lot of control over the victim's browser. They can push the victim to any page in the kit, force a reload, or wipe the session and put them back at the start. They can reject a card based on its first six digits, which identify the issuing bank. And when the victim types into a custom field, the keystrokes stream back to the panel as they type.

Behind every deployment is the same setup. Querying any of the thirty app_ids at transportstyrelsen-biljett.top returns its config, so all thirty live on the same backend. They all pull brand logos from one CDN at storage.duoyu.pw, registered through NameSilo in October 2024. They all use the same master encryption key. The kit is designed so the public phishing domain in each config gets set to whatever host the API request came in on. That's how one backend can be fronted by many Cloudflare domains.

The operation is still adding deployments. Within twenty-four hours of starting to watch the API on 2026.05.28, three new ones appeared. Fine-payment impersonations for Finland, Norway, and Poland. The inventory at the time of writing sits at thirty active campaigns.

If you want to check your own logs against this operation, or block traffic to it, here's what we have:
```
transportstyrelsen-biljett.top   Swedish phishing domain, fronted by Cloudflare
storage.duoyu.pw                 Operator's shared asset CDN (NameSilo, registered 2024.10.03)
api.duoyu.pw                     Operator's API hostname (seen in CT logs)
```

The other 29 public phishing domains aren't in here. The kit is designed so each deployment is fronted by its own Cloudflare-aliased hostname, and the only one we saw in the wild was the Swedish one that started this. The rest weren't enumerable from inside the kit itself.

The polling and decryption tool used to pull the inventory is at [github.com/leoelsolh/duoyu-tracker](https://github.com/leoelsolh/duoyu-tracker).


SMS-delivered phishing operations like this one are not new, and this one isn't even the only one I've come across this month. What's worth pulling out is the live-operator part. A phishing page you tap from an SMS feels like a static thing, but every visitor's session here opens a WebSocket to a panel where someone can step in to ask for whatever code, PIN or app approval gets the operator past the next anti-fraud check. 

The advice for someone who's just received an SMS like this hasn't changed: don't tap the link. It's not rare to see initial recon where they track who clicked and who didn't. Go to the institution's site or app directly, and don't act on the urgency the message is crafted to create. If you've already tapped and entered any card details, call up your bank straight away and freeze the card. 

