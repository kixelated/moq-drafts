---
title: "Media over QUIC - Lite"
abbrev: "moql"
category: info

docname: draft-lcurley-moq-lite-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
v: 3
area: wit
workgroup: moq

author:
 -
    fullname: Luke Curley
    email: kixelated@gmail.com

normative:
  moqt: I-D.ietf-moq-transport

informative:

--- abstract

Moq-Lite is designed to fanout live content from publishers to any number of subscribers across the internet.
Liveliness is achieved by using QUIC to prioritize the most important content, potentially starving or dropping other content, to avoid head-of-line blocking while respecting encoding dependencies.
While designed for media, it is an agnostic transport, allowing relays and CDNs to forward content without knowledge of codecs, containers, or encryption keys.

--- middle

# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Rationale
This draft is based on MoqTransport [moqt].
The concepts, motivations, and terminology are very similar and when in doubt, refer to the upstream draft.
However, a few things have been renamed because I think they make the concepts clearer (and I couldn't help myself).

I absolutely believe in the motivation and potential of Media over QUIC.
The layering is phenomenal and addresses many of the problems with current live media protocols.
I fully support the goals of the working group and the IETF process.
But it's been difficult to design such an experimental protocol via committee.

MoqTransport has become too complicated.
There are too many messages, optional modes, and half-baked features.
Too many hypotheses, too many potential use-cases, too many diametrically opposed opinions.
This is expected (and even desired) as compromise gives birth to a standard.

But the specification has become a distraction and I think it impedes progress.
I believe that MoQ is an experiment that needs to be proven before it can be cemented.
We should spend more time building an *actual* application and less time arguing about a hypothetical one.
I can't waste any more time on FETCH and other fringe functionality.

moq-lite is the bare minimum needed for a real-time conferencing application *only*.
Every feature from MoqTransport that is not necessary (or has not been implemented yet) has been removed for simplicity.
This includes many great ideas (ex. group order) that are just not worth implementing at the moment.
This draft is the current state, not the end state.


# Concepts
moq-lite consists of:

- **Session**: An established QUIC connection between a client and server.
- **Broadcast**: A collection of Tracks from a single publisher.
- **Track**: An series of Groups, each of which can be delivered and decoded *out-of-order*.
- **Group**: An series of Frames, each of which must be delivered and decoded *in-order*.
- **Frame**: A sized payload of bytes representing a single moment in time.

The application determines how to split data into broadcast, tracks, groups, and frames.
The moq-lite layer provides fanout, prioritization, and caching even for latency sensitive applications.

## Session
A Session consists of a connection between a QUIC client and server.

A session is established after the necessary QUIC, WebTransport, and moq-lite handshakes.
The moq-lite handshake consists of version and extension negotiation.

The intent is that sessions are transparently chained together via relays.
A broadcaster could establish a session with an CDN ingest edge while the viewers establish separate sessions to CDN distribution edges.
A moq-lite session is hop-by-hop, but the application should be designed end-to-end.

## Broadcast
A Broadcast is a collection of Tracks from a single publisher.
This cooresponds to a MoqTransport "track namespace".

A publisher may produce multiple broadcasts.
The available broadcasts are advertised via an ANNOUNCE message and a subscriber can discover available broadcasts via an ANNOUNCE_PLEASE message.
These announcements are live and can change over time, allowing for dynamic origin discovery.

A broadcast consists of any number of Tracks.
These tracks are related by name only and there's no requirement that they have the same content.
Tracks are not advertised as part of a broadcast; they must be discovered via an out-of-band mechanism.
For example, a "catalog" file or track that describes the broadcast.

## Track
A Track is a series of Groups identified by a unique name within a Broadcast.

A track consists of a single active Group at any moment.
When a new Group is started, the previous Group is closed and any unconsumed content may be dropped.

Each subscription is scoped to a single Track.
A subscription will always start at the latest Group and continues until either the publisher or subscriber cancels the subscription.
The publisher closes a Group when a new Group is started.

A subscriber chooses the priority of each subscription, hinting to the publisher which Track should arrive first during congestion.
This enables the most important content to arrive during network degradation while still respecting encoding dependencies.

## Group
A Group is an ordered stream of Frames within a Track.

Each group consists of a sequence number and an appendable set of Frames.
The sequence number is an increasing integer for each new group in the same Track.
Different tracks may use the same group sequence number for alignment purposes; it's up to the application to handle this.

A Group is served by a dedicated QUIC stream which is closed on completion, reset by the publisher, or cancelled by the subscriber.
The Frames within a Group will arrive reliably and in order thanks to the QUIC stream.
In contrast, Groups may temporarily arrive out of order due to network congestion and the application should be prepared to handle this.

## Frame
A Frame is a payload of bytes within a Group.

A frame is used to represent a chunk of data with a known size.
A frame should represent a single moment in time and avoid any buffering that would increase latency.


# Flow
This section outlines the flow of messages within a moq-lite session.
See the section for Messages section for the specific encoding.

## Connection
moq-lite runs on top of WebTransport.
WebTransport is a layer on top of QUIC and HTTP/3, required for web support.
The API is nearly identical to QUIC, however notably lacks stream IDs and has fewer available error codes.

How the WebTransport connection is established is out-of-scope for this draft.
For example, a service MAY use the WebTransport handshake to perform authentication via the URL.


## Termination
QUIC bidirectional streams have an independent send and receive direction.
Rather than deal with half-open states, moq-lite combines both sides.
If an endpoint closes the send direction of a stream, the peer MUST also close the send direction.

moq-lite contains many long-lived transactions, such as subscriptions and announcements.
These are terminated when the underlying QUIC stream is terminated.

To terminate a stream, an endpoint may:
- close the send direction (STREAM with FIN) to gracefully terminate (all messages are flushed).
- reset the send direction (RESET_STREAM) to immediately terminate.

After resetting the send direction, an endpoint MAY close the recv direction (STOP_SENDING).
However, it is ultimately the other peer's responsibility to close their send direction.

## Handshake
After a connection is established, the client opens a Session Stream and sends a SESSION_CLIENT message, to which the server replies with a SESSION_SERVER message.
The session is active until either endpoint closes or resets the Session Stream.

This session handshake is used to negotiate the moq-lite version and any extensions.
See the Extension section for more information.


# Streams
moq-lite uses a bidirectional stream for each transaction.
If the stream is closed, potentially with an error, the transaction is terminated.

## Bidirectional Streams
Bidirectional streams are used for control streams.
There's a 1-byte STREAM_TYPE at the beginning of each stream.

|------|------------|------------|
|     ID | Stream       | Creator      |
|-------:|:-------------|--------------|
|    0x0 | Session      | Client       |
| ------ | ------------ | ------------ |
|    0x1 | Announce     | Subscriber   |
| ------ | ------------ | ------------ |
|    0x2 | Subscribe    | Subscriber   |
| ------ | ------------ | ------------ |

### Session
There is a single Session Stream per WebTransport session.

The client MUST open a single Session Stream immediately
After establishing the QUIC/WebTransport session, the client opens a Session Stream.
There MUST be only one Session Stream per WebTransport session and its closure by either endpoint indicates the moq-lite session is closed.

The client sends a SESSION_CLIENT message indicating the supported versions and extensions.
If the server does not support any of the client's versions, it MUST close the stream with an error code and MAY close the connection.
Otherwise, the server replies with a SESSION_SERVER message to complete the handshake.

Afterwards, both endpoints SHOULD send SESSION_UPDATE messages, such as after a significant change in the session bitrate.

This draft's version is combined with the constant `0xff0dad00`.
For example, moq-lite-draft-03 is identified as `0xff0dad03`.


### Announce
A subscriber can open a Announce Stream to discover broadcasts matching a prefix.
This is OPTIONAL and the application can determine track paths out-of-band.

The subscriber creates the stream with a ANNOUNCE_PLEASE message.
The publisher replies with ANNOUNCE messages for any matching broadcasts.
Each ANNOUNCE message contains one of the following statuses:

- `active`: a matching broadcast is available.
- `ended`: a previously `active` broadcast is no longer available.

Each broadcast starts as `ended` and MUST alternate between `active` and `ended`.
The subscriber MUST reset the stream if it receives a duplicate status, such as two `active` statuses in a row or an `ended` without `active`.
When the stream is closed, the subscriber MUST assume that all broadcasts are now `ended`.

Path prefix matching and equality is done on a byte-by-byte basis.
There MAY be multiple Announce Streams, potentially containing overlapping prefixes, that get their own copy of each ANNOUNCE.

## Subscribe
A subscriber can open a Subscribe Stream to request a Track.

The subscriber MUST start a Subscribe Stream with a SUBSCRIBE message followed by any number of SUBSCRIBE_UPDATE messages.
The publisher MUST reply with an SUBSCRIBE_OK message.

The publisher SHOULD close the stream after the track has ended.
Either endpoint MAY reset/cancel the stream at any time.



## Unidirectional Streams
Unidirectional streams are used for data transmission.

|------|--------|-----------|
|     ID | Stream   | Creator     |
|-------:|:---------|-------------|
|    0x0 | Group    | Publisher   |
| ------ | -------- | ----------- |

### Group
A publisher creates Group Streams in response to a Subscribe Stream.

A Group Stream MUST start with a GROUP message and MAY be followed by any number of FRAME messages.
A Group MAY contain zero FRAME messages, potentially indicating a gap in the track.
A frame MAY contain an empty payload, potentially indicating a gap in the group.

Both the publisher and subscriber MAY reset the stream at any time.


# Encoding
This section covers the encoding of each message.

## STREAM_TYPE
All streams start with a short header indicating the stream type.

~~~
STREAM_TYPE {
  Stream Type (i)
}
~~~

The stream ID depends on if it's a bidirectional or unidirectional stream, as indicated in the Streams section.
A reciever MUST close the session if it receives an unknown stream type.

## SESSION_CLIENT
The client initiates the session by sending a SESSION_CLIENT message.

~~~
SESSION_CLIENT Message {
  Supported Versions Count (i)
  Supported Version (i)
  Extension Count (i)
  [
    Extension ID (i)
    Extension Payload (b)
  ]...
}
~~~


## SESSION_SERVER
The server responds with the selected version and any extensions.

~~~
SESSION_SERVER Message {
  Selected Version (i)
  Extension Count (i)
  [
    Extension ID (i)
    Extension Payload (b)
  ]...
}
~~~

## SESSION_UPDATE

~~~
SESSION_UPDATE Message {
  Session Bitrate (i)
}
~~~

**Session Bitrate**:
The estimated bitrate of the QUIC connection in bits per second.
This SHOULD be sourced directly from the QUIC congestion controller.
A value of 0 indicates that this information is not available.


## ANNOUNCE_PLEASE
A subscriber sends an ANNOUNCE_PLEASE message to indicate it wants to receive an ANNOUNCE message for any broadcasts with a path that starts with the requested prefix.

~~~
ANNOUNCE_PLEASE Message {
  Broadcast Path Prefix (s),
}
~~~

**Broadcast Path Prefix**:
Indicate interest for any broadcasts with a path that starts with this prefix.

The publisher MAY close the stream with an error code if the prefix is too expansive.
Otherwise, the publisher SHOULD respond with an ANNOUNCE message for any matching broadcasts.



## ANNOUNCE
A publisher sends an ANNOUNCE message to advertise a broadcast in response to an ANNOUNCE_PLEASE.
Only the suffix is encoded on the wire, the full path is constructed by prepending the requested prefix.

~~~
ANNOUNCE Message {
  Announce Status (i),
  Broadcast Path Suffix (s),
}
~~~

**Announce Status**:
A flag indicating the announce status.

- `ended` (0): A path is no longer available.
- `active` (1): A path is now available.

**Broadcast Path Suffix**:
This is combined with the broadcast path prefix to form the full broadcast path.


## SUBSCRIBE
SUBSCRIBE is sent by a subscriber to start a subscription.

~~~
SUBSCRIBE Message {
  Subscribe ID (i)
  Broadcast Path (s)
  Track Name (s)
  Subscriber Priority (i)
}
~~~

**Subscribe ID**:
A unique idenfier chosen by the subscriber.
A Subscribe ID MUST NOT be reused within the same session, even if the prior subscription has been closed.

**Subscriber Priority**:
The transmission priority of the subscription relative to all other active subscriptions within the session.
The publisher SHOULD transmit *higher* values first during congestion.
If there is a tie, a publisher MAY use the Publisher Priority as a tiebreaker.


## SUBSCRIBE_UPDATE
A subscriber can modify a subscription with a SUBSCRIBE_UPDATE message.

~~~
SUBSCRIBE_UPDATE Message {
  Subscriber Priority (i)
}
~~~

**Subscriber Priority**:
The new subscriber priority; see SUBSCRIBE.


## SUBSCRIBE_OK
The SUBSCRIBE_OK is sent in response to a SUBSCRIBE.
It contains information about the subscription

~~~
SUBSCRIBE_OK Message {
  Publisher Priority (i)
}
~~~

**Publisher Priority**:
The priority of the subscription as indicated by the publisher.
This SHOULD be used as a tiebreaker when the Subscriber Priority is the same.

**Meta**: This field isn't super useful and could have been removed, but we should encode *something* to acknowledge the subscription.

## GROUP
The GROUP message contains information about a Group, as well as a reference to the subscription being served.

~~~
GROUP Message {
  Subscribe ID (i)
  Group Sequence (i)
}
~~~

**Subscribe ID**:
The corresponding Subscribe ID.
This ID is used to distinguish between multiple subscriptions for the same track.

**Group Sequence**:
The sequence number of the group.
This SHOULD increase by 1 for each new group.


## FRAME
The FRAME message is a payload at a specific point of time.

~~~
FRAME Message {
  Payload (b)
}
~~~

**Payload**:
An application specific payload.
A generic library or relay MUST NOT inspect or modify the contents unless otherwise negotiated.



# Appendix A: Changelog
A quick comparison of moq-lite and moq-transport-10:

## Deleted Messages
- GOAWAY
- MAX_SUBSCRIBE_ID
- SUBSCRIBES_BLOCKED
- SUBSCRIBE_ERROR
- UNSUBSCRIBE
- SUBSCRIBE_DONE
- FETCH
- FETCH_OK
- FETCH_ERROR
- FETCH_CANCEL
- TRACK_STATUS_REQUEST
- TRACK_STATUS
- ANNOUNCE_OK
- ANNOUNCE_ERROR
- ANNOUNCE_CANCEL
- SUBSCRIBE_ANNOUNCES_OK
- SUBSCRIBE_ANNOUNCES_ERROR
- UNSUBSCRIBE_ANNOUNCES
- FETCH_HEADER
- OBJECT_DATAGRAM
- OBJECT_DATAGRAM_STATUS

## Deleted Fields
Some of these fields occur in multiple messages.

- Track Alias
- Group Order
- Filter Type
- StartGroup
- StartObject
- EndGroup
- Expires
- ContentExists
- Largest Group ID
- Largest Object ID
- Subscribe Parameters
- Announce Parameters
- Subgroup ID
- Object ID
- Object Status
- Extension Headers

## Misc Changes
- Messages don't have an encoded length.
- Track Namespace (renamed to Broadcast Path) is a string, not an array of bytes.
- Track Name is a string, not bytes.


# Security Considerations
TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
