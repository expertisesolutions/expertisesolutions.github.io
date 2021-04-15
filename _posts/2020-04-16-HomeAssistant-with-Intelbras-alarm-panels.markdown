---
layout: post
title:  "HomeAssistant with Intelbras alarm panels"
date:   2021-04-16 11:20:47 -0300
categories: embedded homeassistant automation intelbras alarm
image: "/assets/img/2021-04-16-HomeAssistant-entry.png"
share-img: "/assets/img/2021-04-16-HomeAssistant-shareimg.png"
author: Felipe Magno de Almeida
---

Intelbrás manufactures very popular alarm panels in Brazil market and
some of them have network protocols that communicates with monitoring
services through GPRS and Ethernet. Can we leverage this communication
to use them in Home Automation systems?

# Now we can

At Expertise's github [homeassistant
fork](https://github.com/expertisesolutions/core/tree/amt-alarms) you
can check the `amt_alarms` component to connect your AMT2018 or AMT4010
alarm panels.

You can either copy the `homeassistant/components/amt_alarm` to
custom_components or you can checkout the actual branch and use it.

# What is supported

The integration polls the alarm panel for motion sensor changes. This
means that you can use the motion sensors from the alarm panel to your
automations, the motion sensors are automatically added as entities to
the HomeAssistant.

Additionally, the partitions are also added as separate alarm panels, allowing
the user to arm, disarm and have triggers come from specific partitions.

# Implementation

The protocol implementation was and still is completely reverse
engineered. Absolutely no documentation is available about the
protocol and monitoring services use proprietary applications to read
the information transmitted by the alarm panels.

The reverse engineering was done by inspecting TCP packets between the
Android application and the alarm panel, extracting information and
calculating correct checksum.

# Virtual sensors

If you want to have switches on HomeAssistant that can actually
trigger as sensors on the alarm panel, we can't do this through the
network protocol. Fortunately though, the Alarm Panel has a RS485 bus
which we also reverse engineered and created a
[MQTT-to-XEZ-emulator](https://github.com/expertisesolutions/xez-4008-emulator-mqtt)
that can be used through a RS485 serial adapter.

The RS485 communication protocol mentions the use of Modbus in the
device's documentation, however we've noticed that even though it does
use timing protocol just as Modbus does, the actual packets are
completely proprietary. We were able to reverse engineer this through
trial and error.

The XEZ-4008 device can add more sensors through the RS485 bus, which
is what the emulator pretends to do, and listens to MQTT packets in
`"amt/{0-47}"` topics and looks for on and off payloads. If it is on,
then it communicates as triggered to the alarm panel.

This allows to create smart sensors by acquiring information from
other entities in the HomeAssistant. For example: You can use motion
detection from multiple cameras to trigger one or more sensors in the
alarm panel, but you can use conditions such as if it is after a
certain time or if nobody is home and other infinite possibilities.

# Dashboard example

This is my home dashboard as an example:

![Dashboard](/assets/img/2021-04-16-HomeAssistant.png "Dashboard")

# Conclusion

HomeAssistant is a very powerful project, and adding more integrations
to it is far from difficult if you know how to talk to your hardware
and is super useful and fun!

