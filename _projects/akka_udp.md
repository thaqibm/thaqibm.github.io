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
# Networking prerequisites
We need some basic networking code to implement the main discovery system.

```scala
import akka.io.Inet.{AbstractSocketOptionV2, DatagramChannelCreator}
import java.net.{DatagramSocket, Inet4Address, InetAddress, NetworkInterface, StandardProtocolFamily}
import java.nio.channels.DatagramChannel
```


To begin, we need to obtain a IPv4 multicast network interface. 
This idea was inspired by the following code 
[snippet](https://github.com/akka/akka/blob/main/akka-docs/src/test/scala/docs/io/ScalaUdpMulticastSpec.scala#L31).
To do this, we can use the NetworkInterface class from the ``java.net package`` , which represents a 
Network Interface in the Java environment. 

Once we have a NetworkInterface object, we can use the getInetAddresses method to obtain a
Enumeration of ``InetAddress`` objects, 
which represent the IP addresses associated with the ``NetworkInterface``. 
We can then iterate through this Enumeration and filter out any IP addresses that do not support multicast, 
using the ``isMulticastAddress`` method of the ``InetAddress`` class.
```scala 
def ipv4iface = NetworkInterface.getNetworkInterfaces.asScala.find{
    x => x.supportsMulticast && x.isUp && x.getInetAddresses.asScala.exists(_.isInstanceOf[Inet4Address])
}
```

Using this ``NetworkInterface`` we can create a ``MulticastGroup`` class 
that will join a multicast group to begin receiving all datagrams sent to the group.


```scala
final case class InetProtocolFamily() extends DatagramChannelCreator {
    override def create(): DatagramChannel =
                           DatagramChannel.open(StandardProtocolFamily.INET)
}

final case class MulticastGroup(group: InetAddress) extends AbstractSocketOptionV2 {
    override def afterBind(s: DatagramSocket): Unit = {
        try {
            s.getChannel.join(group, .ipv4iface.get)
        }
        catch {
            case e: Throwable => e.printStackTrace()
        }
    }
}
```
  

