---
title: An Abstract Application Layer Interface to Transport Services
abbrev: TAPS Interface
docname: draft-trammell-taps-interface-latest
date:
category: info

ipr: trust200902
area: Transport
workgroup: TAPS Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: B. Trammell
    name: Brian Trammell
    role: editor
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: M. Welzl
    name: Michael Welzl
    role: editor
    org: University of Oslo
  -
    ins: G. Fairhurst
    name: Gorry Fairhurst
    org: University of Aberdeen
    email: gorry@erg.abdn.ac.uk
  -
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    org: ETH Zurich
    email: mirja.kuehlewind@tik.ee.ethz.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: C. Perkins
    name: Colin Perkins
    org: University of Glasgow
    street: School of Computing Science
    city: Glasgow  G12 8QQ
    country: United Kingdom
    email: csp@csperkins.org
  -
    ins: P. Tiesel
    name: Philipp S. Tiesel
    org: TU Berlin
    email: philipp@inet.tu-berlin.de
  -
    ins: T. Enghardt
    name: Theresa Enghardt
    org: TU Berlin
    email: theresa@inet.tu-berlin.de
  -
    ins: C. Wood
    name: Chris Wood
    org: Apple Inc.
    street: 1 Infinite Loop
    city: Cupertino, California 95014
    country: United States of America
    email: cawood@apple.com

normative:
  I-D.ietf-tsvwg-sctp-ndata:
  I-D.ietf-tsvwg-rtcweb-qos:
  TAPS-ARCH:
    title: An Architecture for Transport Services
    docname: draft-pauly-taps-arch-00
    author:
      -
        ins: T. Pauly
        role: editor
      -
        ins: B. Trammell
        role: editor
      -
        ins: A. Brunstrom
      -
        ins: G. Fairhurst
      -
        ins: C. Perkins
      -
        ins: P. Tiesel
      -
        ins: C. Wood

--- abstract

This document describes an abstract programming interface to the transport
layer, following the Transport Services Architecture. It supports the
asynchronous, atomic transmission of messages over transport protocols and
network paths dynamically selected at runtime. It is intended to replace the
traditional BSD sockets API as the lowest common denominator interface to the
transport layer, in an environment where endpoints have multiple interfaces
and potential transport protocols to select from.

--- middle

# Introduction

The BSD Unix Sockets API's SOCK_STREAM abstraction, by bringing network
sockets into the UNIX programming model, allowing anyone who knew how to write
programs that dealt with sequential-access files to also write network
applications, was a revolution in simplicity. It would not be an overstatement
to say that this simple API is the reason the Internet won the protocol wars
of the 1980s. SOCK_STREAM is tied to the Transmission Control Protocol (TCP),
specified in 1981 {{?RFC0793}}. TCP has scaled remarkably well over the past
three and a half decades, but its total ubiquity has hidden an uncomfortable
fact: the network is not really a file, and stream abstractions are too
simplistic for many modern application programming models.

In the meantime, the nature of Internet access, and the variety of Internet
transport protocols, is evolving. The challenges that new protocols and access
paradigms present to the sockets API and to programming models based on them
inspire the design principles of a new approach, which we outline in {{principles}}.

As a first step to realizing this design, {{TAPS-ARCH}}
describes a high-level architecture for transport services. This document
builds a modern abstract programming interface atop this architecture,
deriving specific path and protocol selection properties and supported
transport features from the analysis provided in {{?RFC8095}} and
{{?TAPS-MINSET=I-D.ietf-taps-minset}}.

# Terminology and Notation

This API is described in terms of Objects, which an application can interact
with; Actions the application can perform on these objects; Events, which an
object can send to an application asynchronously; and Parameters associated
with these Actions and Events.

The following notations, which can be combined, are used in this document:

- Object := Action()

>> An Action creates an Object.

- Object.Action()

>> An Action is performed on an Object.

- Object -> Event&lt;>

>> An Object sends an Event.

- Action(parameter, parameter, ...) / Event&lt;parameter, parameter, ...>

>> An Action takes a set of Parameters; an Event contains a set of Parameters.

Actions associated with no object are Actions on the abstract interface
itself; they are equivalent to actions on a per-application global context.

How these abstract concepts map into concrete implementations of this API in a
given language on a given platform is largely dependent on the features of the
language and the platform. Actions could be implemented as functions or method
calls, for instance, and Events could be implemented via callback
passing or other asynchronous calling conventions.

