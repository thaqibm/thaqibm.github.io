---
layout: page
title: Actor discovery on local networks 
description: Local actor discovery in akka **(typed)** using UDP
img: assets/img/udp.png
tags: scala, akka
importance: 1
categories: Work
---

This post describes how to use Scala and Akka to implement simple actor discovery on local networks using UDP multicasts.
I aimed to ensure that the discovery process was both reliable and able to withstand errors. 
Additionally, I wanted to avoid the issue of ["split brain"](https://en.wikipedia.org/wiki/Split-brain_(computing)), 
in which the system becomes divided and unable to function properly.








