---
title: "Networking Realtime Games from the Ground Up"
tags: [networking, game dev, unity3d]
---

*This is part 1 in a multi-part series on realtime game networking. This is a
technical article is not meant for a general audience and assumes at least a
rudimentary understanding of with low level programming concepts like pointers
and bitwise operations..*

tl;dr: Use UDP or UDP based application protocols.

Since we are starting from the ground up, the first step is choosing how your
game will communicate with remote machines. This is by no means a comprehensive
look at all of the options, but an attempt to summarize what to look for when
setting up.

There is next to zero code here on this page since each protocol is different.
Ultimately it comes down to finding the remote clients to send packets to, maybe
setting up a connection, then sending and receiving packets to and from them.

## Option 1: TCP
TCP is the common backbone of almost every reliable internet service you see.
The fact that you are reading this web page is because of TCP. TCP focuses on
deliviering reliable and ordered delivery of data, at the cost of latency.
Therein lies the issue with TCP for realtime games: latency. TCP requires data
sent down the connection to be ordered. If packet 32 does not arrive, even if
packet 33 arrives before packet 32, packet 33 will not be processed until it
does. Each packet must also be acknowledged by the other end of the connection
before, and will cease to send new data until the latest packet is acknowledged.
This is allievated with multiple techniques, but is fundamental to the design of
TCP.

TCP implementations also commonly use Nagle's Algorithm, which attempts to bunch
up small packets into larger chunks before sending it out, or until some
deadline passes, usually 200ms. This is usually done to minimize network
congestion by lowering the number of packets in flight across a network at any
given time. However, in realtime games, that's a potential addition of 200ms of
delay to any packet sent on the network. This can, however, be disabled by using
TCP\_NODELAY when initializing the connection.

While TCP is incredibly useful in many other contexts, the design principles and
featrue choices sacrifice latency for functionality. For realtime games, latency
is everything, and thus core gameplay communications often avoid TCP based
transport and application protocols. However, for non-core gameplay where
reliability is paramount (i.e. match lobby configuration), TCP can be used.

## Option 2: UDP
UDP is the other half of the Internet Protocol transport layer. As a base, it
provides rudimentary message based communications. It is not connection based,
each message is fire and forget. By itself, UDP is stateless, and provides no
guarentees of reliability or ordering, but does guarentee that if a message
arrives, it is as it was sent by the origin, with mangled packets being dropped
entirely. However, because of these properties, UDP is ideal for low latency
applications like realtime games.

Even if UDP by itself is stateless and unreliable, multiple reliable conneciton
based transport protocols have been built atop UDP. Some even support the idea
of channels, or multiple queues of messages within over a single connection.
This would allow reliable, and possibly ordered, messages to be sent without
blocking other streams of data. A common use case in games may be to send
position updates from a server to clients over a unreliable channel, and power
up notifications along a relialbe ordered channels. Posittion updates don't need
to be reliable or ordered, they can be repeatedly sent and only use the latest
recieved one, while power ups usually must be assured to arrive.

## Suboption 1: HTTP
HTTP(s) deserves a special mention since there is so much network infrastructure
built to support HTTP communications. HTTP is supported virtually everywhere
nowadays, and up until HTTP/3, it is based on TCP connections. Many web
developers and devops are familiar with HTTP, which makes it good for general
game infrastruture management. As mentioned in the TCP section, this may include
things like matchmaking, server-authoritative inventories, etc. However, for
realtime games, it is by no means a good protocol to build atop of. In additon
to the laundry list of issues with TCP, HTTP adds a large overhead in the form
of text-encoded HTTP headers on each message, which can up when sending over
3,600 messages per minute.

## Suboption 2: Websockets
A further suboption of HTTP are websockets. As of time of writing, websockets
are the only way to do bidriectional realtime raw packet exchanges from
browsers. Websockets run over the established TCP connection of a HTTP request,
and continue to do two-way message framing akin to UDP. In the case of realtime
web games, this may be the best option available, but it still is built upon TCP
and carries some of the same intrinsic issues it has. This may change when a
websocket standard for HTTP/3 is released.

## Suboption 3: Proprietary Protocols

## Encryption

