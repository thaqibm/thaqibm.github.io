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
  
# Singleton Server 
We need to design a server that broadcasts its address using UDP multicast. 
This server will be implemented as an Akka cluster singleton actor. 
A singleton actor is a good choice for this server because it serves as a central authority for 
cluster-wide consistent decisions, such as managing the process of clients joining the cluster.
[More on Akka Cluster Singleton](https://doc.akka.io/docs/akka/current/typed/cluster-singleton.html)
```scala 
import akka.actor.typed._
import akka.actor.typed.scaladsl.Behaviors
import akka.actor.typed.scaladsl.adapter._
import akka.cluster.typed.Cluster
import akka.event.Logging
import akka.io.{IO, Udp}
import akka.util.ByteString
import akka.{actor => classic} // alias for untyped Actor
import java.net.InetSocketAddress
import scala.concurrent.duration.DurationInt
```
In order to interact with the Akka UDP module, we will need to use untyped actors. 
This is because there is no typed interface available for interacting with Akka UDP.
We can use the [Actor Sink Pattern](https://doc.akka.io/docs/akka/current/stream/actor-interop.html)
to interop with the classic actor. 
```scala 
private final class ServerActor(
        localInetAddr: InetSocketAddress,
        remoteAddr: InetSocketAddress,
        sink: ActorRef[String]) // Actor sink to forward messages to
    extends classic.Actor with classic.Timers {

        val TIMER_KEY = "DISCOVERY"
        val delay = 5.seconds
        import context._
        val log = Logging(system, this)
        override def preStart(): Unit = {
            super.preStart()
                val udpManager = IO(Udp)
                udpManager ! Udp.Bind(self, localInetAddr)
        }

        override def receive: Receive = {
            case Udp.Bound(localAddress) => {
                log.info(s"UDP Bound to ${localAddress}")
                    val udpConn = sender()
                    timers.startTimerWithFixedDelay(TIMER_KEY, SendDiscovery, delay)
                    context.become(ready(udpConn))
            }
        }
        def ready(udpConnection: classic.ActorRef): Receive  ={
            case Udp.Received(data, senderIp) =>{
                log.info(s"Received ${data} from ${senderIp}")
                    sink ! data.utf8String // forward senders akka address to sink 
            }
            case SendDiscovery => {
                log.info(s"Sending Discovery $SendDiscovery")
                    val cluster = Cluster(context.system.toTyped)
                    udpConnection ! Udp.Send(ByteString(cluster.selfMember.address.toString), remoteAddr) // Send self cluster address to sender
            }
            case b@Udp.Unbind => {
                udpConnection ! b
            }
            case Udp.Unbound => {
                stop(self)
            }
        }
        object SendDiscovery
}

// Sink actor

object ServerActorSink {
    def apply(localInet: InetSocketAddress, multicastAddr: InetSocketAddress): Behavior[String] = Behaviors.setup{ ctx =>
        val udpServerActor = ctx.actorOf(
                classic.Props(classOf[ServerActor], localInet, multicastAddr, ctx.self), "ServerActor")
            Behaviors.empty
    }
}
```

# Client Code
The client will listen for UDP multicasts until it receives one. 
When it receives a multicast message, the client will connect to 
the cluster seed node specified in the message.

```scala
private final class UdpListenerActor(localInet: InetSocketAddress,
                                     multicastAddr: InetAddress)
  extends classic.Actor with classic.Timers {
    val log = Logging(context.system, this)
    override def preStart(): Unit = {
        super.preStart()
        val manager = Udp(context.system).manager
        val udpOpts = List(InetProtocolFamily(), MulticastGroup(multicastAddr))
        manager ! Udp.Bind(self, localInet, udpOpts)
    }

    override def receive: Receive = {
        case Udp.Bound(_) => {
            context.become(ready(sender()))
        }
    }
    private def ready(udpRef: classic.ActorRef): Receive = {
        case Udp.Received(data, senderIp) => {
            val dataStr = data.utf8String
            log.info(s"Connecting to cluster ${dataStr}")
            dataStr match {
                case AddressFromURIString(addr) => {
                    val cluster = Cluster(context.system.toTyped)
                    log.info(s"Received ${dataStr} from $senderIp")
                    cluster.manager ! Join(addr) // Join the server cluster 
                }
            }
        }
        case Joined => context.become(joined(udpRef)) // stop
        case b@Udp.Unbind => {
            udpRef ! b
        }
        case Udp.Unbound => {
            context.stop(self)
        }
    }
    private def joined(udpRef: classic.ActorRef): Receive = {
        case Discover => context.become(ready(udpRef))
    }
}

// Sink Actor

object DiscoveryClientSink {
    def apply(localInet: InetSocketAddress, udpMulticastAddr: InetAddress): Behavior[MemberEvent] = Behaviors.setup{ ctx =>
        val cluster = Cluster(ctx.system)
        val selfMember = cluster.selfMember
        val clientDiscoveryActor = ctx.actorOf(classic.Props(classOf[UdpListenerActor], localInet, udpMulticastAddr), "listener")
        cluster.subscriptions ! Subscribe(ctx.self, classOf[MemberEvent])
        Behaviors.receiveMessagePartial {
            case ClusterEvent.MemberJoined(member) if (member == selfMember) => {
                clientDiscoveryActor ! Joined // Stop discovery again if selfMember joined cluster
                Behaviors.same
            }
            case ClusterEvent.MemberUp(member) if (member == selfMember) => {
                clientDiscoveryActor ! Joined
                Behaviors.same
            }
            case ClusterEvent.MemberLeft(member)  if (member == selfMember) => {
                clientDiscoveryActor ! Discover // Start discovery again if selfMember left cluster
                Behaviors.same
            }
            case ClusterEvent.MemberExited(member) if (member == selfMember) => {
                clientDiscoveryActor ! Discover  
                Behaviors.same
            }
            case ClusterEvent.MemberDowned(member) if (member == selfMember) => {
                clientDiscoveryActor ! Discover 
                Behaviors.same
            }
        }
    }
}
```

# Runner Code

We can configure the system to specify which mode it should run in. 
This allows us to customize the behavior of the system based on the requirements of the service.

```hocon
akka {

  loggers = ["akka.event.slf4j.Slf4jLogger"]

  # Log level used by the configured loggers (see "loggers") as soon
  # as they have been started; before that, see "stdout-loglevel"
  # Options: OFF, ERROR, WARNING, INFO, DEBUG
  loglevel = "ERROR"

  # Log level for the very basic logger activated during ActorSystem startup.
  # This logger prints the log messages to stdout (System.out).
  # Options: OFF, ERROR, WARNING, INFO, DEBUG
  stdout-loglevel = "ERROR"
  actor {
    provider = cluster
  }
  remote {
    artery {
      enabled = on
      transport = tcp
      canonical.hostname = "0.0.0.0"
      canonical.port = 2554
    }
  }

  cluster {
    # auto downing is NOT safe for production deployments.
    # you may want to use it during development, read more about it in the docs.
    auto-down-unreachable-after = 10s
    min-nr-of-members = 1
  }
}

# Enable metrics extension in akka-cluster-metrics.
akka.extensions=["akka.cluster.metrics.ClusterMetricsExtension"]

# Sigar native library extract location during tests.
# Note: use per-jvm-instance folder when running multiple jvm on one host.
akka.cluster.metrics.native-library-extract-folder=${user.dir}/target/native
akka.actor.allow-java-serialization = on

udp-multicast-address = "239.255.100.100" # Multicast address to listen to
udp-multicast-port = 9443
system-name = "sys"
mode = "client" # Mode client or server
singleton-key = "singleton-key" # name for singleton actor
```

Then in ``Main.scala`` we have
```scala
import com.typesafe.config.{Config, ConfigFactory, ConfigValueFactory}
import akka.{actor => classic}

object Main extends App{
    val ipaddr = "X.X.X.X" // get host local ip address 

    val config: Config = ConfigFactory.load("config").withValue(
        "akka.remote.artery.canonical.hostname",
        ConfigValueFactory.fromAnyRef(ipaddr)
    )

    val sysName = config.getString("system-name")

    val sys = classic.ActorSystem(sysName, config)

    val mode = config.getString("mode")

    val port = config.getInt("udp-multicast-port")
    val multicastStr = config.getString("udp-multicast-address")
    val localInet = new InetSocketAddress(port)
    val multicastAddr = InetAddress.getByName(multicastStr)
    val remoteInet = new InetSocketAddress(
            InetAddress.getByName(config.getString("udp-multicast-address")),
            config.getInt("udp-multicast-port")
            )

    val singletonKey = config.getString("singleton-key")

    val cluster = Cluster(sys.toTyped)
    val singletonManager = ClusterSingleton(sys.toTyped)
    val singletonProxy = singletonManager.init(
            SingletonActor(
                Behaviors.supervise(ServerActorSink(localInet, remoteInet))
                .onFailure(SupervisorStrategy.restart), 
                singletonKey
                )
            )
    if (mode == "server"){
        println("Starting in server mode creating cluster ...")
            cluster.manager ! JoinSeedNodes(Seq(cluster.selfMember.address))
    }
    else{
        println("Starting in client mode")
            sys.spawn(DiscoveryClientSink(localInet, multicastAddr), "client_actor")
    }
}
```
The Akka singleton feature ensures that if a singleton actor goes down, 
the oldest node in the cluster will automatically spawn a new singleton server to take its place. 
This helps to prevent a single point of failure and ensures that the system remains available even if the 
original singleton actor fails. This helps to maintain the availability and reliability of the system.


