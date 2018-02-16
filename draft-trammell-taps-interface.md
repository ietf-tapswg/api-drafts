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
    street: PO Box 1080 Blindern
    city: 0316  Oslo
    country: Norway
    email: michawe@ifi.uio.no
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
    street: Marchstraße 23
    city: 10587 Berlin
    country: Germany
    email: philipp@inet.tu-berlin.de
  -
    ins: T. Enghardt
    name: Theresa Enghardt
    org: TU Berlin
    street: Marchstraße 23
    city: 10587 Berlin
    country: Germany
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

informative:
  I-D.pauly-taps-transport-security:

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

The pre-establishment phase allows applications to specify parameters for
the connections they're about to make, or to query the API about potential
connections they could make.

A Preconnection object represents a potential connection. It has state that
describes parameters of a Connection that might exist in the future.  This
state comprises information about the local and remote endpoints (see
{{endpointspec}}), the transport parameters (see {{transport-params}}), and
the security parameters (see {{security-parameters}}):

~~~
   localEndpoint   := ...
   remoteEndpoint  := ...
   transportParams := ...
   securityParams  := ...

   preConnection := NewPreconnection(localEndpoint, remoteEndpoint,
                                     transportParams, securityParams);

~~~

The Local Endpoint MUST be specified if the Preconnection is used to Listen()
for incoming connections, but is OPTIONAL if it is used to Initiate()
connections. The Remote Endpoint MUST be specified in the Preconnection is used
to Initiate() connections, but is OPTIONAL if it is used to Listen() for
incoming connections.

\[NOTE: note also that framers and de-framers should be bound to the
Preconnection object during pre-establishment, forward-reference
{{send-framing}} and {{receive-framing}}]




## Specifying Endpoints {#endpointspec}

The transport services API uses Endpoint objects to refer to local and remote endpoints.
Endpoint objects can be configured using various representations of endpoint identifiers, 
including IP addresses, hostnames, or interface names as well as port
numbers and service names:

~~~
remoteSpecifier := NewEndpoint()
remoteSpecifier.withHostname("example.com")
remoteSpecifier.withService("https")
~~~

~~~
remoteSpecifier := NewEndpoint()
remoteSpecifier.withIPv6Address(2001:db8:4920:e29d:a420:7461:7073:0a)
remoteSpecifier.withPort(443)
~~~

~~~
remoteSpecifier := NewEndpoint()
remoteSpecifier.withIPv4Address(192.0.2.21)
remoteSpecifier.withPort(443)
~~~

~~~
localSpecifier := NewEndpoint()
localSpecifier.withInterface("en0")
localSpecifier.withPort(443)
~~~

Implementations may also support additional endpoint representations and
provide a single NewEndpoint() call that takes different endpoint representations.


\[TASK: match with #initiate / #listen / #rendezvous and make sure the transport
stack used is communicated ]

The transport services API resolves names internally, when Initiate() is
called to transform a Preconnection object into a Connection. The API does
not need an explicit name resolution method, although it may be useful to
provide one for reasons unrelated to transport.

\[NOTE: the API needs MUST be explicit about when name resolution occurs,
        since the act of resolving a name leaks information, and there
        may be security implications if this happens unexpectedly.]

## Specifying Transport Parameters {#transport-params}

A Preconnection object holds parameters reflecting the application's
requirements and preferences for the transport.  These include Transport
Preferences (towards protocol selection and path selection) as well as
Application Intents (hints to the transport system what to optimize for)
and Protocol Properties (to configure transport protocols).

All Transport Parameters are organized within a single name space, that is
shared with Send Parameters (see {{send-params}}). While Application
Intents and Protocol Properties take parameter specific values, Transport
Preferences use one of five fixed levels (see {{transport-prefs}}).

Some parameters express strict requirements that the application relies on,
while others are hints of what transport features would be helpful
for the application. For example, if an application asks for reliable data
transfer, choosing a transport protocol such as UDP, which does not have this
feature, will break the application's functionality. On the other hand, the
option to not require checksums when receiving an individual Content can help
optimize for low latency, but if not present, it will most likely not break the
fundamental assumptions of the application.
Moreover, there can be conflicts between parameters set by
the application: If multiple features are requested which are offered
by different protocols, it may not be possible to satisfy all requirements.
Consequently, a transport system must prioritize Transport Parameters and
consider the relevant trade-offs, see also {{?TAPS-MINSET=I-D.ietf-taps-minset}}.

