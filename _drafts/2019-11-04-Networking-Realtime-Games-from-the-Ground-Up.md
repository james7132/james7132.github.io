---
title: "Networking Realtime Games from the Ground Up"
tags: [networking, game dev, unity3d]
---

Every video gamer since 90s has at one point or another in their lives thought:
"I want to make a game. One like Halo World Of Warcraft Diablo Smash Bros in
Minecraft. It can't be that hard right?". Oh how wrong we all were. Realtime
networked multiplayer games are one of the most difficult things to develop and
is an open, unsolved problem.

Game development is not simple: it's one of the most grueling software
development paradigms, surfacing some of the most complex problems a developer
will probably ever tackle. "How do I make users think the agent they are
interacting with is actually intelligent?", "How do I ensure that my visuals are
consistent from platform to platform?.", or "How do I actively engage the player
in a meaningful way?" are common problems game developers grapple with on a
daily basis.

Realtime games only expand this by adding strict performance requirements. 16
milliseconds. The entire game loop must consistently run in this time to
maintain a 60 frames per second render rate. Any large perturbation to that
standard in active gameplay, and it will negatively impact the player's
experience. This includes: rendering, input processing, AI, player logic, audio
processing, physics, and more. All must be done in that tight time requirement.
Even running on all CPU cores, any subsystem that takes longer will delay all
others. Here, performance is king: code cleanliness, best practices, and even
code safety be damned. Anything to shave another few milliseconds off of that
game loop.

Networked games also expand upon these issues by adding in the complexities of
distributed computing. "How do I architect my game to be communicatable over the
netwokr?", "Game clients are commonly behind NATs, how do we punch through those
blockades?", "How do I synchronize the state between multiple clients?", or "How
do I prevent cheaters from negatively impacting other players' experiences
online?". The complexities of modern networked communications is a deep rabbit
hole that developers have labored much of their working lives over.

Realtime networked games compound all of these issues into one towering
behemoth. With network latencies across the Earth reaching beyond 200
milliseconds, multiple game ticks will pass while a packet flies across the
globe. How do we overtake the speed of light itself to provide a smooth
experience for the player?

This is the first part in a multi-part series of posts detailing the approaches
of how I tackled many of these problems as I am working on Fantasy Crescendo, a
realtime networked platform fighting game. This series is tailored for Unity and
C#, including code snippets I wrote for the game, but the ideas and subjects
are not isolated based on engine or programming language. This series is also
bottom up, building the networking for a game almost entirely from scratch.

 * [Part 0: How NOT to write a realtime netwokred game]()
 * [Part 1: Choosing a Network Protocol]()
 * [Part 2: Message Serialization]()
 * [Part 3: Choosing a Network Topology]()
 * [Part 4: Realtime Networking Simulation Strategies]()
