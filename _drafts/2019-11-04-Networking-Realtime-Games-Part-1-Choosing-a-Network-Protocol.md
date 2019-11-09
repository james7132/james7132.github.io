---
title: "Networking Realtime Games from the Ground Up"
tags: [networking, game dev, unity3d]
---

*This is part 1 in a multi-part series on realtime game networking.*

**tl;dr: Use UDP or UDP based application protocols.**

## Introduction
Since we are starting from the ground up, the first step is choosing how your
game will communicate with remote machines. In this section, we lay the
foundational groundwork for establishing connections and sending messages from
machine A to machine B, and assumes zero knowledge of lower level networking
protocols.

## Considerations
There are several major considerations when choosing a protocol for realtime
games.

Latency is the full end-to-end time it takes to push data correctly from one
machine to another. In realtime games, latency takes precedence above all else,
The choice of protocol can heavily impact the baseline latency of a game.

Reliability: physical hardware and communication processes are not perfect,
packets can be dropped or mangled mid-flight. Most transport protocols
impliictly trust lower level protocols to do error detection and correction, and
will often have relatively simple error checking mechanisms as a last line of
defense against transmission errors, dropping packets if they fail to be sent
correctly. While reliabilty can be implemented at the transport protocol level,
they can also shift the responsibility to the application level.

Ordering guarentees that packets sent in a specific order arrive on the other
side in that same order. Network routing is not static and constantly changing.
This can lead to situations where packets arrive in an inconsistent order (e.g.
packet 2 arrives before packet 1 does). For games, this may sound like it is a
highly desired trait: timed execution of game actions is strictly ordered in
cause and effect. Actually from the network's perspective, it's more like a ball
of wibbly wobbly.. timey wimey... stuff. We can engineer guarenteed ordering,
even if the underlying protocol does not support it.

Bandwidth is one of the lesser concerns. While bandwidth usage definitely must
be kept in mind when developing a networked game, the choice of transport
protocol is often a negligible overhead.

Other considerations may include encryption, player privacy, compression, and
more. These are nice things to have in the transport layer, and recently have
become regular staples in more recently developed protocols, but are not
critical and can be replicated elsewhere.

Here we can see a common theme: "If it can be done at a different level, shift
it elsewhere, prioritize latency." It's true, games often shunt a lot of the
responsiblities of heavier, more feature complete transport layers into the
applicaiton layer, requiring the developer to write their own implementations of
those features. This screams in the face of common sense for most software
developers, who geneally want to avoid repeating and reimplementing the same
thing again, but in return it gives ample opprotunity for the developer to
customize and optimize to their hearts content.

## Option 1: TCP
TCP is the common backbone of almost every reliable internet service you see.
The fact that you are reading this web page is because of TCP. TCP focuses on
deliviering reliable and ordered delivery of data, at the cost of latency.
TCP requires data sent down the connection to be ordered. If packet 32 does not
arrive, even if packet 33 arrives before packet 32, packet 33 will not be
processed until it does. Each packet must also be acknowledged by the other
end of the connection, and will cease to send new data until the latest packet
is acknowledged. This universally blocks all data sent down the connection.
This is allievated with multiple techniques, but is fundamental to the design of
the protocol. It is thus ill-advised to build your realtime game atop TCP.

TCP implementations also commonly use Nagle's Algorithm, which attempts reduce
network congestion by batching up small packets into larger ones before sending
it out, or until some deadline passes, usually 200ms. This lowers the number of
messages in flight across a network and decreases overall network congestino.
However, in realtime games, that's an additional delay to any packet sent. This
can, however, be disabled by using TCP\_NODELAY.

While TCP is incredibly useful in many other contexts, the design principles and
feature choices sacrifice latency for functionality. For realtime games, latency
is everything, and thus core gameplay communications often avoid TCP based
transport and application protocols. However, for non-core gameplay where
reliability is paramount (i.e. match lobby configuration), TCP is a good option.
However, as we shall soon see, it's not the only one. In other game contexts,
like turn-based games, where latency isn't a pressing problem, go nuts with TCP.

## Option 2: UDP
UDP is the other half of the Internet Protocol transport layer.  UDP is a
minimal wrapping around the network layer. It adds host to host port based
multiplexing, as well as the same 16-bit checksum used in TCP. It has a 8-byte
header, compared to the 20 byte header used in TCP. However, beyond that it's
basically dumping the raw application message into the IP data frame. As a
result, it inherits a lot of intrinsic properties of IP: it's connectionless,
and provides no guarentees of reliability or ordering.  However, because of
these properties, UDP is ideal for low latency applications with a minimum send
rate like realtime games.