Default preference levels and possible combinations of Transport Parameters and
preference levels are specified in {{appendix-preferences}}.



### Transport Preferences {#transport-prefs}

Transport Preferences drive protocol selection and path selection on connection
establishment.
Not all transport protocols work on all paths. Thus, transport protocol
selection is tied to path selection, which may involve choosing between
multiple local interfaces that are connected to different access networks.

The Transport Preferences form part of the information used to create a
Preconnection object. As such, they can be configured during the
pre-establishment phase, but cannot be changed once a Connection has been
established.

To reflect the needs of an individual connection, they can be
specified with different preference, whereby the preference is one of the
following levels:

   | Preference   | Effect                                                    |
   |--------------|-----------------------------------------------------------|
   | `require `   | Fail if requested feature/property can not be met         |
   | `prefer  `   | Proceed if requested feature/property can not be met      |
   | `don't care` | None / clear defaults                                     |
   | `avid    `   | Proceed if requested feature/property can not be avoided  |
   | `prohibit`   | Fail if requested feature/property can not be avoided     |

There need to be sensible defaults for the Transport Preferences.
There need to be sensible defaults for the Protocol Selection Properties. The
defaults given in the following section represent a configuration that can be
implemented over TCP. An alternate set of default Protocol Selection Properties
would represent a configuration that can be implemented over UDP.

The following properties apply to Connections and Connection Groups:

* Reliable Data Transfer:
  This boolean property specifies whether the application needs the
  transport protocol to ensure that data is received completely and without
  corruption on the other side. This also entails being notified when a
  Connection is closed or aborted. This property applies to connections and
  connection groups.  This is a strict requirement. The default is to
  enable Reliable Data Transfer.

* Preservation of data ordering:
  This boolean property specifies whether the application needs the
  transport protocol to assure that data is received by the application on
  the other end in the same order as it was sent. This property applies to
  connections and connection groups. This is a strict requirement. The
  default is to preserve data ordering.

* Configure reliability for individual Content:
  This boolean property specifies whether an application considers it
  useful to indicate its reliability requirements on a per-Content basis.
  This property applies to connections and connection groups. This is not a
  strict requirement.  The default is to not have this option.

* Request not to delay acknowledgment (SACK) of Content:
  This boolean property specifies whether an application considers it
  useful to request for Content that its acknowledgment be sent out as
  early as possible (SACK) instead of potentially being bundled with other
  acknowledgments. This property applies to connections and connection
  groups. This is not a strict requirement. The default is to not have this
  option.

* Use 0-RTT session establishment with idempotent Content:
  This boolean property specifies whether an application would like to
  supply a Content to the transport protocol before Connection
  establishment, which will then be reliably transferred to the other side
  before or during connection establishment, potentially multiple times.
  See also {{send-idempotent}}.  This is a strict requirement. The default
  is to not have this option.

* Use Connection Groups with priorities:
  This boolean property specifies whether an application considers it
  useful to create Connection Groups and explicitly prioritize between
  Connections within a Connection Group.

* Suggest a timeout to the peer:
  This boolean property specifies whether an application considers it
  useful to propose a timeout until the connection is assumed to be lost.
  This property applies to Connections and Connection Groups. This is not a
  strict requirement. The default is to have this option.

* Notification of special errors (excessive retransmissions, ICMP error message arrival):
  This boolean property specifies whether an application considers it
  useful to be informed in case sent data was retransmitted more often than
  a certain threshold, or when an ICMP error message arrives. This property
  applies to Connections and Connection Groups. This is not a strict
  requirement. The default is to have this option.

* Control checksum coverage on sending or receiving:
  This boolean property specifies whether the application considers it
  useful to enable / disable / configure a checksum when sending Content,
  or decide whether to require a checksum or not when receiving Content.
  This property applies to Connections and Connection Groups. This is not a
  strict requirement, as it signifies a reduction in reliability. The
  default is full checksum coverage without being able to change it, and
  requiring a checksum when receiving.


* Interface Type to prefer:
  This property specifies which kind of access network interface, e.g.,
  WiFi, Ethernet, or LTE, to prefer over others for this connection, in
  case they are available.  This is not a strict requirement. The default
  is to use the default interface configured in the system policy.

