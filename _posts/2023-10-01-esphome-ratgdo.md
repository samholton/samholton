---
layout: post
title: "ESPHome Version of RATGDO"
date: 2023-10-01 11:39
categories: Tinker
tags: [diy, local control, home assistant]

image:
  alt: ESPHome logo
  path: /assets/images/headers/esphome.webp
  lqip: data:image/webp;base64,UklGRmwAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSC4AAAARL6CgbRs2VRTt/omIwGdwoBAA0KYnUAYXYMcgf68QIvqfsPaa+Wp0+mXcElgDVlA4IBgAAAAwAQCdASoQAAgABUB8JaQAA3AA/vBpgAA=
---

A few weeks ago I setup Rage Against the Garage Door Opener (RATGDO) with the stock 2.1 firmware on my two garage doors - see info [here]({% post_url 2023-09-10-local-garage-door-control-ratgdo %}). One door has been working consistently, while one as been randomly showing as disconnected in HomeAssistant. They are both mounted side by side, so Wifi shouldn't be an issue. The problem device also allows me to connect to the limited web interface. I started to monitor the MQTT topics and started to debug the firmware. Yeah, at this point I should have tried a different board to eliminate hardware issues...


## Limitations of the Original Firmware

As I was debugging the firmware, I realized it did not expose the wireless remote lockout via the HomeAssistant auto-discovery. I found this [closed issue](https://github.com/PaulWieland/ratgdo/issues/22) where it was discussed, but was not added due to confusing terminology. The `lock` feature does not actually lock the door, but disables any wireless remotes. This is the same functionality (and name) on the wall controller itself. Its possible to add an MQTT action in HomeAssistant to send a message to the `lock` topic, but I think it should still be exposed.

I was prepping a PR to add a "Disable Wireless Remotes" toggle in auto discovery when I stumbled across a port of the firmware to ESPHome.


## New Firmware

[ESPHome-RATGDO](https://github.com/RATGDO/esphome-ratgdo) is a port of the original firmware but as an ESPHome module. Not only does this expose the lock functionality, but also includes a console log in the web interface. Note that this uses the ESPHome API server framework rather than MQTT - see some discussion on the [ESPHome docs](https://esphome.io/components/api.html#advantages-over-mqtt).

I decided to give this a try to see if it corrects my unavailable issues I've been seeing with the stock firmware.


## ESP Web Flasher

When attempting to connect to the ESP board with the web flasher, I kept receiving the following error

```console
Failed to execute 'open' on 'serialport': failed to open serial port.
```

Initially I thought maybe I was using a bad micro USB cord (everything is USB-C now). Or maybe it was a power only cord. After swapping cords a few times, I tried the same cord on my laptop. Everything worked fine. At this point I realized I had never connected on the machine I was using. Checking the permissions on the device, I found the issue

```console
ls -l /dev/ttyUSB0 
crw-rw---- 1 root dialout 188, 0 Oct  1 11:51 /dev/ttyUSB0
```

My user did not have permissions to the device.

```console
adduser sam dialout
```


## Retrieve the Rolling Code

If you are upgrading from the stock RATGDO firmware, be sure to capture the rolling code number so you can restore it after flashing the new ESPHome version. See the [docs](https://github.com/ratgdo/esphome-ratgdo#moving-from-stock-ratgdo) for more information.