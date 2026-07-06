---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "n0 QUIC NAT Traversal"
abbrev: "n0 QNT"
category: info

docname: draft-bruynooghe-n0-quic-nat-traversal-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Transport"
workgroup: "QUIC"
keyword:
 - QUIC
 - NAT Traversal
 - hole punching

author:
 -
    fullname: Floris Bruynooghe
    organization: n0 Inc.
    email: flub@n0.computer

normative:
   QUIC-TRANSPORT: rfc9000
   QUIC-MULTIPATH: I-D.ietf-quic-multipath
   SEEMANN-QNT: I-D.draft-seemann-quic-nat-traversal
   QUIC-ADDR-DISCOVERY: I-D.draft-ietf-quic-address-discovery

informative:

...

--- abstract

A description of how [noq] performs NAT traversal for [iroh],
including all current bugs.
This is not yet a specification, though further revisions migth be.

[noq]: https://github.com/n0-computer/noq
[iroh]: https://github.com/n0-computer/iroh


--- middle

# Introduction

This document describes how [noq] currently performs NAT traversal using a QUIC extension.
It is based on {{SEEMANN-QNT}}, but with heavy modifications.
This is not expected to be the final shape of the protocol, and does not aim to be a specification.

The NAT traversal described here functions as a QUIC extension which needs to be negotiated in the transport parameters.
Once negotiated the server can share NAT traversal address candidates using ADD_ADDRESS frames.
NAT traversal is initiated by the client using REACH_OUT frames,
and performed using {{QUIC-TRANSPORT}} PATH_CHALLEGE and PATH_RESPONSE frames in probing packets to achieve NAT traversal.

The current implementation is combined with {{QUIC-ADDR-DISCOVERY}} for gathering address candidates and {{QUIC-MULTIPATH}} for opening multiple paths concurrently,
however these mechanisms are somewhat orthogonal and not not require tight integration.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# NAT Traversal

## Establishing a connection

Both endpoints are expected to be able to establish a QUIC connection already.
This is done via a "relay" path, the transport of this is out of scope.
However in iroh this looks like a normal IP path that uses IPv6 Unique Local Addresses as far as the QUIC stack is concerned.

Once the connection is established with the extension negotiated the endpoints can perform NAT traversal.


## Gathering address candidates

Both endpoints need to gather address candidates to participate in the holepunching from the same UDP socket that takes part in the holepunching.
This is done using {{QUIC-ADDR-DISCOVERY}} using a sparate connection to a cooperating server.

The discovered addresses are propagated up to the application,
which has full control over which ones it decides to use for NAT traversal by passing any desired address candidates into the QUIC stack as NAT traversal address candidates.


## Negotiating Extension use {#negotiation}

Endpoints advertise their support of the extension by sending the max_remote_nat_traversal_addresses (0x3d7f91120401) set to a non-zero variable-length integer.

The value advertised by the server is not used by the client.
It is only used by the server itself to limit the number of address candidates the application is allowed to use.

The value sent by the client limits the number of address candidates the server can advertise to the client at any given time.


## Exchanging address candidates

The server advertises address candidates using ADD_ADDRESS frames.
Any address candidates advertised by the server that exceed the allowed number the client advertised in the transport parameter,
are ignored by the client.
Each ADD_ADDRESS frame uses a unique and monotonically incrementing sequence ID.

If the server wants to remove an address candidate from the set of address candidates the client can use,
it uses the REMOVE_ADDRESS frame using a sequence ID of a previously sent ADD_ADDRESS frame.
The client ignores an unknown sequence ID in any REMOVE_ADDRESS frames.

If the server receives an ADD_ADDRESS or REMOVE_ADDRESS frame it closes the connection with the PROTOCOL_VIOLATION transport error.


## Initiating NAT traversal

The client initiates NAT traversal by sending one or more REACH_OUT frames,
each containing one of the client's address candidates.
At the same time the client sends out probes to all the address candidates the server advertised.

Upon receipt of a REACH_OUT frame the server also sends out a probe to the client's address candidate advertised in the frame.

NAT traversal happens in rounds, each round has a sequence ID.
The receipt of a higher sequence ID than the current round implies the termination of the previous round.
This means that any probes which have not yet had a response will no longer be retried.

If the server sends a REACH_OUT frame the client closes the connection with the PROTOCOL_VIOLATION transport error.


### NAT probes

NAT probes are valid QUIC packets containing a PATH_CHALLENGE frame,
sent off-path to the address candidate provided by the remote endpoint.
They are treated as in {{Section 8.2.1 of QUIC-TRANSPORT}},
except they are not padded to 1200 bytes.
As a result any valid PATH_RESPONSE received for such a challenge does not validate the path.

Such NAT probes are treated as *probing frames* as defined in {{Section 9.1 of QUIC-TRANSPORT}}.
Since this extension is used together with {{QUIC-MULTIPATH}} this re-introduces the concept of probing frames even when {{QUIC-MULTIPATH}} is negotiated.