* Interface Type to prohibit:
  This property specifies which kind of access network interface, e.g.,
  WiFi, Ethernet, or LTE, to not use for this connection. This is a strict
  requirement and connection establishment will fail if no other interface
  is available. The default is to not prohibit any particular interface.



### Protocol Properties {#protocol-props}

Protocol Properties represent the configuration of a transport protocol once
it has been selected. A transport protocol may not support all Protocol
Properties, depending on the available transport features. 
As with Transport Preferences ({{transport-prefs}}), Protocol Properties are
specified on the Preconnection object, and are using during initiation of a
Connection to help the system choose an appropriate transport.
The system will only actually set those protocol properties that are actually 
supported by the chosen transport protocol. 
These properties all apply to Connections and Connection groups.
The default settings of these properties depends on the chosen protocol and on
the system configuration.

* Set timeout for aborting Connection:
  This numeric property specifies how long to wait before aborting a
  Connection attempt.  It is given in seconds.

* Set timeout to suggest to the peer:
  This numeric property specifies the timeout to propose to the peer. It is
  given in seconds.

* Set retransmissions before "Excessive Retransmissions":
  This numeric property specifies after how many retransmissions to inform
  the application about "Excessive Retransmissions".

* Set required minimum coverage of the checksum for receiving:
  This numeric property specifies the part of the received data that needs
  to be covered by a checksum. It is given in Bytes. A value of 0 means
  that no checksum is required, and a special value (e.g., -1) indicates
  full checksum coverage.

* Set scheduler for connections in a group:
  This property specifies which scheduler should be used among Connections
  within a Connection Group. It applies to connection groups. For now we
  suggest we the schedulers defined in {{I-D.ietf-tsvwg-sctp-ndata}}.

* Maximum Content Size Before Connection Establishment:
  This numeric property represents the maximum Content size that can be sent
  before or during Connection establishment, see also {{send-idempotent}}.
  It is given in Bytes.

### Application Intents {#intents}

Application Intents are a group of properties expressing what an application wants
to achieve, knows, assumes or prefers regarding its communication. They are not
strict requirements. In particular, they should not be used to express any
Quality of Service expectations that an application might have. Instead, an
application should express its intentions and its expected traffic
characteristics in order to help the transport system make decisions that best match
it, but on a best-effort basis. Even though Application Intents do not represent
Quality of Service requirements, a transport system may use them to determine a DSCP
value, e.g. similar to Table 1 in {{I-D.ietf-tsvwg-rtcweb-qos}}.

Application Intents can influence protocol selection, protocol configuration, path
selection, and endpoint selection. For example, setting the "Timeliness" Intent
to "Interactive" may lead the transport system to disable the Nagle algorithm for a
connection, while setting the "Timeliness" to "Background" may lead it to
setting the DSCP value to "scavenger". If the "Size to be Sent" Intent is set
on individual Content, it may influence path selection, e.g., when the TAPS
system schedules big Content over an interface with higher bandwidth, and
small Content over an interface with lower latency.

Specifying Application Intents is not mandatory. An application can specify any
combination of Application Intents. 
If specified, Application Intents are defined as parameters passed to the
Preconnection object, and may influence the Connection established from
that Preconnection.
If a Connection is cloned to form a Connection Group, and associated
Application Intents are cloned along with the other transport parameters.
Some Intents can also be specified for individual Content, similar to the
properties in {{send-params}}.


#### Traffic Category

This Intent specifies what the application expect the dominating traffic
pattern to be.

Possible Category values are:

Query:
: Single request / response style workload, latency bound

Control:
: Long lasting low bandwidth control channel, not bandwidth bound

Stream:
: Stream of bytes/Content with steady data rate

Bulk:
: Bulk transfer of large Content, presumably bandwidth bound

The default is to not assume any particular traffic pattern. Most categories
suggest the use of other intents to further describe the traffic pattern
anticipated, e.g., the bulk category suggesting the use of the Message Size
intents or the stream category suggesting the Stream Bitrate and Duration
intents.


#### Size to be Sent / Received

This Intent specifies what the application expects the size of a transfer to be.
It is a numeric property and given in Bytes. This Intent can also apply to
individual Content.


Duration

This Intent specifies what the application expects the lifetime of a transfer
to be. It is a numeric property and given in milliseconds.


Stream Bit-rate Sent / Received

This Intent specifies what the application expects the bit-rate of a transfer to
be. It is a numeric property and given in Bytes per second.


