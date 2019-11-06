---
title: "Networking Realtime Games from the Ground Up"
tags: [networking, game dev, unity3d]
---

*This is part 2 in a multi-part series on realtime game networking. This is a
technical article is not meant for a general audience and assumes at least a
rudimentary understanding of with low level programming concepts like pointers
and bitwise operations..*

Now that we can establish simple connections and send arbitrary binary
messages. The next part is to start making sure we read and write meaningful
information from these messages.

The first thing most developers will think of is JSON or other text based
serialization methods. It works great in HTTP servers, why not here? Sure, JSON
will probably work as a start, but we will soon see it's too large an encoding
for our messages, and beyond bandwidth there are hidden costs to having large
messages in realtime games. Our goal is realtime games over the network, and for
that we must decimate latency. Part of latency is the potential for packet loss.
Most networks have a defined MTU (Maximum Transmission Unit), before messages
are chunked into smaller messages. If any of these chunks are lost or mangled
mid transmission, the entire message is lost, increasing latency. As more chunks
are added to a message, the likelihood of any individual chunk gets mangled
expoentially increase, so minimizing chunking is essential. For most ethernet
based systems, this lowest common MTU found is ~1200-1500 bytes. Use of JSON or
other text based protocols is highly wasteful of this precious space.

So what alternatives are there?

 * BSON, MessagePack - A step in the right direction, a binary encoding of
   JSON-like data. Still encodes the field names as strings, which is wasteful.
 * Protocol Buffers, FlatBuffers, Cap'n Proto - Binary serializable message
   formats that relies on all participants to be in agreement at compile time
   the message definition. Reduces field names from strings to integers. Covers
   90% of what is listed below.
 * Custom Binary Serializer - A custom written serialization format for each
   message. Gives the developer tight control over exactly what gets written to
   and read from the wire.

To maximize savings on the wire, we'll be going more into depth into the custom
binary serializer. The others can easily be implemented via a preexisting
library.  What is detailed below is the design ideas behind Hourai Networking's
custom serializer and how to utilize it to it's maximum potential.

## Serializing and Deserializing Fields
One may ask: why not just `memcpy` the message as if it were a C struct directly
into the message buffer? While this may work in some cases, it doesn't
universally work, and may take up more room on the wire than it needs to.

Each device's processor is either little endian or big endian. This determines
the byte ordering for multi-byte values. Every integer and derived type is
affected by this. Load a little endian struct from a big endian encoded message
and you will get garbage. Message protocols like protobufs solve this by forcing
a certain endianness (usually big-endian) in their wire formats. We'll aim to do
the same with our serializer.

### Basics: Symmetrical Serialization/Deserializtion Code
First we should define some basic data types for serialization:
```csharp
public unsafe struct Serializer {
  byte* _start;
  byte* _current;
  byte* _end;

  public static Serializer Create(byte* start, int len) {
    return new Serializer {
      _start = start,
      _current = start,
      _end = _start + len
    };
  }

  void CheckRemainingSize(int size) {
    if (_current + size >= _end) {
      throw new IndexOutOfRangeException("Serializer: Reached end of fixed size buffer");
    }
  }

  void Write(byte value) {
    CheckRemainingSize(1);
    *_current++ = value;
  }

}

public unsafe struct Deserializer {
  byte* _start;
  byte* _current;
  byte* _end;

  public static Deserializer Create(byte* start, int len) {
    return new Deserializer {
      _start = start,
      _current = start,
      _end = _start + len
    };
  }

  void CheckRemainingSize(int size) {
    if (_current + size >= _end) {
      throw new IndexOutOfRangeException("Serializer: Reached end of fixed size buffer");
    }
  }

  byte ReadByte() {
    CheckRemainingSize(1);
    return *_current++;
  }
}

public interface INetworkSerializable {

  void Serialize(ref Serializer serializer);
  void Deserialize(ref Deserializer deserializer);

}
```

Here `Serializer` and `Deserializer` are mirrors of each other, providing a
write-only and read-only interface for creating message buffers. Not shown here
are the various overloads of `Serializer.Write` and `Deserializer.ReadXXX`.

