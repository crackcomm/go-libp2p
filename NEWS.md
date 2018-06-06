# go-libp2p release notes

## 6.0.0

We're pleased to announce go-libp2p 6.0.0. This release includes a massive
refactor of go-libp2p that paves the way for new transports such as QUICK.
Unfortunately, as it is a sweeping change, there are some breaking changes,
*especially* for maintainers of custom transports.

Below, we cover the changes you'll likely care about.

### For Transport Maintainers

For transport maintainers, quite a bit has changed. Before this change,
transports created simple, unencrypted, stream connections and it was the job of
the libp2p Network (go-libp2p-swarm) to negotiate security, multiplexing, etc.

However, when attempting to add support for the QUIC protocol, we realized that
this was going to be a problem: QUIC already handles authentication and
encryption (using TLS1.3) and multiplexing. After much debate, we inverted our
current architecture and made transports responsible for encrypting/multiplexing
their connections (before returning them).

To make this palatable, we've also introduced a new ["upgrader"
library](https://github.com/libp2p/go-libp2p-transport-upgrader) for upgrading
go-multiaddr-net connections/listeners to full libp2p transport
connections/listeners. Transports that don't support encryption/multiplexing out
of the box can expect to have an upgrader passed into the constructor.

To get a feel for how this new transport system works, take a look at the TCP
and WebSocket transports and the transport interface documentation:

* [TCP Transport](https://github.com/libp2p/go-tcp-transport)
* [WebSocket Transport](https://github.com/libp2p/go-ws-transport)
* [Transport Interface](https://godoc.org/github.com/libp2p/go-libp2p-transport)

#### go-addr-util

In go-addr-util, we've removed the `SupportedTransportStrings` and
`SupportedTransportProtocols` transport registries and the associated
`AddTransport` function. These registries were updated by `init` functions in
packages providing transports and were used to keep track of known transports.

However, *importing* a transport doesn't mean any libp2p nodes have been
configured to actually *use* that transport. Therefore, in the new go-libp2p,
it's go-libp2p-swarm's job to keep track of which transports are supported
(i.e., which transports have been registered with the swarm).

We've also removed the associated `AddrUsable`, `FilterUsableAddrs`, and
`AddrUsableFunc` functions.

### For End Users

Libp2p users should be aware of a few major changes.

* Guarantees and performance concerning connect/disconnect notification
  processing have improved.
* Handling of half-closed streams has changed (READ THIS SECTION).
* Some constructors and method signatures have changed slightly.

#### Bandwidth Metrics

To start out on an unhappy note, bandwidth metrics are now less accurate. In the
past, transports returned "minimal" connections (e.g., a TCP connection) so we
could wrap these transport connections in "metrics" connections that counted
every byte sent and received.

Unfortunately, now that we've moved encryption and multiplexing down into the
transport layer, the connection we're wrapping has significantly more
under-the-covers overhead.

However, we do hope to improve this and get even *better* bandwidth metrics than
we did before. See
[libp2p/go-libp2p-transport#31](https://github.com/libp2p/go-libp2p-transport/issues/31)
for details.

#### Notifications

This release brings new, performant, easy to understand guarantees around libp2p
connect/disconnect event ordering:

1. For any given connection/stream, libp2p will wait for all connect/open event
   handlers to finish exit before triggering a disconnect/close event for the
   connection/stream.
2. When a user calls the `Close` (or `Reset`) method on a connection or stream,
   go-libp2p will process the close event asynchronously (i.e., not block the
   call to `Close`). Otherwise, a call to `Close` from within a connect event
   handler would deadlock.
3. Unless otherwise noted, events will be handled in parallel.

What does this mean for end users? Well:

1. Reference counting connections to a peer using connect/disconnect events
   should "just work" and should never go negative.
2. Under heavy connect/disconnect loads, connecting to new peers should be
   faster (usually).

In the past, (dis)connect and stream open/close notifications have been a bit of
a pain point. For a long time, they were fired of in parallel and one could, for
example, process a disconnect notification before a connect notification (we had
to support *negative* ref-counts in several places to account for this).

After no end of trouble, we finally "fixed" this by synchronizing notification
delivery. We still delivered notifications to all notifiees in parallel, we just
processed the events in series.

Unfortunately, under heavy connect/disconnect load, new connections could easily
get stuck on open behind a queue of connect events all being handled in series.
In theory, these events should have been handled quickly but in practice, it's
very hard to avoid locks *entirely* (bitswap's event handlers were especially
problematic).

Worse, this serial delivery guarantee didn't actually provide us with an
*in-order* delivery guarantee as it was still possible for a disconnect to
happen before we even *started* to fire the connect event. The situation was
slightly better than before because the events couldn't overlap but still far
from optimal.

#### Conn.GetStreams Signature Change

The signature of the `GetStreams` method on `go-libp2p-net.Conn` has changed from:

```go
GetStreams() ([]Stream, error)
```

To:

```go
GetStreams() []Stream
```

Listing the streams on an open connection should never involve IO or do anything
that can fail so we removed this error to improve usability.

### Libp2p Constructor

If you're not already doing so, you should be using the `libp2p.New` constructor
to make your libp2p nodes. This release brings quite a few new options to the
libp2p constructor so if it hasn't been flexible enough for you in the past, I
recommend that you try again. A simple example can be found in the
[echo](https://github.com/libp2p/go-libp2p/tree/feat/refactor/examples/echo)
example.

Given this work and in an attempt to consolidate all of our configuration logic
in one place, we've removed all default transports from go-libp2p-swarm.

TL;DR: Please use the libp2p constructor.

### Zombie Streams

libp2p streams provide two "close" methods: `Close` and `Reset`. `Close` closes
the stream for writing but leaves it open for reading. `Reset` "kills" the
stream (canceling any in-progress writes and any buffered data waiting to be
sent). So, the *polite* way to close a connection is to call `Close` and then
read from the connection until you see an EOF from the other end (possibly
timing out and resetting the stream if the other side doesn't respond).

Calling `Close` on a libp2p stream closes stream for writing but not for
reading. That means that the stream isn't full
closes *their* end.
