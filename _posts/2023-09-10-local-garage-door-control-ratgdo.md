---
layout: post
title: "Local Garage Door Control with RATGDO"
date: 2023-09-10 9:54
categories: Tinker
tags: [diy, local control, home assistant]

image:
  alt: ratgdo mounted with zip ties
  path: /assets/images/headers/ratgdo-mounted.webp
  lqip: data:image/webp;base64,UklGRkwAAABXRUJQVlA4IEAAAADwAQCdASoQAAgABUB8JYwAAuzXocUFC6AA9Ggl7AgUq8pFUQq4nyAAfewRDEsBvKGAtyJHIjuHuydBHvywAAAA
---

In my previous house, I used a [custom board and NodeMCU]({% post_url 2023-09-10-local-garage-door-control-nodemcu %}) to integrate my garage door opener. However, the new garage door opener is Security+ 2.0 and used an encrypted communication protocol.

## The Plan

Buy some cheap Security+ 2.0 openers from Amazon and interface directly with their button contacts. Rather than using a NodeMCU like last time, I was going to use a [Shelly](https://www.shelly.com) flashed with [Tasmota](https://tasmota.github.io/docs/Download/) firmware. This compact device can accept input from the reed switch and trigger a button press with the relay. I actually had tested all of this out on my workbench and was just waiting to install in the garage. But them I stumbled across [Rage Against the Garage Door Opener](https://github.com/PaulWieland/ratgdo).


## The Better Plan

Rage Against the Garage Door Opener is a project that can interface with the Security+ 2.0 openers. They cleverly take advantage of the protocol that allows new wall control boards to integrate with an existing system.

When I first found [RATGDO](https://github.com/PaulWieland/ratgdo), they were at their initial version. This included soldering contacts directly to the logic board inside the opener. It also had some limitations on which versions of the opener logic board were supported. A few short months later and they released 2.0. Not only does this expand the supported openers, but there is no more wiring to the opener logic board. You simply wire their board to the existing control terminals on the opener.

![wiring connecting to garage door opener](/assets/images/posts/ratgdo-opener-wiring.webp){: width="400" height="400" lqip="data:image/webp;base64,UklGRlIAAABXRUJQVlA4IEYAAADwAQCdASoMABAABUB8JZwAAlr1j6NiRoAA/uc5k2Rn9j31qwheI3xZaJ/7Bt5tb00P1MeQy1pBEI4MlJoqUGsFH1BOccAA"}
_Combined with existing wires on the opener terminals_

The installation was very smooth. I ran some small gauge wire from one opener over to my second. This way, I could share a single USB plug to power both boards. 

![RATGDO mounted and wired](/assets/images/posts/ratgdo-wiring.webp){: width="400" height="400" lqip="data:image/webp;base64,UklGRkwAAABXRUJQVlA4IEAAAADQAQCdASoQAAwABUB8JZQAAlcS1dSZgAD30Y2+Sjjnmyq8e6RQ93SJ8RyHh5JyQ8Bix2g9r6k2xa5mFF7rAAAA"}
_Mounted with zip ties to the opener support_

I purchased the shield from their [online store](https://checkout.square.site/buy/W5CKGBWB4KFD645XMRNMSEDY) and used their online flasher to prepare my boards. After flashing, you can use the web UI to configure the board, including MQTT. The firmware registers with Home Assistants MQTT discovery so you can quickly add it to dashboards. Overall, the process was very simple and their documentation is great. I would definitely take a look if you want local control of any Security+ 2.0 opener.