#### Timeliness

This Intent specifies what delay characteristics the applications prefers. It
provides hints for the transport system whether to optimize for low latency or other
criteria. Note that setting this Intents does not imply any guarantees on
whether an application's requirements can actually be satisfied. This Intent
can also apply to individual Content.

Stream:
: Delay and packet delay variation should be kept as low as possible

Interactive:
: Delay should be kept as low as possible, but some variation is tolerable

Transfer:
: Delay and packet delay variation should be reasonable, but are not critical

Background:
: Delay and packet delay variation is no concern

The default is "Transfer".


#### Disruption Resilience

This Intent describes what an application knows about its own ability to deal
with disruption of its communication, e.g., connection loss. It provides hints
of how well an application assumes it can recover from such disturbances and
can have an impact on the trade-off between providing failover techniques and
resource utilization. This Intent can also apply to individual Content.

Sensitive:
: Disruptions result in application failure, disrupting user experience

Recoverable:
: Disruptions are inconvenient for the application, but can be recovered from

Resilient:
: Disruptions have minimal impact for the application

The default is "Sensitive".


#### Cost Preferences

This Intent describes what an application prefers regarding monetary costs,
e.g., whether it considers it acceptable to utilize limited data volume. It
provides hints to the transport system on how to handle trade-offs between cost
and performance or reliability. This Intent can also apply to individual
Content.

No Expense:
: Avoid transports associated with monetary cost

Optimize Cost:
: Prefer inexpensive transports and accept service degradation

Balance Cost:
: Use system policy to balance cost and other criteria

Ignore Cost:
: Ignore cost, choose transport solely based on other criteria

The default is "Balance Cost".



### Transport Parameters Object

All transport parameters used in the pre-establishment phase are collected
in a TransportParameters object that is passed to the Preconnection object.
For the sake of convenience, we suggests implementing TransportParameters
class as sub-class of an existing Dictionary or Set class to allow in-place 
notion of Transport Parameters.

~~~
transportParameters := NewTransportParameters()
~~~

The Individual parameters are then added to the TransportParameters object.
While Protocol Properties and Application Intents use the `add` call,
Transport Preferences use special calls for the levels defined in {{transport-params}}.

~~~
transportParameters.add(intent, value)

transportParameters.add(parameter, value)

transportParameters.require(preference)
transportParameters.prefer(preference)
transportParameters.dontcare(preference)
transportParameters.avid(preference)
transportParameters.prohibit(preference)
~~~

For an existing connection, the Transport Parameters can be queried any time
by using the following call on the Connection object:

~~~
transportParameters := connection.getTransportParameters()
~~~

\[Note that most properties are only considered for connection establishment
and can not be changed later on. {{appendix-specify-query-params}} gives an
overview of what Transport Parameters can be specified and queried during which
phase. ]

\[Note: We need to more clearly separate out parameters that can be changed
once a connection has been established from those that cannot. (csp)]

Connections can be cloned at any time, before or after establishment.
A cloned connection and its parent are entangled: they share the same
TransportParameters object, changing any parameter for one of them also
changes the parameter for the other, connecting one of them also connects
the other, etc.
Cloning connections during pre-establishment is encouraged, as it
informs the transport system about the intent to form Connection Groups.

