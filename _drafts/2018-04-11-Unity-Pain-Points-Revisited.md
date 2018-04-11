---
title: "Unity3D Pain Points: 2018 Update"
tags: [unity3d]
---

About one year ago on this blog, I enumerated the pain points while working with
the Unity3D engine. In this post I would like to revisit these pain points and
gauge how Unity Technologies has dealt with these issues over the past year.

## Case 1. An Outdated Mono
A year ago, the scripting system had an outdated compiler, garbage collector,
runtime, and standard library. Over the past year, Unity has been scrambling to
catch up with almost 16 years of updates to the Mono/.NET ecosystem.  Now in
April, Unity 2018.1 is soon to be released with C# 6.0, a .NET 4.6
Equivalent runtime, and .NET Standard 2.0 support. There's even a few
experimental C# 7.0 features that one can opt into (via mcs.rsp) such as ref
returns. This has been a long wait for these updates and many of the updates
makes developing code in Unity that much smoother.

The one caveat is that shared Asset Store packages still need to support many
versions of Unity, often combining conditional compilation with source
distribution. Many Asset Store publishers may not be able to leverage these
updates until Unity 2017 has long since been deprecated.

This also has not been without many teething issues as the new scripting engine
components were integrated with Unity. Having avidly used the .NET Experimental
4.6 runtime there were multiple issues that cropped up over the past year: for
example, running Unity in batch mode and running unit tests while on the new
runtime seems to cause a deadlock. This caused numerous builds on Unity Cloud
Build to run for almost an hour before being forcibly terminated by the service,
and ultimately required me to disable automatically running tests on builds until
the issue is fixed.

## Case 2. Poor API Design
As stated in the previous post, Unity's scripting APIs had many fundamental
issues that lead to poor testability, bad async support, and generally required
heavy use of boilerplate code. Most of these issues stemmed from the need to
tightly couple user created scripts (MonoBehaviours) with the containers that they
lived in (GameObjects). For the most part, the general GameObject-Component based
APIs for controlling game elements remains rigidly in place today as much as it
did last year. MonoBehavviours remain as difficult to test and as boilerplate-prone
as always.

However, that is not to say that there have not been improvements in these
realms. On the contrary, the new Unity ECS system promises to resolve all of the
metioned issues and more. Instead of components that live both in managed and
unmanaged memory, the new ECS components are lightweight plain old C# structs.
You can construct them like any other C# object, making it much easier create
modular systems (via composition of components) and write tests. Likewise, since
they're simple data objects that are meant only to hold data, there's no extra
boilerplate code needed to bootstrap what is considered idiomatically correct
code. Who cares if the fields are public? There's no need to add accessor
properties or `SerializeField` annotations for a simple data object. Finally, the
ECS was built to fully utilize multi-core processors: as components are
value-typed data objects without inherent state shared between threads, it's
trivial to parallelize operations across multiple threads, and the Unity Jobs API
attempts to make this as simple writing normal single threaded code, albiet it's
not as clean cut as they initially advertised it to be.

I've since rebuilt my [bullet hell library](https://github.com/james7132/DanmakU)
using the Unity Jobs System, and, by far, it has been one of the most
straightfoward multithreading experiece I've had in recent memory, particuarly
given the constraints on the amount of garbage generated. Based on the
engineering requirements presented, I applaud Unity Technologies for a well-built
system that delivers on exactly what it had promised.

### Async Support
The general availablitiy of the new .NET 4.6 runtime also allowed use of the C#
5.0 `async`/`await` syntax for handling asynchronous operations. The old system
of using coroutines (engine managed iterator blocks) still remains the
recommended way to do asynchronous operations in Unity. However, using the new
`AsyncOperation.completed` callback with `System.Threading.Tasks` it is possible
to bridge Unity defined async operations and the `async`/`await` pattern without
needing a coroutine or a MonoBehaviour to mediate between the two:

```csharp
public static Task<T> ToTask<T>(this T operation) where T : AsyncOperation {
  var completionSource = new TaskCompletionSource<T>();
  operation.completed += (op) => completionSource.TrySetResult(op);
  return completionSource.Task;
}
```

## Case 3: No Multiproject Management
Unity introduced Assembly Definitions in 2017.3. This feature doesn't directly
address package and depedency management issues, but it does resolve long
compilation times. Assembly definition files are Unity's solution to logically
seperating different binaries. I'm not sure why Unity chose not to use already
existing project definition files (i.e. \*.csproj), but it's better than waiting
for several minutes for code to recompile otherwise minor changes.

There is a reasonable workaround if using git as your source control: git
submodules allows packages seperation. The main caveat to this is that

## Case 4: No Package Manager
The new Unity Package Manager, introduced in 2017.2, which is currently slated to
allow Unity users to update components of the engine without needing to
update the entire engine in a monolithic fashion. It's currently only used for
Unity published packages, but the hope is that eventually third-party users will
be able to publish packages through it as well.

Regarding other code package managers, most notably Nuget, currently is not on
the projected roadmap at all. Devs still need to micromanage DLL plugins for
third-party plugins.

## Conclusion
Among the issues originally enumerated, Unity Technologies seems to gradually
addressing these issues, as well as actively engineering to avoid these pitfalls
in the future.
