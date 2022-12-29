---
layout: page
title: Akka actor discovery
description: Local actor discovery in akka <b>typed</b> using UDP multicasts
img: assets/img/udp.png
importance: 1
category: Work
---

This post describes how to use Scala and Akka to implement simple actor discovery on local networks using UDP multicasts.
I aimed to ensure that the discovery process was both reliable and able to withstand errors. 
Additionally, I wanted to avoid the issue of ["split brain"](https://en.wikipedia.org/wiki/Split-brain_(computing)), 
in which the system becomes divided and unable to function properly. 

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="assets/img/udp.png" title="udp multicast" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    UDP Multicast
</div>

{% raw %}
```scala
def add1(i: Int) = i + 1
```
{% endraw %}