Why use unsafe structs? Structs allocated on the stack are extremely quick to
allocate and deallocate. The end goal is to read to or write from existing
continguous buffers in memory, which ultimately means sequentially reading bytes
based on the position in memory. Switching to using pointers also skips the C#
bounds checking when accessing managed arrays. Some functions will repeatedly
access sequential bytes we are assured to be safe to access. Additional bounds
checking only slows the encoding.

The expected use of these types is to define symmetrical serialization and
deserialization routines on implementation of `INetworkSerializable`. For
example:

```csharp
public struct TwinJoysticks : INetworkSerializable {

  public Vector2 Left;
  public Vector2 Right;

  public void Serialize(ref Serializer serializer) {
    serializer.Write(Left.x);
    serializer.Write(Left.y);
    serializer.Write(Right.x);
    serializer.Write(Right.y);
  }

  public void Deserialize(ref Deserializer deserializer) {
    Left.x = deserializer.ReadFloat();
    Left.y = deserializer.ReadFloat();
    Right.x = deserializer.ReadFloat();
    Right.y = deserializer.ReadFloat();
  }

}
```

Note how the values are wrtten to and read from in the exact same order. The
ordering implicitly defines which field is written to and read from. This means
changes to the serialization format or ordering will likely be incompatible with
previous versions, but for most games, backwards compatibility in the network is
not a first class concern. It also means there are no field markers in the
message, meaning the message is only the size of the values held by the message,
and contain no structural overhead. The above `TwinJoysticks` message is always
16 bytes large: 4 bytes for each float encoded. If encoded to JSON, the message
would be `{"Left":{"x":0.0,"y":0.0},"Right":{"x":0.0,"y":0.0}}` and be at least
52 bytes, 225% larger than the custom serializer. Likewise a protobuf encoding
would be at minimum 21 bytes, accounting for the additional field markers.

### Reducing Message Size: Varint Encoding
Unsigned integers (`uint`) are 4 bytes in C#. This means, ignoring endianness,
values that don't use the top 2-3 bytes are wasting that space and are filling
it with nothing but 0s. Varint encoding fixes this by encoding only the bytes
that actually have information. There are multiple ways of doing varint
encoding, here we will be using SQLite's implementation:
https://sqlite.org/src4/doc/trunk/www/varint.wiki. The SQLite implementation has
a maximum overhead of one byte, so a 4 byte integer will encode to 1-5 bytes ,
and a 8 byte integer will encode to 1-9 bytes. Overall, if smaller numbers are
more likely to appear than big numbers (i.e. game data vs random seeds), there
will be signifigant savings from using varint encoding.

### Reducing Message Size: ZigZag Encoding
One downside to using varint encoding is that negative numbers for signed
integers will always have the highest bits set:

```
0  = 00000000000000000000000000000000
-1 = 11111111111111111111111111111111
```

When fed directly through a varint encoder, this always results in negative
numbers being encoded as a maximum length varint. The commonly seen solution is
to encode signed integers via ZigZag encoding. The idea is to encode numbers
with low absolute value encode with a small varint by zig-zagging across the
number line back and forth through the positive and negative numbers. -1 becomes
1, 1 becomes 2, -1 becomes 3, and so on.

|Signed Original|Encoded As|
|:--|:--|
|0| 0|
|-1|  1|
|1| 2|
|-2|  3|
|2147483647|  4294967294|
|-2147483648| 4294967295|

This is efficiently computed via the following snippet:

```csharp
(n << 1) ^ (n >> 31)      // For 32-bit integers
(n << 1) ^ (n >> 63)      // For 64-bit integers
```

Again, this assumes that the message values are biased towards numbers with low
absolute value. If this isn't the case, it may actually be more efficient to
encode the entire integer on the wire.