Even if UDP by itself is stateless and unreliable, multiple reliable connections
based transport protocols have been built atop UDP. Some even support the idea
of channels, or multiple queues of messages within over a single connection.
This would allow reliable, and possibly ordered, messages to be sent without
blocking other streams of data. A common use case in games may be to send
position updates from a server to clients over a unreliable channel, and power
up notifications along a relialbe ordered channels. Posittion updates don't need
to be reliable or ordered, they can be repeatedly sent and only use the latest
recieved one, while power ups usually must be assured to arrive.

## Suboption 1: QUIC
QUIC is a transport level protocol built using UDP as a base that offers much of
the same benefits as TCP. It was originally developed by Google as a
experimental TCP replacement to reduce web service latency, and is expected to
be the base for HTTP/3, with the explicit goal of lowering HTTP latency.
It offers multiplexed connections: multiple streams of reliable data in a single
connection.  Additionally, QUIC offers a new feature via a randomly generated
64-bit connection ID included in every packet that uniquely identifies said
connection. This decouples the connection from the underlying machine IPs and
ports, allowing both ends of the connection to change the routing specifics with
minimal interruptions. This theoretically would allow use cases like mobile apps
to switch from Wi-Fi to mobile networks and back without interrupting ongoing
persistent connections, or for servers to transparently and automatically load
balance existing connections. Also unlike both TCP and UDP, QUIC is always
encrypted, it embeds TLS 1.3 within the protocol itself and the TLS handshake is
required as a part of establishing a connection.

At the time of writing, QUIC has yet to be fully standardized, and thus has not
seen any real use in non-web software. Some companies have adopted it early as a
way to lower latencies for web traffic, but no official standard has been
established yet.

For realtime games, this doesn't differ much from TCP. While built upon UDP,
QUIC currently offers only reliable ordered streams akin to that of TCP. There's
no way to send ad-hoc messages to the remote host within the protocol. However,
this may change in the future: watch this space.

## Suboption 2: HTTP
HTTP(s) deserves a special mention since there is so much network infrastructure
built to support HTTP communications. HTTP is supported virtually everywhere
nowadays. HTTP is built atop TCP (QUIC in HTTP/3). Many web developers and
devops are familiar with HTTP, which makes it useful for general game
infrastruture management. As mentioned in the TCP section, this may include
things like matchmaking, server-authoritative inventories, etc. However, for
realtime games, it is by no means a good protocol to build core gameplay on. In
additon to the laundry list of issues with TCP/QUIC, HTTP adds a large overhead
in the form of text-encoded HTTP headers on each message, which can explode
bandwidth usage when sending over 3,600 messages per minute.

## Suboption 3: Websockets/WebRTC
As of time of writing, websockets and WebRTC are the only ways to do realtime
raw packet exchanges from browsers.

Websockets rides along the existing TCP connection established by a HTTP(s)
request, and allows bidirectional binary message based communications atop the
connection.

WebRTC (Web Real Time Communications) is a collection of protocols built to
establish peer-to-peer realtime communications in the browser. This enables
features like BitTorrent or video conferencing directly from the browser. It's
built atop UDP and a host of other peer-to-peer tools.

If you are developing a realtime browser game, both of these are viable options,
and that entirely depends on the network topology of your game. Websockets may
be useful in client-server topologies where an established well-known server is
connected to by clients. WebRTC is preferable when direct peer-to-peer
connections are desirable. We'll dive deeper into the differences in these
topologies in a later article.

## Suboption 4: Other Protocols
The Internet Protocol Suite primarily defines TCP and UDP as the main driving
transport layer protocols, and many network hardware installations only
understand these two protocols. However, many other transport protocols have
been atop TCP and UDP, extending or altering features. It's worth mentioning
that some proprietary game networks require use of proprietary protocols.
Examples include Steam, Discord, Xbox, and PlayStation networks all require some
closed source library to access their network infrastructure.  Generally
speaking, most of these protocols are UDP based and offer extensions to the
protocol that simplify code or that make developing on their platform easier.
For example, Steam's offers a UDP like peer-to-peer networking library, but
instead of directly specifying an IP, Steam user account IDs are used instead.
It also extends UDP by enabling reliable message sends, and connection
multiplexing akin to that of QUIC.  If you are developing for thsoe platforms
and desire to leverage that infrastructure, you consider using these protocols
over raw UDP.

## Conclusion

