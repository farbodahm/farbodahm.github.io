---
author: "Farbod Ahmadian"
title: "Devlog August"
date: "2026-08-19"
description: "A short devlog: getting hands-on with LoRa, jailbreaking my Kindle for KOReader, and publishing a Kubernetes workshop."
tags:
- devlog
- lora
---

I found that it's been a while since my last blog, so I decided to write this short devlog on what I'm busy nowadays with.

## LoRa

I had zero knowledge about LoRa. I knew the name, I knew it had something to do with long range and low power radio, and that was the end of it. What I wanted to find out was pretty simple: what LoRa actually is, how you use it, how two devices talk to each other over it, and whether I could write my own communication protocol on top of it.

The annoying part about starting from zero with hardware is the material. It's either a five minute video that blinks an LED, or an RF engineering textbook. I wanted to be hands-on fast.

So I wrote down the subjects I wanted to learn, in the order I wanted to learn them, and asked Claude to turn that outline into a course. The syllabus and the topics are mine, the written content is generated. It's here if it's useful to anyone: [lora-from-zero](https://github.com/farbodahm/lora-from-zero).

The hardware is two Heltec WiFi LoRa 32 V3 boards (an ESP32-S3 with an SX1262 radio, 868 MHz for the EU), a LiPo battery, antennas, and a cheap multimeter. Two boards rather than one, because you need a second node to measure anything real.

![My LoRa setup: two Heltec WiFi LoRa 32 V3 boards](/images/lora-1.png)

Right now I'm at the Meshtastic stage. Meshtastic is open source firmware that turns these boards into an off-grid mesh network. Nodes relay each other's messages, and your phone talks to a node over Bluetooth. No internet, no SIM, no infrastructure.

After that the plan is to drop Meshtastic entirely and drive the SX1262 directly, design my own wire format, and see how far I can push distributed systems ideas over a LoRA channel.
That constraint is what makes it interesting to me. Gossip and anti-entropy look very different when a single 40 byte message costs you half a second of airtime.

## I made my Kindle more practical

Thanks to a suggestion from [Farid](https://www.reddit.com/r/rss/comments/1v3awmi/comment/oz2d0ya/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button), I jailbroke my Kindle Paperwhite and installed [KOReader](https://koreader.rocks/) and [wallabag](https://wallabag.it/).

So now I can send a blog post or an article from my laptop and read it on the Kindle. Which is what I wanted the device to do in the first place.

Funny part: I had to keep the Kindle in airplane mode for a few weeks so it wouldn't pull a firmware update. From what I can tell, the main job of those updates is to close the jailbreak.

## A hands-on Kubernetes workshop

I gave a Kubernetes workshop and published the material afterwards: [hands-on-k8s-workshop](https://github.com/farbodahm/hands-on-k8s-workshop).

It's a fast hands-on introduction using k3s, covering application deployment, databases, namespaces, and the core concepts. The goal is to get you using Kubernetes as quickly as possible without working through a long, theory-heavy course first.
I made it public after the feedback from the participants was good.

Looking at all three of these together, my theme for this month seems to be learning things by doing them as fast as possible.
Maybe bacause of [attention](https://glyphack.com/attention/) we are all experiencing nowadays?
