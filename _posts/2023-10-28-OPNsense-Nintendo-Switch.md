---
layout: post
title: "OPNSense and Nintendo Switch NAT Type"
date: 2023-10-28 22:42
categories: OPNSense
tags: [opnsense, nintendo switch, nat, networking]

image:
  alt: Nintendo Switch
  path: /assets/images/headers/nintendo-switch.webp
  lqip: data:image/webp;base64,UklGRqIAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSCsAAAARJ6CQbQQ4dRgn8+JcRATEHQoBoGGCCSGBtvlTXYSI/k8ARlfL7PTnjhgBAFZQOCBQAAAAEAIAnQEqEAAIAAVAfCWwAnR/ABkeQwxtAAD+z3XLJNMMBFn7roEZeeY/AOybh25QDlZgFoF93cv1tktwugfDIEZv8DY7BRM5AbXvQ4CyAAA=
---

My son was recently trying to play multiplayer Mario Kart 8 on Nintendo Switch with his friends. However, he was unable to join his friend's room and his friend was unable to join his room. Both rooms were visible, but some generic error about not being able to connect kept popping up. After scrolling through the Nintendo network settings, I found a test option. His Nintendo was connected fine but reported a `NAT Type` of `D`.

Several YouTube videos and other guides recommended port forwarding a wide range of ports to the Nintendo Switch. However, moving to a `NAT Type` of `B` was good enough to allow online multiplayer.

The following steps and screenshots were performed on OPNSense 23.7.2.


## Configure static IP address for the Nintendo

1. Navigate to Services > DHCPv4 (I'm using v4 internally) > [network name]

1. Scroll to the bottom and click the [+] button under **DHCP Static Mappings for this interface**

1. Populate the MAC address and desired IP address along with any other desired fields


## Create firewall alias for Nintendo

1. Navigate to Firewall > Aliases

1. Click the [+] button at the bottom of the screen

1. Enter a name, select `Host(s)` for type, and enter the static IP created above in "Content"


## Add outbound NAT rule for Nintendo

1. Navigate to Firewall > NAT > Outbound

1. Ensure "Hybrid outbound NAT rule generation" is selected

1. Click the [+] button under "Manual rules"

1. Save the rule with the following settings:

    * Interface: `WAN`
    * Protocol: `UDP`
    * Source address: `<nintendo switch alias created above>`
    * Source port: `any`
    * Destination address: `any`
    * Destination port: `any`
    * Translation / target: `WAN address`
    * Static port: checked

1. Run the Nintendo Switch connection test again to confirm NAT type `B`