### Reducing Message Size: Compression
One general way to further cut bandwidth and fragmentation is through use of
compression. There are plenty of compression algorithms out there to use on
these messages (DEFLATE, Brotli, LZMA, LZ4, etc). Choose one that works for you,
I used LZ4 (https://github.com/HouraiTeahouse/LZF). Binary data generally
doesn't compress well, but repetitive structures like those in games may
actually see some gains via compression. I actually added a conditional
compression step.  Sometimes compression actually makes the message bigger than
it is uncompressed, usually by way of adding a header that is larger than  the
original message. In these cases, I added yet another one byte header: 0 if the
rest of the message is uncompressed, and any other number if it is compressed.
Below is the compression code:

```csharp
var out_buffer = (byte*)UnsafeUtility.Malloc(msg_size + 1, Allocator.Persistent);
int size = CLZF2.TryCompress(msg, msg_size, out_buffer + 1, msg_size);
// Outputs 0 if compression failed
if (size > 0) {
  out_buffer[0] = 1;
  size = size + 1;
} else {
  out_buffer[0] = 0;
  UnsafeUtilty.MemCpy(out_buffer + 1, msg, msg_size);
  size = msg_size + 1;
}
UnsafeUtility.Free(msg);
return size;
```

### Manual Message Size Reduction
In the previous sections, we went over how the serializer could be improved to
save space; however, that can only go so far. If bandwidth and fragmentation are
still a problem, the only way is for the developer to hand optimize the
messages. While the upfront cost to the developer is high, the benefits can be
immense.

For this section, I will be using Fantasy Crescendo's input structure as an
example. It consists of two 2D joysticks (Move and Smash), and 5 buttons (
Attack, Special, Grab, Shield, and Jump). This consists at runtime of 4 32-bit
floats, and 5 4-byte boolean values, for a grand total size of 36 bytes. In the
next two sections, I will be reducing this to 5 bytes, a greater than 6-fold
reduction in size.

#### Bit-Packing
Booleans are extremely wasteful, espeicaly in C#. Most systems represent `bool`
types as a 4-byte integer: 0 for false, anything else for true. If this is true,
31/32 bits are basically unused.  You can save 75% of the space by converting
all of thse 4-byte booleans to 1 byte easily:

```csharp
byte value = boolValue ? 1 : 0;
serializer.Write(value);
```

This would reduce the size of the buttons to 5 bytes, but we can go even
further. Each bit in a byte can hold a boolean value, fitting the entire 5
boolean values into a single byte.

```csharp
void WriteBit(ref byte bitmask, ref int bit, bool value) {
  bitmask |= ((value ? 1 : 0) << bit);
  bit++;
}

void WriteButtons(ref Serializer serializer) {
  int bit = 0;
  byte buttons = 0;
  WriteBit(ref buttons, ref bit, Attack);
  WriteBit(ref buttons, ref bit, Special);
  WriteBit(ref buttons, ref bit, Shield);
  WriteBit(ref buttons, ref bit, Grab);
  WriteBit(ref buttons, ref bit, Jump);
  serializer.Write(buttons);
}
```

#### Downcasting Numeric Types
Floats are 4 bytes, but all I need in my inputs is the range from [-1, 1], and
256 distinct values in that range is more than enough for my purposes. It is
thus trivial to transform an input value into a signed byte:

```csharp
sbyte AxisFromFloat(float value) {
  var clamped = Mathf.Clamp(value, -1.0f, 1.0f);
  var scaled = clamped * sbyte.MaxValue; // 127
  return (sbyte)scaled;
}

float FloatFromAxis(sbyte value) {
  return value / (float)sbyte.MaxValue; // 127
}
```

This will reduce the 16 bytes used on both input joysticks to 4 bytes.

#### Delta Compression
Delta compression is a stateful compression technique: instead of sending a full
message, send only the bits that change. There are various ways of attempting
this, but all of them involve storing the same state on both ends of the
connection, applying a binary diffing algorithm (i.e.
[FossilDelta](https://github.com/endel/FossilDelta)), then reconstructing the
updated state on the other end of the connection.

However, this can be quite difficult to implement properly on unreliable and
unordered connections. One other option is to apply intra-message delta
compression. Due to the unreliable nature of the underlying transport protocols,
most games will repeatedly send batches of messages until it is acknolwedged
(i.e. player input). These messages tend to be very similar to each other and
delta compress very well. Store the complete first message, then delta
compressing the rest. The remote can then reconstruct the entire batch by
progressively applying the delta patches.

### Minimizing Garbage Collection
This is problem specific to C# and other garbagge collected language runtimes.
If your game does not use these kinds of runtimes (i.e. C, C++, Rust), feel free
to skip this over.

C# is a garbage collected language. Objects are commonly allocated on the heap,
and this includes the `byte[]` buffers. Allocating and copying data from bufffer
to buffer will add more garbage to collect at the next collection cycle. In
engines like Unity, where the garbage collector can suddenly induce a 10-100ms
lag spike, allocating garbage every game tick to send messages will regularly
induce this spike, so we should avoid allocating garbage as much as possible.
There are several ways we can minimize our garbage produced.

#### Pooling
In C# the `System.Buffers.ArrayPool` class can be used to
preallocate and pool `byte[]` buffers. The idea is simple: reuse buffers instead
of deallocating them. Instead of calling `new byte[2048`, call
`ArrayPool<byte>.Shared.Rent(2048)` instaed. This will pull one from the pool if
there is one, and allocate only if there are no available buffers of a specified
size. It should be noted that ArrayPool returns based on minimum size, not exact
size, so the array may be bigger than expected. When the use of the array is
done, it is imperative to call `ArrayPool<byte>.Shared.Return(buffer)` otherwise
the buffer will not be reused. This can be forgotten and it

#### Native Allocations
One way to avoid managed allocations and thus garbage collection is to allocate
memory that isn't managed. In Unity,
`Unity.Collections.LowLevel.Unsafe.UnsafeUtility` exposes C# style allocation
functions: equivalent calls for `malloc`, `free`, `memcpy` are provided under
that class. We can use this method by calling `UnsafeUtility.Malloc` to allocate
a buffer, then `UnsafeUtility.Free` to free it is no longer used. However, we
MUST remember to call Free, or else there will be a memory leak.

#### Stack Allocation
The final option is to just allocate the buffer directly on the stack. In C#,
each thread has a maximum stack size of 1MB, which is ~700x larger than the 1500
MTU we are trying to keep under. We can allocate a buffer on the stack in C#
with:

```csharp
byte* buffer = stackalloc byte[1500];
```

The buffer will automatcally be deallocated as soon as stack is unwound, which
uusally means when the local function exits. It is also importantt to note that
no bounds checking is done, the buffer is a fixed size, and what's returned is a
unsafe pointer without knowledge of the size of the buffer. We can easily
overwrite parts of the stack if we overshoot the end of the buffer. Thus it's
best that exercise caution when using this method.

The upside is that this method is blistering fast. Stack allocation and
deallocation only involves moving the stack pointer, a virtually free
operation. Additionally, since we know our MTU, we can confidently allocate 1500
bytes on the stack knowing that our messages will never be bigger. If messages
are indeed bigger, we will need to use one of the other two methods.

## Message Differentiation
After implementing all of the above we are finally able to serialize arbitrary
developer defined messages to a buffer. But how do we differentiate between each
message? They all are mangled byte strings and there's no way to tell them
apart. The answer is a message header: a value preceding the message that
denotes which message type it is. Generally this is determined by the developer
and static at runtime. If we look at the Hourai Networking source code we see a
pattern show up:

```csharp
void SendMessage<T>(in T message, ...) where T : INetworkSerializable {
  // Allocate a stack buffer
  byte* buffer = stackalloc byte[2048];
  // Create a serializer around it
  var serializer = Serializer.Create(buffer, 2048);
  // Write a 1-byte message header.
  serializer.Write((byte)MessageHeader);
  // Write the rest of the message
  message.Serialize(ref serializer);
  // Send the message
  SendNetworkMessage(serializer.AsFixedBuffer(), serializer.Position, ...);
}
```

When reading the message, we first read the header to decide what message type
it is, then look for a handler to handle that message:

```csharp
void OnNetworkMessage(byte[] msg, int len) {
  fixed (byte* msgBuf = msg) {
    // Create deserializer around message
    var deserializer = Deserializer.Create(msgBuf, len);
    // Read the header
    var header = deserializer.ReadByte();
    // Handle the message
    Handlers[header]?.Invoke(deserializer, ...);
  }
}
```

This lets us send multiple types of messages through a single connection.