# Design Principles {#principles}

We begin with a set of initial design principles for the abstract interface to
realize.

- Transport protocol stack independence in line with the Transport Services
  Architecture {{TAPS-ARCH}}, allowing applications to be written in terms of
  the semantics best for the application's own design, separate from the
  protocol(s) used on the wire to achieve them. This enables applications
  written to a single API to make use of transport protocols in terms of the
  features they provide.

- Explicit support for multistreaming and multipath transport protocols.

- Explicit support for security properties as first-order transport features,
  and for long-term caching of cryptographic identities and parameters for
  associations among endpoints.

- Atomic transmission of content, using application-assisted framing and
  deframing where the underlying transport does not provide these.

- Asynchronous connection establishment, transmission, and reception, allowing
  most application interactions with the transport layer to be event-driven.

# API Summary

\[TASK: write three paragraph summary here, should state how all this works for common cases from the application's PoV.]

In the following sections, we describe the details of application interaction with Objects through Actions and Events in each phase of a connection, following the phases described in {{TAPS-ARCH}}.

# Pre-Establishment Phase

Establishment begins with the creation of a Connection...

Connection := NewConnection(localSpecifier, remoteSpecifier, transportParameters, cryptographicParameters)

\[NOTE: note also that framers and deframers should be bound to connections
during pre-establishment, forward-reference {{send-framing}} and
{{receive-framing}}]

## Resolving Remote Endpoints {#resolving}

Name resolution is no explicit step of a transport service API.
Therefore, name resolution may be deferred until connection establishment
to incorporate transport parameters.
Instead, a remoteSpecifier object representing the remote endpoint is created
providing an appropriate endpoint representation, which include IP addresses,
hostnames and URLs:

remoteSpecifier := Endpoint()
remoteSpecifier.withUrl("https://example.com")

remoteSpecifier := Endpoint()
remoteSpecifier.withHostname("example.com"
remoteSpecifier.withService("https")

remoteSpecifier := Endpoint()
remoteSpecifier.withIPv6Address(2001:db8:4920:e29d:a420:7461:7073:0a)
remoteSpecifier.withPort(443)

remoteSpecifier := Endpoint()
remoteSpecifier.withIPv4Address(192.0.2.21)
remoteSpecifier.withPort(443)

Implementations may also support additional endpoint representations and
provide a single Endpoint() call that takes different endpoint representations.

Endpoint representations may imply transport protocols, pseudotransport protocols,
or families of protocols, e.g., remoteSpecifier.withUrl("https://example.com") 
implies either using HTTP over TLS over TCP or using HTTP over QUIC over UDP.
Whether the protocols implied by the endpoint representation are provided by the
transport system is implementation specific, but MUST BE tunable using 
pre-establishment properties.
Implementations SHOULD provide all parts of the implied transport stack they
implement unless specified otherwise using pre-establishment properties.
For example, the transport system may provide TLS over TCP in the above example
and let the application implement HTTP pseudo-transport itself.

\[TASK: match with #initiate / #listen / #rendezvous and make sure the transport
stack used is communicated ]
   
   
## Specifying Transport Parameters {#transport-params}

When creating a connection, an application needs to specify transport
parameters reflecting its requirements and preferences regarding its
communication. These Transport parameters include Protocol Selection
Properties, Protocol Properties (configuration of a transport protocol once it
has been selected), Path Selection Properties, and Socket Intents (hints to the
TAPS system what to optimize for).

Some Protocol Selection Properties are strict requirements that the application
relies on, while others are hints of what transport features would be helpful
for the application. For example, if an application asks for reliable data
transfer, choosing a transport protocol such as UDP, which does not have this
feature, will break the application's functionality. On the other hand, the
option to disable checksums when sending an individual message can help
optimize for low latency, but if not present, it will most likely not break the
fundamental assumptions of the application.

Moreover, there can be conflicts between properties set by the application: If
multiple features are requested which are offered by different protocols, it
may not be possible to satisfy all requirements. Consequently, a TAPS system
must prioritize transport parameters and consider the relevant trade-offs, see
also {{?TAPS-MINSET=I-D.ietf-taps-minset}}.

\[Should it be possible for an application to specify which properties are most
important to it, or to set properties as required?]

There need to be sensible defaults for the Protocol Selection Properties. The
defaults given in the following section represent a configuration that can be
implemented over TCP. An alternate set of default Protocol Selection Properties
would represent a configuration that can be implemented over UDP.

Note that some parameters can also be set later in the lifetime of a
connection. However, Protocol Selection Properties and Path Selection
Properties must be specified before Initiate() to be useful. For other
properties, it is beneficial for the application to set them as early as
possible in order to help the TAPS system optimize.

Connections can be cloned at any time, before or after establishment.
A cloned connection and its parent are entangled: they share the same properties,
changing any parameter for one of them also changes the parameter for the other,
connecting one of them also connects the other, etc. There is only one exception:
priority assignment ((see also {{groups}} for more details).
Cloning connections during pre-establishment is encouraged, as it
informs the transport system about the intent to use Connection Groups.

Connection := Create(ProtocolSelectionProperties, ProtocolProperties,
PathSelectionProperties, SocketIntents)

Connection.Configure(ProtocolProperties, SocketIntents)

Connection.QueryProperties()

Connection := Connection.Clone()


### Protocol Selection Properties {#protocol-selection-props}

Reliable Data Transfer

This boolean property specifies whether the application needs the transport
protocol to ensure that data is received completely and without corruption on
the other side. This property applies to connections and connection groups.
This is a strict requirement. The default is to enable Reliable Data Transfer.

Notification of peer closing/aborting

This boolean property specifies whether the application needs the transport
protocol to inform it in case the connection terminates. This property applies
to connections and connection groups. This is a strict requirement. The default
is to enable such notifications.

Preservation of data ordering

This boolean property specifies whether the application needs the transport
protocol to assure that data is received by the application on the other end in
the same order as it was sent. This property applies to connections and
connection groups. This is a strict requirement. The default is to preserve data
ordering.

Option to configure reliability for individual messages

This boolean property specifies whether an application considers it useful to
indicate its reliability requirements on a per-message basis. This property
applies to connections and connection groups. This is not a strict requirement.
The default is to not have this option.

Option of unordered message delivery

This boolean property specifies whether an application allows messages to be
delivered in a different order than they were sent, potentially at the benefit
of lower latency for individual messages. This property applies to connections
and connection groups. This is not a strict requirement. The
default is to not have this option, so all data arrives strictly in-order.

Option to request not to delay acknowledgement (SACK) of a message

This boolean property specifies whether an application considers it useful to
request for a message that its acknowledgement be sent out as early as possible
(SACK) instead of potentially being bundled with other acknowledgements. This
property applies to connections and connection groups. This is not a strict
requirement. The default is to not have this option.

Option to hand over a message to reliably transfer (possibly multiple times)
before connection establishment (0-RTT idempotent)

This boolean property specifies whether an application would like to supply a
message to the transport protocol before connection establishment, which will
then be reliably transferred to the other side before or during connection
establishment, potentially multiple times. See also {{send-idempotent}}. This
property applies to connections and connection groups. This is a strict
requirement. The default is to not have this option.

Option to assign priority or weight per connection in a group

This boolean property specifies whether an application considers it useful to
have the option to explicitly prioritize between multiple connections within a
connection group. This property applies to connection groups. This is not a
strict requirement. The default is to not have this option.

Option to choose a scheduler between connections in a group

This boolean property specifies whether an application considers it useful to
have the option to explicitly set a scheduler between the connections within a
connection group. This property applies to connection groups. This is not a
strict requirement. The default is to not have this option.

Suggest a timeout to the peer

This boolean property specifies whether an application considers it useful to
propose a timeout until the connection is assumed to be lost. This
property applies to connections and connection groups. This is not a strict
requirement. The default is to have this option.

Notification of excessive retransmissions (early warning before abortion threshold)

This boolean property specifies whether an application considers it useful to
be informed in case sent data was retransmitted more often than a certain
threshold. This property applies to connections and connection groups. This is
not a strict requirement. The default is to have this option.

Notification of ICMP error message arrival

This boolean property specifies whether an application considers it useful to
be informed in case an ICMP error message is received related to this
connection. This property applies to connections and connection groups. This is
not a strict requirement. The default is to have this option.

Specify checksum coverage used by sender

This boolean property specifies whether the application considers it useful to
have the option to specify what parts of the transmitted data should be covered
by the transport protocol checksum. This property applies to connections and
connection groups. This is not a strict requirement, as it signifies a
reduction in reliability. The default is full checksum coverage without being
able to change it.

Option to disable checksum when sending an individual message

This boolean property specifies whether the application considers it useful to
have the option to disable checksums for individual messages. This property
applies to connections and connection groups. This is not a strict requirement,
as it signifies a reduction in reliability. The default is full checksum
coverage without the option to disable it.

Specify minimum checksum coverage required by receiver

This boolean property specifies whether the application considers it useful to
have the option to specify what parts of the received data it needs to be
covered by the checksum. This property applies to connections and connection
groups. This is not a strict requirement, as it signifies a reduction in
reliability. The default is full checksum coverage without being able to change
it.

Option to disable checksum requirement when receiving an individual message

This boolean property specifies whether an application considers it useful to
have the option to communicate to the remote endpoint that it does not require
an individual message to be covered by a checksum. This property applies to
connections and connection groups. This is not a strict requirement, as it
signifies a reduction in reliability. The default is full checksum coverage
without the option to disable it.

### Protocol Properties {#protocol-props}

Protocol Properties represent the configuration of a transport protocols once
it has been selected. A transport protocol may not support all Protocol
Properties, depending on the available transport features. An application
should specify the Protocol Properties as early as possible to help the TAPS
system optimize. However, a TAPS system will only actually set those protocol
properties that are actually supported by the chosen transport protocol. Some
of these properties can also be set on individual messages, similar to the
properties in {{send-props}}.

The default settings of these properties depends on the chosen protocol and on
the system configuration.

Set timeout for aborting connection

This numeric property specifies how long to wait before aborting a connection.
It is given in seconds. This property applies to connections and connection
groups.

Set timeout to suggest to the peer

This numeric property specifies the timeout to propose to the peer. It is given
in seconds. This property applies to connections and connection groups.

Set retransmissions before "Excessive Retransmissions"

This numeric property specifies after how many retransmissions to inform the
application about "Excessive Retransmissions". It applies to connections and
connection groups.

Set whether to use a checksum for sending

This boolean property specifies whether to use a checksum for sending messages
on this connection. The default is to use checksums. This property applies to
connections, connection groups, and messages.

Set length of send checksum

This numeric property specifies the length of the checksum of messages sent on
this connection. It is given in Bytes. This property applies to connections,
connection groups, and messages.

Set whether to require a checksum for receiving

This boolean property specifies whether to communicate to the remote endpoint
that it does not require messages on this connection to be covered by a
checksum. This property applies to connections, connection groups, and
messages.

Set required minimum coverage of the checksum

This numeric property specifies the part of the received data that needs to be
covered by a checksum. It is given in Bytes. This property applies to
connections, connection groups, and messages.

Set scheduler for connections in a group

This property specifies which scheduler should be used among connections within
a connection group. It applies to connection groups. For now we suggest we
suggest the schedulers defined in {{I-D.ietf-tsvwg-sctp-ndata}}.

Configure the priority of a connection for the scheduler

This property specifies what priority to assign to a connection within a
connection group. It applies to a connection. We suggest the priority as
described in {{I-D.ietf-tsvwg-sctp-ndata}}.

Set low watermark for buffer to nofity the application

This numeric property specifies the buffer threshold below which the
application wants to be informed. It is given in Bytes. This property applies
to connections and connection group.

Maximum Message Size Before Connection Establishment

This numeric property can be queried by the application after creating a
connection. It represents the maximum message size that can be sent before or
during connection establishment, see also {{send-idempotent}}. It is given in
Bytes.


### Path Selection Properties {#path-selection-props}

Not all transport protocols work on all paths. Thus, transport protocol
selection is tied to path selection, which may involve choosing between
multiple local interfaces that are connected to different access networks.

Path Selection Properties are requirements, prohibitions, or preferences, that
an application has regarding path selection. These properties should be
specified as early as possible in order to help the TAPS system optimize.
However, they may also be specified later.

Interface Type to prefer

This property specifies which kind of access network interface, e.g., WiFi,
Ethernet, or LTE, to prefer over others for this connection, in case they are
available.  This is not a strict requirement. The default is to use the default
interface configured in the system policy.

Interface Type to prohibit

This property specifies which kind of access network interface, e.g., WiFi,
Ethternet, or LTE, to not use for this connection. This is a strict requirement
and connection establishment will fail if no other interface is available. The
default is to not prohibit any particular interface.


### Socket Intents {#socket-intents}

Socket Intents are a group of properties expressing what an application wants
to achieve, knows, assumes or prefers regarding its communication. They are not
strict requirements. In particular, they should not be used to express any
Quality of Service expectations that an application might have. Instead, an
application should express its intentions and its expected traffic
characteristics in order to help the TAPS system make decisions that best match
it, but on a best-effort basis. Even though Socket Intents do not represent
Quality of Service requirements, a TAPS system may use them to determine a DSCP
value, e.g. similar to Table 1 in {{I-D.ietf-tsvwg-rtcweb-qos}}.

Socket Intents can influence protocol selection, protocol configuration, path
selection, and endpoint selection. For example, setting the "Timeliness" Intent
to "Interactive" may lead the TAPS system to disable the Nagle algorithm for a
connection, while setting the "Timeliness" to "Background" may lead it to
setting the DSCP value to "scavenger". If the "Size to be Sent" Intent is set
on a series of messages, it may influence path selection, e.g., when the TAPS
system schedules big messages over an interface with higher bandwidth, and
small messages over an interface with lower latency.

Specifying Socket Intents is not mandatory. An application can specify any
combination of Socket Intents. All Socket Intents can be specified for
connections and connection groups. Some Intents can also be specified for
individual messages, similar to the properties in {{send-props}}.


Traffic Category

This Intent specifies what the application expect the dominating traffic
pattern to be. It applies to connections and connection groups

Possible Category values are:

Query:
: Single request / response style workload, latency bound

Control:
: Long lasting low bandwidth control channel, not bandwidth bound

Stream:
: Stream of bytes/messages with steady data rate

Bulk:
: Bulk transfer of large messages, presumably bandwidth bound

The default is to not assume any particular traffic pattern. Most categories
suggest the use of other intents to further describe the traffic pattern
anticipated, e.g., the bulk category suggesting the use of the Message Size
intents or the stream category suggesting the Stream Bitrate and Duration
intents.


Size to be Sent / Received

This Intent specifies what the application expects the size of a transfer to be.
It is a numeric property and given in Bytes. It can also apply to individual
messages.


Duration

This Intent specifies what the application expects the lifetime of a transfer
to be. It is a numeric property and given in milliseconds. It applies to
connections and connection groups.


Stream Bitrate Sent / Received

This Intent specifies what the application expects the bitrate of a transfer to
be. It is a numeric property and given in Bytes per second. It applies to
connections and connection groups.


Burstiness

This Intent specifies what the application expects the sender-side burst
characteristics of the traffic to be. The application can provide hints about
the anticipated communication pattern, i.e., how it expects the number of sent
bytes to vary over time and the expected length of sequences of consecutively
sent messages. Note that the actual burst characteristics will depend on the
network, especially at the receiver side. This Intents applies to connections
and connection groups.

Possible Burstiness values are:

No Bursts:
: Application sends traffic at a constant rate

Regular Bursts:
: Application sends bursts of traffic periodically

Random Bursts:
: Application sends bursts of traffic irregularly

Bulk:
: Application sends a bulk of traffic

The default is to not assume any particular burst characteristics.


Timeliness

This Intent specifies what delay characteristcs the applications prefers. It
provides hints for the TAPS system whether to optimize for low latency or other
criteria. Note that setting this Intents does not imply any guarantees on
whether an application's requirements can actually be satisfied. This Intents
applies to connections, connection groups, or messages.

Stream:
: Delay and packet delay variation should be kept as low as possible

Interactive:
: Delay should be kept as low as possible, but some variation is tolerable

Transfer:
: Delay and packet delay variation should be reasonable, but are not critical

Background:
: Delay and packet delay variation is no concern

The default is "Transfer".


Disruption Resilience

This Intent describes what an application knows about its own ability to deal with disruption of its communication, e.g., connection loss. It provides hints of how well an application assumes it can recover from such disturbances and can have an impact on the tradeoff between providing failover techniques and resource utilization. This Intent applies to connections, connection groups, and messages.

Sensitive:
: Disruptions result in application failure, disrupting user experience

Recoverable:
: Disruptions are inconvenient for the application, but can be recovered from

Resilient:
: Disruptions have minimal impact for the application

The default is "Sensitive".


Cost Preferences

This Intent describes what an application prefers regarding monetary costs, e.g., whether it considers it acceptable to utilize limited data volume. It provides hints to the TAPS system on how to handle tradeoffs between cost and performance or reliability.
This Intent applies to connections, connection groups, and messages.

No Expense:
: Avoid transports associated with monetary cost

Optimize Cost:
: Prefer inexpensive transports and accept service degradation

Balance Cost:
: Use system policy to balance cost and other criteria

Ignore Cost:
: Ignore cost, choose transport solely based on other criteria

The default is "Balance Cost".


## Specifying Cryptographic Parameters {#crypto-params}

\[TASK: write me. separate out cryptographic parameters, since these bind to a
local and a remote. chris?]

# Establishing Connections

Before a Connection can be used for data transfer, it must be established.
Establishment ends the pre-establishment phase; all transport and
cryptographic parameter specification must be complete before establishment,
as these parameters will be used to select candidate Paths and Protocol Stacks
for the Connection. Establishment may be active, using the Initiate() Action;
passive, using the Listen() Action; or simultaneous for peer-to-peer, using
the Rendezvous() Action. These Actions are described in the subsections below.

## Active Open: Initiate {#initiate}

Active open is the action of establishing a connection to an endpoint presumed
to be listening for incoming connection requests, commonly used by clients in
client-server interactions. Active open is supported by this interface through
the Initiate action:

Connection.Initiate()

Before calling Initiate, the caller must have initialized the Connection
during the pre-establishment phase with local and remote endpoint specifiers,
as well as all parameters necessary for candidate selection. After calling
Initiate, no further parameters may be bound to the Connection, and no
subsequent establishment call may be made on the Connection.

Once Initiate is called, the candidate Protocol Stack(s) may cause one or more
transport-layer connections to be created to the specified remote endpoint.
The caller may immediately begin sending Content on the Connection (see
{{sending}}) after calling Initate, though it may wait for one of the
following events before doing so.

Connection -> Ready&lt;>

The Ready event occurs after Initiate has established a transport-layer
connection on at least one usable candidate Protocol Stack over at least one
candidate Path. No Receive events (see {{receiving}}) will occur until after
the Ready event for connections established using Initiate.
\[MICHAEL: This is a difficult read. Can we phrase this as "...will occur before the Ready event for connections...",
or did you have a specific reason to write "until after the Ready event"?]

Connection -> InitiateError&lt;>

An InitiateError occurs either when the set of local and remote specifiers and
transport and cryptographic parameters cannot be fulfilled on a connection for
initiation (e.g. the set of available Paths and/or Protocol Stacks meeting the
constraints is empty), when the remote specifier cannot be resolved, or when
no transport-layer connection can be established to the remote endpoint (e.g.
because the remote endpoint is not accepting connections, or the application
is prohibited from opening a connection by the operating system).

## Passive Open: Listen {#listen}

Passive open is the action of waiting for connections from remote endpoints,
commonly used by servers in client-server interactions. Passive open is
supported by this interface through the Listen action:

Connection.Listen()

Before calling Listen, the caller must have initialized the Connection
during the pre-establishment phase with local endpoint specifiers,
as well as all parameters necessary for Protocol Stack selection. After calling
Listen, no further parameters may be bound to the Connection, and no subsequent
establishment call may be made on the Connection.

Connection -> ConnectionReceived&lt;Connection>

The ConnectionReceived event occurs when a remote endpoint has established a
transport-layer connection to this Connection or when the remote endpoint has
sent its first Content, causing a new Connection to be
created. The resulting Connection is contained within the ConnectionReceived
event, and is ready to use as soon as it is passed to the application via the
event.

\[MICHAEL: JFYI, just to explain why I added "or when the remote endpoint has
sent its first Content" above: in case the connection is in fact a
stream, nothing may happen on the wire when doing Connect, and the
first thing the listener gets may already be the first data block.]

Connection -> ListenError&lt;>

A ListenError occurs either when the set of local specifier, transport and
cryptographic parameters cannot be fulfilled for listening, when the local
specifier cannot be resolved, or when the application is prohibited from
listening by the operating system.

## Peer to Peer Establishment: Rendezvous {#rendezvous}

Connection.Rendezvous()

Connection -> Ready&lt;>

Connection -> RendezvousError&lt;>

## Connection Groups {#groups}

Groups of Connections can be created using Clone action:

Connection := Connection.Clone()

Calling this once yields a group of two Connections: the parent Connection -- whose
Clone action was called -- and the resulting clone. Calling Clone on any of these two
Connections adds a third Connection to the group, and so on.
All Connections in a group are entangled. This means that they automatically share
all properties: changing a parameter for one of them also changes the parameter
for all others, closing one of them also closes all others, etc.

There is only one Protocol Property that is not entangled, i.e. it is a separate
per-Connection Property for individual Connections in the group: a priority.
This priority, which can be represented as a non-negative integer or float, expresses
a desired share of the Connection Group's available network capacity, such that an
ideal transport system implementation would assign the Connection the capacity
share P x C/sum_P, where P = priority, C = total available capacity and sum_P = sum
of all priority values that are used for the Connections in the same Connection Group.
The priority setting is purely advisory; no guarantees are given.
 
Connection Groups should be created (i.e., the Clone action should be used)
as early as possible, ideally already during the Pre-Establishment phase, in order
to aid the Transport System in choosing and configuring the right protocols
(see also {{transport-params}}).

# Sending Data {#sending}

Once a Connection has been established, it can be used for sending data. Data
is sent by passing a Content object and additional properties
{{send-props}} to the Send action on an established connection:

Connection.Send(Content, ...)

The type of the Content to be passed is dependent on the implementation, and
on the constraints on the Protocol Stacks implied by the Connection's
transport parameters. It may itself contain an array of octets to be
transmitted in the transport protocol payload, or be transformable to an array
of octets by a sender-side framer (see {{send-framing}}).

If Send is called on a Connection which has not yet been established, an
Initiate action will be implicitly performed simultaneously with the Send.
Used together with the Idempotent property (see {{send-idempotent}}), this can
be used to send data during establishment for 0-RTT session resumption on
Protocol Stacks that support it.

Like all Actions in this interface, the Send action is asynchronous.

Connection -> Sent&lt;contentRef>

The Sent event occurs when a previous Send action has completed, i.e. when the
data derived from the Content has been passed down or through the underlying
Protocol Stack and is no longer the responsibility of the implementation of
this interface. The exact disposition of Content when the Sent event occurs is
specific to the implementation and the constraints on the Protocol Stacks
implied by the Connection's transport parameters. The Sent event contains an
implementation-specific reference to the Content to which it applies.

Sent events allow an application to obtain an understanding of the amount
of buffering it creates. That is, if an application calls the Send action multiple
times without waiting for a Sent event, it has created more buffer inside the
transport system than an application that only issues a Send after this event fires.


Connection -> Expired&lt;contentRef>

The Expired event occurs when a previous Send action expired before
completion; i.e. when the data derived from the Content was not sent before
its Lifetime (see {{send-lifetime}}) expired. This is separate from SendError,
as it is an expected behavior for partially reliable transports. The Expired
event contains an implementation-specific reference to the Content to which it
applies.

Connection -> SendError&lt;contentRef>

A SendError occurs when Content could not be sent due to an error condition:
some failure of the underlying Protocol Stack, or a set of send parameters not
consistent with the Connection's transport parameters.

## Send Properties {#send-props}

The Send action takes five per-Content properties which control how it will be
sent down to the underlying Protocol Stack and transmitted. Note that some of
these properties are not compatible with transport parameters; attempting to
Send with such an incompatibility yields a SendError.

### Lifetime {#send-lifetime}

Lifetime specifies how long a particular Content can wait to be sent to the
remote endpoint before it is irrelevant and no longer needs to be
(re-)transmitted. When a Content's Lifetime is infinite, it must be
transmitted reliably. The type and units of Lifetime are
implementation-specific.

### Niceness {#send-niceness}

Niceness represents an unbounded hierarchy of priorities of Content, relative
to other Content sent over the same Connection and/or Connection Group (see
{{groups}}). It is most naturally represented as a non-negative integer.
Content with Niceness 0 will yield to Content with Niceness 1, which will
yield to Content with Niceness 2, and so on. Niceness may be used as a
sender-side scheduling construct only, or be used to specify priorities on the
wire for Protocol Stacks supporting prioritization.

Note that this inversion of normal schemes for expressing priority has a
convenient property: priority increases as both Niceness and Lifetime
decrease.

### Ordered {#send-ordered}

Ordered is a boolean property. If true, this Content should be delivered after
the last Content passed to the same Connection via the Send action; if false,
this Content may be delivered out of order.

### Immediate {#send-immediate}

Immediate is a boolean property. If true, the caller prefers immediacy to
efficient capacity usage for this Content. For example, this means that
the Content should not be bundled with other
Content into the same transmission by the underlying Protocol Stack.

### Idempotent {#send-idempotent}

Idempotent is a boolean property. If true, the application-layer entity in the
Content is safe to send to the remote endpoint more than once for a single
Send action. It is used to mark data safe for certain 0-RTT establishment
techniques, where retransmission of the 0-RTT data may cause the remote
application to receive the Content multiple times.

\[NOTE: we need some way to signal to the transport that we want to wait for
0RTT data on Initiate. Probably a transport parameter]
\[MICHAEL: why? As an app programmer, I can just use Send instead of Initiate.]

## Sender-side Framing {#send-framing}

Sender-side framing allows a caller to provide the interface with a function
that takes Content of an appropriate type and returns an array of octets, the
on-the-wire representation of the content to be handed down to the Protocol
Stack. It consists of a Framer object with a single Action, Frame. Since the
Framer depends on the protocol used at the application layer, it is bound to
the Connection during the pre-establishment phase:

Connection.FrameWith(Framer)

OctetArray := Framer.Frame(Content)

Sender-side framing is a convenience feature of the interface, for parity with
receiver-side framing (see {{receive-framing}}).

# Receiving Data {#receiving}

Once a Connection is established, Content may be received on it. The interface
notifies the application that content has been received via the Received
event:

Connection -> Received&lt;Content>

As with sending, the type of the Content to be passed is dependent on the
implementation, and on the constraints on the Protocol Stacks implied by the
Connection's transport parameters. The Content may also contain metadata from
protocols in the Protocol Stack for logging and debugging purposes.

The Content object must provide some method to retrieve an octet array
containing application data, corresponding to a single message within the
underlying Protocol Stack's framing.  See {{receive-framing}} for handling
framing in situations where the Protocol Stack provides octet-stream transport
only.

Connection -> ReceiveError&lt;>

A ReceiveError occurs when data is received by the underlying Protocol Stack
that cannot be fully retrieved or deframed, or when some other indication is
received that reception has failed. Such conditions that irrevocably lead the
the termination of the Connection are signaled using ConnectionError instead
(see {{termination}}).

## Application-Layer Backpressure at the Receiver {#receive-backpressure}

Implementations of this interface must provide some way for the application to
indicate that it is temporarily not ready to receive new Content. Since the
mechanisms of event handling are implementation-platform specific, this
document does not specify the exact nature of this interface.

## Receiver-side Deframing over Stream Protocols {#receive-framing}

The Receive event is intended to be fired once per application-layer Content
sent by the remote endpoint; i.e., it is a desired property of this interface
that a Send at one end of a Connection maps to exactly one Receive on the
other end. This is possible with Protocol Stacks that provide
message boundary preservation, but is not the case over Protocol Stacks that
provide a simple octet stream transport.

For preserving message boundaries over stream transports, this interface
provides receiver-side deframing. This facility is based on the observation
that, since many of our current application protocols evolved over TCP, which
does not provide message boundary preservation, and since many of these protocols
require message boundaries to function, each application layer protocol has
defined its own framing. A Deframer allows an application to push this
deframing down into the interface, in order to transform an octet stream into
a sequence of Content.

Concretely, receiver-side deframing allows a caller to provide the interface
with a function that takes an octet stream, as provided by the underlying
Protocol Stack, reads and returns a sigle Content of an appropriate type for
the application and platform, and leaves the octet stream at the start of the
next Content. It consists of a Deframer object with a single Action, Deframe.
Since the Deframer depends on the protocol used at the application layer, it
is bound to the Connection during the pre-establishment phase:

Connection.DeframeWith(Deframer)

Content := Deframer.Deframe(OctetStream, ...)

# Connection Termination {#termination}

Close terminates a Connection after satisfying all the requirements that were specified regarding the delivery of Content that the application has already given to the transport system. For example, if reliable delivery was requested for Content handed over before calling Close, the transport system will ensure that such Content is indeed delivered. If the peer still has data to send, it cannot be received after this call.

Connection.Close()

This event can (i.e., this is not guaranteed to happen) inform the application that the peer has closed the Connection:

Connection -> Finished&lt;>

Abort terminates a Connection without delivering remaining data:

Connection.Abort()

This event can (i.e., this is not guaranteed to happen) inform the application that the other side has aborted the Connection:

Connection -> ConnectionError&lt;>

# Event and Error Handling

\[NOTE: point out that events and errors may be handled differently, although
they are the modeled the same in this specification.]

# IANA Considerations

RFC-EDITOR: Please remove this section before publication.

This document has no actions for IANA.

# Security Considerations

be paranoid
