---
layout: post
title: Actor discovery on local networks 
date: 2021-12-02 17:09:00
description: Local actor discovery in akka **(typed)** using UDP
tags: scala, akka
categories: Work
---

This post describes how to use Scala and Akka to implement simple actor discovery on local networks using UDP multicasts.
I aimed to ensure that the discovery process was both reliable and able to withstand errors. 
Additionally, I wanted to avoid the issue of ["split brain"](https://en.wikipedia.org/wiki/Split-brain_(computing)), 
in which the system becomes divided and unable to function properly.