\[Note that priority assignment ((see also {{groups}} for more details) is
not shared among cloned connections. Therefore, the priority assignment
MUST NOT be realized using the connection level TransportParameters object.]



## Specifying Security Parameters and Callbacks {#security-parameters}

Common parameters such as TLS ciphersuites are known to implementations. Clients SHOULD
use common safe defaults for these values whenever possible. However, as discussed in
{{I-D.pauly-taps-transport-security}}, many transport security protocols require specific
security parameters and constraints from the client at the time of configuration and
actively during a handshake. These configuration parameters are created as follows

~~~
securityParameters := NewSecurityParameters()
~~~

Security configuration parameters and sample usage follow:

- Local identity and private keys: Used to perform private key operations and prove one's
identity to remote peers. (Note, if private keys are not available, e.g., since they are
stored in HSMs, handshake callbacks MUST be used. See below for details.)

~~~
securityParameters.AddIdentity(identity)
securityParameters.AddPrivateKey(privateKey, publicKey)
~~~

- Supported algorithms: Used to restrict what parameters are used by underlying transport security protocols.
When not specified, these algorithms SHOULD default to known and safe defaults for the system. Parameters include:
ciphersuites, supported groups, and signature algorithms.

~~~
securityParameters.AddSupportedGroup(22)    // secp256k1
securityParameters.AddCiphersuite(0xCCA9)   // TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
securityParameters.AddSignatureAlgorithm(7) // ed25519
~~~

- Session cache: Used to tune cache capacity, lifetime, re-use,
and eviction policies, e.g., LRU or FIFO.

~~~
securityParameters.SetSessionCacheCapacity(1024)     // 1024 elements
securityParameters.SetSessionCacheLifetime(24*60*60) // 24 hours
securityParameters.SetSessionCacheReuse(1)           // One-time use
~~~

- Pre-shared keying material: Used to install pre-shared keying material established
out-of-band. Each pre-shared keying material is associated with some identity that typically identifies
its use or has some protocol-specific meaning to peers.

~~~
securityParameters.AddPreSharedKey(key, identity)
~~~

Security decisions, especially pertaining to trust, are not static. Thus, once configured,
parameters must also be supplied during live handshakes. These are best handled as
client-provided callbacks. Security handshake callbacks include:

- Trust verification callback: Invoked when a peer's trust must be validated before the
handshake protocol can proceed.

~~~
trustCallback := NewCallback({
  // Handle trust, return the result
})
securityParameters.SetTrustVerificationCallback(trustCallback)
~~~

- Identity challenge callback: Invoked when a private key operation is required, e.g., when
local authentication is requested by a remote.

~~~
challengeCallback := NewCallback({
  // Handle challenge
})
securityParameters.SetIdentityChallengeCallback(challengeCallback)
~~~

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

Connection := Preconnection.Initiate()

Before calling Initiate, the caller must have populated a Preconnection
object with local and remote endpoint specifiers, as well as all parameters
necessary for candidate selection. After calling Initiate, no further
parameters may be bound to the Connection. The Initiate() call consumes
the Preconnection and creates a Connection object. A Preconnection can
only be initiated once.

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

Preconnection.Listen()

Before calling Listen, the caller must have initialized the Preconnection
during the pre-establishment phase with local endpoint specifiers, as well
as all parameters necessary for Protocol Stack selection. 
The Listen() action consumes the Preconnection. Once Listen() has been
called, no further parameters may be bound to the Preconnection, and no
subsequent establishment call may be made on the Preconnection.

Preconnection -> ConnectionReceived&lt;Connection>

The ConnectionReceived event occurs when a remote endpoint has established a
transport-layer connection to this Preconnection or when the remote endpoint has
sent its first Content, causing a new Connection to be
created. The resulting Connection is contained within the ConnectionReceived
event, and is ready to use as soon as it is passed to the application via the
event.

\[MICHAEL: JFYI, just to explain why I added "or when the remote endpoint has
sent its first Content" above: in case the connection is in fact a
stream, nothing may happen on the wire when doing Connect, and the
first thing the listener gets may already be the first data block.]

Preconnection -> ListenError&lt;>

A ListenError occurs either when the set of local specifier, transport and
cryptographic parameters cannot be fulfilled for listening, when the local
specifier cannot be resolved, or when the application is prohibited from
listening by the operating system.

## Peer to Peer Establishment: Rendezvous {#rendezvous}

Preconnection.Rendezvous()

Preconnection -> Ready&lt;>

Preconnection -> RendezvousError&lt;>

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
is sent by passing a Content object and additional parameters
{{send-params}} to the Send action on an established connection:

Connection.Send(Content, SentParameters)

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


## Send Parameters {#send-params}

The Send action takes per-Content send parameters which control how the
contents will be sent down to the underlying Protocol Stack and transmitted.

If Send Parameters should be overridden for a specific content, an
empty sent parameter Object can be acquired and all desired Send Parameters
can be added to that object. A Send Parameters object can be reused for
sending multiple contents with the same properties.

~~~
sentParameters := NewSentParameters()
sentParameters.add(parameter, value)
~~~

The Send Parameters are organized in *Content Properties*,
*Protocol Properties*, and *Application Intents*.
The Send Parameters share a single namespace with the Transport Parameters (see
{{transport-params}}). This allows to specify Protocol Properties and that can
be overridden on a per content basis or Application Intents that apply to a
specific content.
See {{appendix-specify-query-params}} for an overview.

\[Note that some of these parameters are not compatible with transport
parameters; attempting to Send with such an incompatibility yields a SendError.]

### Content Properties

#### Lifetime {#send-lifetime}

Lifetime specifies how long a particular Content can wait to be sent to the
remote endpoint before it is irrelevant and no longer needs to be
(re-)transmitted. When a Content's Lifetime is infinite, it must be
transmitted reliably. The type and units of Lifetime are
implementation-specific.

#### Niceness {#send-niceness}

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

#### Ordered {#send-ordered}

Ordered is a boolean property. If true, this Content should be delivered after
the last Content passed to the same Connection via the Send action; if false,
this Content may be delivered out of order.

#### Immediate {#send-immediate}

Immediate is a boolean property. If true, the caller prefers immediacy to
efficient capacity usage for this Content. For example, this means that
the Content should not be bundled with other
Content into the same transmission by the underlying Protocol Stack.

#### Idempotent {#send-idempotent}

Idempotent is a boolean property. If true, the application-layer entity in the
Content is safe to send to the remote endpoint more than once for a single
Send action. It is used to mark data safe for certain 0-RTT establishment
techniques, where retransmission of the 0-RTT data may cause the remote
application to receive the Content multiple times.

\[NOTE: we need some way to signal to the transport that we want to wait for
0RTT data on Initiate. Probably a transport parameter]
\[MICHAEL: why? As an app programmer, I can just use Send instead of Initiate.]

#### Checksum {#send-checksum}

This numeric property specifies the length of the checksum to be used on the Content.
A value of 0 means that no checksum is required, and a special value (e.g. -1) can
be used to indicate full checksum coverage (which is also the default). Only
full coverage is guaranteed, any other requests are advisory.

###Protocol Properties

\[NOTE: this will be mostly protocol specific stuff as urgent data for TCP - maybe also move Checksum here]

###Application Intents

#### Size to be Received

On a bi-directional connection, this Intent specifies what amount of data the
application expects to receive in reply to the content sent.
It is a numeric property and given in Bytes.

#### Stream Bitrate Sent

This Intent specifies what bitrate the application wishes the content to be
sent with. This is useful as input for any transport that does packet pacing.
It is a numeric property and given in Bytes per second.

#### Stream Bitrate Received

On a bi-directional connection, this Intent specifies what bitrate the
application expects to receive in reply to the content sent.
It is a numeric property and given in Bytes per second.

#### Cost Preferences

This Intent describes what an application prefers regarding monetary costs,
e.g., whether it considers it acceptable to utilize limited data volume. It
provides hints to the transport system on how to handle tradeoffs between cost
and performance or reliability. This Intent can also apply to individual
Content.

No Expense:
: Avoid transports associated with monetary cost

Optimize Cost:
: Prefer inexpensive transports and accept service degradation

Balance Cost:
: Use system policy to balance cost and other criteria

Ignore Cost:
: Ignore cost, choose transport solely based on other criteria

The default is "Balance Cost".


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
protocols in the Protocol Stack for logging and debugging purposes. In particular,
when this information is available, the value of the Explicit Congestion Notification
(ECN) field is contained in such metadata.

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

# Maintenance

The application can access the following properties on an established Connection:

* Transport Features of the protocols that were selected. These features correspond to the properties given in {{transport-params}} and can only be queried.
* Protocol Properties of the protocols in use. These properties correspond to the properties given {{protocol-props}} and can be set and queried.
* Path Properties of the path(s) in use. These properties can be derived from the local provisioning domain, measurements by the protocol stack, or other sources. They can only be queried.

~~~
Properties := Connection.getProperties()
~~~

{{appendix-specify-query-params}} gives a more detailed overview of the different types of properties that can be set and queried at different times.

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

This document describes a generic API for interacting with a transport services (TAPS) system.
Part of this API includes configuration details for transport security protocols, as discussed
in Section {{security-parameters}}. It does not recommend use (or disuse) of specific
algorithms or protocols. Any API-compatible transport security protocol should work in a TAPS system.

# Acknowledgements

This work has received funding from the European Union's Horizon 2020 research and
innovation programme under grant agreements No. 644334 (NEAT) and No. 688421 (MAMI).

--- back

# Sample API definition in Go {#appendix-api-sketch}

This document defines an abstract interface. To illustrate how this would map concretely into a programming language, this appendix contains an API interface definition in Go, using callbacks for event handling. The documentation for the API sketch is available online at https://godoc.org/github.com/mami-project/postsocket.

~~~~~~~
package postsocket

import (
	"crypto/tls"
	"io"
	"net"
	"time"
)

type TransportContext interface {
	NewTransportParameters() TransportParameters
	NewSecurityParameters() SecurityParameters
	NewRemote() Remote
	NewLocal() Local
	DefaultSendParameters() SendParameters

	SetEventHandler(evh EventHandler)
	SetFramingHandler(fh FramingHandler)

  Preconnect(evh EventHandler, fh FramingHandler,
             rem Remote, loc Local,
             tp TransportParameters,
             sp SecurityParameters) (Preconnection, error)

  Initiate(rem Remote, loc Local,
           tp TransportParameters,
           sp SecurityParameters) (Connection, error)
  Rendezvous(rem Remote, loc Local,
             tp TransportParameters,
             sp SecurityParameters) (Connection, error)
  Listen(loc Local,
         tp TransportParameters,
         sp SecurityParameters) (Connection, error)
}

type Remote interface {
	WithHostname(hostname string) Remote
	WithAddress(address net.IP) Remote
	WithPort(port uint16) Remote
	WithServiceName(svc string) Remote
}

type Local interface {
	WithInterface(iface string) Local
	WithHostname(hostname string) Local
	WithAddress(address net.IP) Local
	WithPort(port uint16) Local
	WithServiceName(svc string) Local
}

type ParameterIdentifier int

const (
	TransportFullyReliable = iota
	// ... and so on
)

type TransportParameters interface {
	Require(p ParameterIdentifier, v int) TransportParameters
	Prefer(p ParameterIdentifier, v int) TransportParameters
	Avoid(p ParameterIdentifier, v int) TransportParameters
	Prohibit(p ParameterIdentifier, v int) TransportParameters
}

type SecurityParameters interface {
	AddIdentity(c tls.Certificate) SecurityParameters
	AddPSK(c tls.Certificate, k []byte) SecurityParameters
	VerifyTrustWith(func(c tls.Certificate) (bool, error)) SecurityParameters
	HandleChallengeWith(func() (bool, error)) SecurityParameters
	Require(p ParameterIdentifier, v interface{} SecurityParameters
	Prefer(p ParameterIdentifier, v interface{}) SecurityParameters
	Avoid(p ParameterIdentifier, v interface{}) SecurityParameters
	Prohibit(p ParameterIdentifier, v interface{}) SecurityParameters
}

type SendParameters struct {
	Lifetime time.Duration
	Niceness uint
	Ordered bool
	Immediate bool
	Idempotent bool
	CorruptionTolerant bool
}

type Preconnection interface {
  AddSpecifier(rem Remote, loc Local,
               tp TransportParameters, sp SecurityParameters)
	Initiate() (Connection, error)
	InitialSend(content interface{}, sp SendParameters) (Connection, error)
	Rendezvous() (Connection, error)
	Listen() (Connection, error)
}

type Connection interface {
	Send(content interface{}, contentref interface{}, sp SendParameters) error

	Clone() (Connection, error)
	Close() error

	GetEventHandler() EventHandler
	SetEventHandler(evh EventHandler)

	GetFramingHandler() FramingHandler
	SetFramingHandler(fh FramingHandler)
}

type Content interface {
	Bytes() []byte
}

// EventHandler defines the interface for connection event handlers.
type EventHandler interface {
	Ready(conn Connection, ante Connection)
	Received(content Content, conn Connection)
	Sent(conn Connection, contentref interface{})
	Expired(conn Connection, contentref interface{})
	Error(conn Connection, contentref interface{}, err error)
	Closed(conn Connection, err error)
}

type FramingHandler interface {
	Frame(content interface{}) ([]byte, error)
	Deframe(in io.Reader) (Content, error)
}
~~~~~~~

# Transport Parameters {#appendix-transport-params}

This appendix provides details about the usage of the Transport Parameters
specified in {{transport-params}}. It clarifies what preference levels an
application can set for which Transport Parameter, and during which phase an
application can specify and query what kinds of Transport Parameters.

## Application Preferences {#appendix-preferences}

As described in {{transport-params}}, an application can specify its preference
regarding a Transport Parameter, i.e., whether a certain property is required,
preferred, to be avoided, prohibited, or an intention. If an application does
not set its preference regarding a Transport Parameter, default preference
levels apply as specified in the following table. A default preference of
"None" means that the transport system assumes that an application does not
have any preference regarding the corresponding Transport Parameter and may not
take this parameter into account for protocol and path selection.

Not every Transport Parameter can be meaningfully assigned every preference
level. For example, if an application explicitly prohibits selecting a
transport protocol that allows to suggest a timeout to the peer, this
restriction will unnecessarily limit transport protocol selection. Instead,
the application could simply not use this feature if it is present in the
selected transport protocol.

The following table illustrates which Transport Parameter has which default
preference level and which alternative preference levels an application may
set.

| Transport Parameter    | Require  | Prefer | Avoid | Prohibit | Default |
|------------------------|----------|--------|-------|----------|---------|
| Reliable Data Transfer | Yes      | Yes    | Yes   | Yes      | Require |
| Preserve Data Ordering | Yes      | Yes    | Yes   | No       | Require |
| Configure Reliability per Content | Yes | Yes | Yes | Yes     | None    |
| Request SACK           | Yes      | Yes    | Yes    | Yes     | None    |
| Use 0-RTT with Idempotent Content | Yes | Yes | Yes | Yes     | None    |
| Use Connection Groups with priorities | Yes | Yes | No | No   | None    |
| Suggest timeout to peer | Yes     | Yes    | No    | No       | None    |
| Notification of special errors | Yes | Yes | Yes   | Yes      | None    |
| Control checksum coverage | Yes   | Yes    | Yes   | Yes      | None    |
| Use a certain network interface type | Yes | Yes | Yes | Yes  | None    |
| Application Intents    | No       | No     | No    | No       | Intend  |

\[List individual Intents? Reformulate some of them as preferences?]

## Specifying and Querying Parameters {#appendix-specify-query-params}

In this appendix we give an overview of the different types of properties, the
objects to which they apply, and at what time an application can query them.

During the Pre-Establishment phase, an application may specify Transport
Parameters for a Connection as described in {{transport-params}}.
Specifically, Protocol Selection Properties, Path Selection Properties, and
Application Intents for the Connection MUST be specified during
Pre-Establishment, as protocol and path selection occur during connection
establishment. An application may query the Transport Parameters that were
specified for a Connection at all times.

Transport Features represent the actual capabilities of specific Protocol
Stacks. They are expressed in the same vocabulary as Protocol Selection
Properties, but have a Boolean value expressing whether a Protocol Stack
support the given Transport Feature or not. An application may query the
Transport Features of Protocol Stacks at all times. Once a Connection is
established, an application may query the Transport Features of the actually
chosen protocols, the Protocol Stack Instances, for this Connection.

Note that it is possible that the Protocol Stack Instances actually chosen by
the transport system do not fully reflect the Transport Parameters that were
originally set. For example, a certain Protocol Selection Property that an
application specified as Preferred may not actually be present in the chosen
Protocol Stack Instances because none of the currently available transport
protocols had this feature.

Protocol Stacks and Protocol Stack Instances also have Protocol Properties,
which represent the specific configuration of a transport protocol. Their
default values can be queried at all times, and an application can override
these defaults for a specific Connection by specifying Protocol Properties
either during pre-establishment or later in the lifetime of a Connection. This
configuration is applied on the Protocol Stack Instance once it is bound to the
Connection.

Note that some Protocol Properties set during Pre-Establishment may not apply
to the actually chosen protocol later, and consequently not be set in the
resulting Protocol Stack Instance. However, it is beneficial for an application
to set these properties as early as possible, so the transport system can use
them to optimize.

Finally, an application may query the properties of the available paths and the
properties of the path(s) chosen for a Connection at all times.

An application may also specify Send Properties per individual Content, as
specified in {{send-params}}.

The following table shows the types of existing properties and what an
application can do with them during what phase:

| Property Type                  | Applies to | Pre-Establishment | Established |
|--------------------------------|------------|-------------------|-------------|
| Transport Parameters           | Connection | Set, Query        | Query       |
| Protocol Features              | ProtocolStack, ProtocolStackInstance | Query | Query |
| Protocol Property defaults     | ProtocolStack | Query          | Query       |
| Protocol Properties            | ProtocolStackInstance | Set    | Set, Query  |
| Path Properties                | Path       | Query             | Query       |
| Send Properties                | Content    | Set               | Set         |
