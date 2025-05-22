---
title: "Media over QUIC - Hang"
abbrev: "hang"
category: info

docname: draft-lcurley-moq-hang-latest
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
  moql: I-D.lcurley-moq-lite
  moqt: I-D.ietf-moq-transport
  webcodecs: WebCodecs

informative:

--- abstract

Hang is a real-time conferencing protocol built on top of moq-lite.
A room consists of multiple participants who publish media tracks.
All updates are live, such as a change in participants or media tracks.

--- middle

# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Terminology
Hang is built on top of moq-lite [moql] and uses much of the same terminology.
A quick recap:

- **Broadcast**: A collection of Tracks from a single publisher.
- **Track**: An series of Groups, each of which can be delivered and decoded *out-of-order*.
- **Group**: An series of Frames, each of which must be delivered and decoded *in-order*.
- **Frame**: A sized payload of bytes representing a single moment in time.

Hang introduces additional terminology:

- **Room**: A collection of participants, publishing under a common prefix.
- **Participant**: A moq-lite broadcaster that may produce any number of media tracks.
- **Catalog**: A JSON document that describes each available media track, supporting live updates.
- **Container**: A tiny header in front of each media payload containing the timestamp.


# Discovery
The first requirement for a real-time conferencing application is to discover other participants in the same room.
Hang does this using moq-lite's ANNOUNCE capabilities.

A room consists of a path.
Any participants within the room MUST publish a broadcast with the room path as a prefix and it SHOULD end with the `.hang` suffix.

For example:

~~~
/room/alice.hang
/room/bob.hang
/other/zoe.hang
~~~

A participant issues an ANNOUNCE_PLEASE message to discover any other participants in the same room.
The server (relay) will then respond with an ANNOUNCE message for any matching broadcasts, including their own.

For example:

~~~
ANNOUNCE_PLEASE prefix=/room/
ANNOUNCE suffix=alice.hang active=true
ANNOUNCE suffix=bob.hang   active=true
~~~

If a publisher no longer wants to participant, or is disconnected somehow, their presence will be unannounced.
Publishers and subscribers SHOULD terminate any subscriptions once a participant is unannounced.

~~~
ANNOUNCE suffix=alice.hang active=false
~~~

# Catalog
The catalog describes the available media tracks for a single participant.
It's a JSON document that extends the the W3C WebCodecs specification.

The catalog is published as a `catalog.json` track within the broadcast so it can be updated live as the participant's media tracks change.
A participant MAY forgo publishing a catalog if it does not wish to publish any media tracks now and in the future.

The catalog track consists of multiple groups, one for each update.
Each group contains a single frame with UTF-8 JSON.
A publisher MUST NOT write multiple frames to a group until a future specification includes a delta-encoding mechanism.

## Root
The root of the catalog is a JSON document with the following schema:

~~~
type Catalog = {
	"audio": AudioTrack[],
	"video": VideoTrack[],
}
~~~

When there are multiple audio or video tracks, they SHOULD describe the same content.
For example, different resolutions, codecs, bitrates, etc.
If a participant wants to publish unrelated content, for example sharing the screen in addition to a webcam, it SHOULD publish a separate broadcast (and catalog).

Additional fields MAY be added based on the application.
The catalog SHOULD be mostly static, delegating any dynamic content to other tracks.
Additionally, a catalog SHOULD describe optional content, allowing the client to decide if it wants to subscribe.

For example, a `"chat"` field should include the name of a chat track, not individual chat messages.
This way catalog updates are rare and a client MAY choose to not subscribe.


## Video
A video track contains the necessary information to decode a video stream.

Hang uses the [VideoDecoderConfig](https://www.w3.org/TR/webcodecs/#video-decoder-config).
Any Uint8Array fields are hex-encoded into a string.

The `track` field includes the name and priority of the track within the broadcast.

~~~
type VideoTrack = {
	"track": {
		"name": string,
		"priority": number,
	},
	"config": VideoDecoderConfig,
}
~~~

For example:

~~~
{
	"track": {
		"name": "video",
		"priority": 2
	},
	"config": {
		"codec": "avc1.64001f",
		"codedWidth": 1280,
		"codedHeight": 720,
		"bitrate": 6000000,
		"framerate": 30.0
	}
}
~~~


## Audio
An audio track contains the necessary information to decode an audio stream.

The `track` field includes the name and priority of the track within the broadcast.

The `config` field contains an [AudioDecoderConfig](https://www.w3.org/TR/webcodecs/#audio-decoder-config).
Any Uint8Array fields are hex-encoded into a string.

~~~
type AudioTrack = {
	"track": {
		"name": string,
		"priority": number,
	},
	"config": AudioDecoderConfig,
}
~~~

For example:

~~~
{
	"track": {
		"name": "audio",
		"priority": 1
	},
	"config": {
		"codec": "opus",
		"sampleRate": 48000,
		"numberOfChannels": 2,
		"bitrate": 128000
	}
}
~~~

# Media
Media tracks are split into groups and further into frames.

A group consists of one or more frames in decode order.
Each group MUST start with a keyframe.
If a codec supports delta frames (video), then all subsequent frames MUST be delta frames.
Otherwise, a group MAY consist of multiple keyframes (audio).

Each "frame" consists of a tiny "container" containing the timestamp and codec specific payload.
The timestamp is the presentation timestamp in microseconds encoded as a QUIC variable-length integer (62-bit max).
The remainder of the frame payload is codec specific.

# Security Considerations
TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