Currently these NAT probes are sent using the active CID of the {{QUIC-MULTIPATH}} PathId they are sent on,
in clear violation of {{Section 9.5 of QUIC-TRANSPORT}}.

Since PATH_CHALLENGE frames are not retransmittable the sending endpoint has to re-send NAT probes if it does not receive a timely response.
For this the normal PTO duration calculation is used,
using the initial RTT for new paths and using no ack-delay since PATH_RESPONSES must be sent immediately.
The exponent of the exponential backoff is the retry count of the probe.
Finally NAT probes are abandoned once a new round is started, or a fixed timeout of 30s elapsed.

Once a successful response is received from a remote address probing of that remote address stops,
even if there are still other un-responded challenges to the same remote address.

In combination with {{QUIC-MULTIPATH}} an acknowledgement for the NAT probe migth be received over another path,
even if no matching PATH_RESPONSE is received.
These acknowledgements do not influence the sending of new challenges when no timely response is received.


### Probe responses

Upon receiving an off-path probing packet containing a PATH_CHALLENGE,
the receiving endpoint handles this as any other path validation or liveness probe response as specified in {{Section 8.2.2 of QUIC-TRANSPORT}}.
Since the receiving endpoint does not know the reason for this path challenge it always expands the packet to the 1200 byte limit required for path validation.

When the client receives an off-path PATH_CHALLENGE it always includes a new PATH_CHALLENGE together with the PATH_RESPONSE it must send, turning the response into a new NAT probe.
This has two benefits:

- It speeds up opening of new paths,
  since as per {{QUIC-MULTIPATH
   }} only the client can open a new path a successful NAT traversal can only be acted upon by the client.
  Sending a new challenge before the next challenge retry would occur is faster.

- It enables NAT traversal even for challenges received from addresses not advertised as address candidates by the server.
  This can occur when the server is behind an Endpoint Dependent Mapping NAT,
  while the client is behind an Endpoint Independent Mapping NAT.


### Successful NAT traversal

Once the client receives a PATH_RESPONSE for a NAT probe it sent,
NAT traversal is successful.

The client now opens a new path using {{QUIC-MULTIPATH}} over the path.
It does however stil have to validate the path before it can use it,
since only the address is validated as part of the NAT traversal.


# Frames

Receipt of any frame while the extension is not negotiated results in closing the connection with a PROTOCOL_VIOLATION transport error.

## ADD_ADDRESS

As per {{SEEMANN-QNT}}, but with different codepoints:

~~~
ADD_ADDRESS Frame {
    Type (i) = 0x3d7f90..0x3d7f91,
    Sequence Number (i),
    [ IPv4 (32) ],
    [ IPv6 (128) ],
    Port (16),
}
~~~

The ADD_ADDRESS frame contains the following fields:

Sequence Number:

: A variable-length integer encoding the sequence number of this address
   advertisement.

IPv4:

: The IPv4 address. Only present if the least significant bit of the frame type
   is 0.

IPv6:

: The IPv6 address. Only present if the least significant bit of the frame type
   is 1.

Port:

: The port number.

ADD_ADDRESS frames are ack-eliciting. When lost, they SHOULD be retransmitted,
unless the address is not active anymore.

This frame is only sent from the server to the client. Servers MUST treat
receipt of an ADD_ADDRESS frame as a connection error of type
PROTOCOL_VIOLATION.


## REMOVE_ADDRESS

As per {{SEEMANN-QNT}}, but with different codepoints:

~~~
REMOVE_ADDRESS Frame {
    Type (i) = 0x3d7f94,
    Sequence Number (i),
}
~~~

The REMOVE_ADDRESS frame contains the following fields:

Sequence Number:

: A variable-length integer encoding the sequence number of the address
   advertisement to be removed.

REMOVE_ADDRESS frames are ack-eliciting. When lost, they SHOULD be
retransmitted.

This frame is only sent from the server to the client. Servers MUST treat
receipt of an REMOVE_ADDRESS frame as a connection error of type
PROTOCOL_VIOLATION.

## REACH_OUT

~~~
REACH_OUT Frame {
    Type (i) = 0x3d7f92..0x3d7f93,
    round (i),
    [ IPv4 (32) ],
    [ IPv6 (128) ],
    Port (16),
}
~~~

The REACH_OUT frame contains the following fields:

Round:

: The sequence number of the NAT Traversal attempt.

IPv4:

: The IPv4 address. Only present if the least significant bit of the frame type
   is 0.

IPv6:

: The IPv6 address. Only present if the least significant bit of the frame type
   is 1.

Port:

: The port number.

REACH_OUT frames are ack-eliciting.

This frame is only sent from the client to the server. Clients MUST treat
receipt of a REACH_OUT frame as a connection error of type
PROTOCOL_VIOLATION.


# Security Considerations

TODO Security

- Both client and server send data to un-validated addresses provided by the peer,
  this should be subject to ani-amplification limits.


# IANA Considerations

TODO codepoints need registering


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
