---
layout: post
title: Mapping out connector J2
tags: 
  - Nextion
categories:
  - Nextion
---

On each TJC/Nextion basic panel is a set of 5 pads labeled J2.  Nextion has always been cagey about these pads and tend to delete any posts mentioning them (it's something of a theme with them).  Here's what they look like:

![2019-01-30-Mapping-out-connector-J2-001.jpg](https://github.com/aderusha/aderusha.github.io/blob/master/images/2019-01-30-Mapping-out-connector-J2-001.jpg?raw=true)

For reference, here's the STM32

![2019-01-30-Mapping-out-connector-J2-002.png](https://github.com/aderusha/aderusha.github.io/blob/master/images/2019-01-30-Mapping-out-connector-J2-002.png?raw=true)

These pads are used to flash the STM32 processor via [Serial Wire Debug](https://www.silabs.com/documents/public/application-notes/an0062.pdf)

Tracing the paths, here's what we can see:

| 1   | 2 | 3     | 4     | 5    |
|-----|---|-------|-------|------|
| GND | RESET  | SWCLK | SWDIO | 3.3V |