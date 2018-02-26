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
    ins: T. Enghardt
    name: Theresa Enghardt
    org: TU Berlin
    street: Marchstraße 23
    city: 10587 Berlin
    country: Germany
    email: theresa@inet.tu-berlin.de
  -
    ins: G. Fairhurst
    name: Godred Fairhurst
    org: University of Aberdeen
    street: Department of Engineering
    street: Fraser Noble Building
    city: Aberdeen, AB24 3UE
    country: Scotland
    email: gorry@erg.abdn.ac.uk
    uri: http://www.erg.abdn.ac.uk/
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
    ins: C. Wood
    name: Chris Wood
    org: Apple Inc.
    street: 1 Infinite Loop
    city: Cupertino, California 95014
    country: United States of America
    email: cawood@apple.com

normative:
  I-D.ietf-taps-minset:
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
{{I-D.ietf-taps-minset}}.

# Terminology and Notation

This API is described in terms of Objects, which an application can interact
with; Actions the application can perform on these objects; Events, which an
object can send to an application asynchronously; and Parameters associated
with these Actions and Events.

The following notations, which can be combined, are used in this document:

~~~
Object := Action()
~~~

>> An Action creates an Object.

~~~
Object.Action()
~~~

>> An Action is performed on an Object.

~~~
Object -> Event<>
~~~

>> An Object sends an Event.

~~~
Action(parameter, parameter, ...) / Event<parameter, parameter, ...>
~~~

>> An Action takes a set of Parameters; an Event contains a set of Parameters.

Actions associated with no object are Actions on the abstract interface
itself; they are equivalent to actions on a per-application global context.

How these abstract concepts map into concrete implementations of this API in a
given language on a given platform is largely dependent on the features of the
language and the platform. Actions could be implemented as functions or method
calls, for instance, and Events could be implemented via callback passing or
other asynchronous calling conventions. The method for registering callbacks
and handlers is left as an implementation detail, with the caveat that the
interface for receiving Messages must require the application to invoke the
Connection.Receive() action once per Message to be received (see
{{receiving}}).

This specification treats events and errors similarly, as errors, just as any
other events, may occur asynchronously in network applications. However, it is
recommended that implementations of this interface also return errors
immediately, according to the error handling idioms of the implementation
platform, for errors which can be immediately detected, such as inconsistency
in transport parameters.

# Interface Design Principles {#principles}

We begin with the architectural design principles defined in {{TAPS-ARCH}};
from these, we derive and elaborate a set of principles on which the design of
the interface is based. The interface defined in this document provides:

- A single interface to a variety of transport protocols to be
  used in a variety of application design patterns, independent of the
  properties of the application and the protocol stacks that will be used at
  runtime, such that  all common specialized features of these protocol
  stacks are made available to the application as necessary in a
  transport-independent way, to enable applications written to a single API
  to make use of transport protocols in terms of the features they provide;

- Explicit support for security properties as first-order
  transport features, and for long-term caching of cryptographic identities and
  parameters for associations among endpoints;

- Asynchronous connection establishment,
  transmission, and reception, allowing most application interactions with the
  transport layer to be event-driven, in line with developments in modern
  platforms and programming languages;

- Explicit support for multistreaming and multipath transport protocols, and
  the grouping of related connections into connection groups through cloning
  of connections, to allow applications to take full advantage of new
  transport protocols supporting these features; and

- Atomic transmission of data, using application-assisted framing and deframing
  where the underlying transport does not provide these.


# API Summary

\[EDITOR'S NOTE: write three paragraph summary here; see
https://github.com/taps-api/drafts/issues/93]

In the following sections, we describe the details of application interaction with Objects through Actions and Events in each phase of a connection, following the phases described in {{TAPS-ARCH}}.

# Pre-Establishment Phase

The pre-establishment phase allows applications to specify parameters for
the connections they're about to make, or to query the API about potential
connections they could make.

A Preconnection object represents a potential connection. It has state that
describes parameters of a Connection that might exist in the future.  This
state comprises Local Endpoint and Remote Endpoint objects that denote the
endpoints of the potential connection (see {{endpointspec}}), the transport
parameters (see {{transport-params}}), and the security parameters (see
{{security-parameters}}):

~~~
   preConnection := NewPreconnection(LocalEndpoint,
                                     RemoteEndpoint,
                                     TransportParams,
                                     SecurityParams);

~~~

The Local Endpoint MUST be specified if the Preconnection is used to Listen()
for incoming connections, but is OPTIONAL if it is used to Initiate()
connections. The Remote Endpoint MUST be specified in the Preconnection is used
to Initiate() connections, but is OPTIONAL if it is used to Listen() for
incoming connections.

Framers (see {{send-framing}}) and deframers (see {{receive-framing}}), if
necessary, should be bound to the Preconnection during pre-establishment.

## Specifying Endpoints {#endpointspec}

The transport services API uses the Local Endpoint and Remote Endpoint types
to refer to the endpoints of a transport connection.
Subtypes of these represent various different types of endpoint identifiers,
such as IP addresses, DNS names, and interface names, as well as port numbers
and service names.

~~~
remoteSpecifier := NewRemoteEndpoint()
remoteSpecifier.withHostname("example.com")
remoteSpecifier.withService("https")
~~~

~~~
remoteSpecifier := NewRemoteEndpoint()
remoteSpecifier.withIPv6Address(2001:db8:4920:e29d:a420:7461:7073:0a)
remoteSpecifier.withPort(443)
~~~

~~~
remoteSpecifier := NewRemoteEndpoint()
remoteSpecifier.withIPv4Address(192.0.2.21)
remoteSpecifier.withPort(443)
~~~

~~~
localSpecifier := NewLocalEndpoint()
localSpecifier.withInterface("en0")
localSpecifier.withPort(443)
~~~

~~~
localSpecifier := NewLocalEndpoint()
localSpecifier.withStunServer(address, port, credentials)
~~~

Implementations may also support additional endpoint representations and
provide a single NewEndpoint() call that takes different endpoint representations.

Multiple endpoint identifiers can be specified for each Local Endpoint
and RemoteEndoint.  For example, a Local Endpoint could be configured with
two interface names, or a Remote Endpoint could be specified via both IPv4
and IPv6 addresses.  The multiple identifiers refer to the same endpoint.

The transport services API will resolve names internally, when the Initiate(),
Listen(), or Rendezvous() method is called establish a connection.
The API does not need the application to resolve names, and premature name
resolution can damage performance by limiting the scope for alternate path
discovery during connection establishment.
The Resolve() method is, however, provided to resolve a Local Endpoint or a
Remote Endpoint in cases where this is required, for example with some NAT
traversal protocols (see {{rendezvous}}).

\[NOTE: the API needs MUST be explicit about when name resolution occurs,
        since the act of resolving a name leaks information, and there
        may be security implications if this happens unexpectedly.]

## Specifying Transport Parameters {#transport-params}

A Preconnection object holds parameters reflecting the application's
requirements and preferences for the transport. These include protocol and path
selection parameters, as well as Generic and Specific Protocol Properties for
configuration of the detailed operation of the selected Protocol Stacks.

All Transport Parameters are organized within a single namespace shared with
Send Parameters (see {{send-params}}). All transport parameters take
paremeter-specific values. Protocol and Path Selection properties additionally
take one of five preference levels, though not all preference levels make sense
with all such properties. Note that it is possible for a set of specified transport
parameters to be internally inconsistent, or for preferences to be inconsistent
with the later use of the API by the application. Application developers should
reduce inconsistency by only using the most stringent preference levels when
failure to meet a preference would break the application's functionality (e.g.
the Reliable Data Transfer preference, which is a core assumption of many
application protocols). Implementations of this interface should also raise
errors in configuration as early as possible, to help ensure these
inconsistencies are caught early in the development process.

The protocol(s) and path(s) selected as candidates during connection
establishment are determined by a set of properties. Since there could be
paths over which some transport protocols are unable to operate, or remote
endpoints that support only specific network addresses or transports,
transport protocol selection is necessarily tied to path selection. This may
involve choosing between multiple local interfaces that are connected to
different access networks.

To reflect the needs of an individual connection, they can be
specified with five different preference levels:

   | Preference | Effect                                                             |
   |------------|--------------------------------------------------------------------|
   | `require ` | Select only protocols/paths providing the property, fail otherwise |
   | `prefer  ` | Prefer protocols/paths providing the property, proceed otherwise |
   | `ignore  ` | Cancel any default preference for this property |
   | `avoid   ` | Prefer protocols/paths not providing the property, proceed otherwise |
   | `prohibit` | Select only protocols/paths not providing the property, fail otherwise |

An implementation of this interface must provide sensible defaults for protocol
and path selection properties. The defaults given for each property below
represent a configuration that can be implemented over TCP. An alternate set of
default Protocol Selection Properties would represent a configuration that can
be implemented over UDP.

The following properties can be used during Protocol and Path selection:

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

* Configure reliability on a per-Message basis:
  This boolean property specifies whether an application considers it
  useful to indicate its reliability requirements on a per-Message basis.
  This property applies to connections and connection groups. This is not a
  strict requirement.  The default is to not have this option.

* Use 0-RTT session establishment with an idempotent Message:
  This boolean property specifies whether an application would like to
  supply a Message to the transport protocol before Connection
  establishment, which will then be reliably transferred to the other side
  before or during connection establishment, potentially multiple times.
  See also {{send-idempotent}}.  This is a strict requirement. The default
  is to not have this option.

* Efficient use of Connection Groups:
  This boolean property specifies whether an
  application considers it useful to create Connection Groups, e.g. to
  explicitly prioritize between Connections within a Connection Group. This
  property will tend to select multistreaming transport protocols, which can
  multiplex Connections into a Connection Group over a single flow. This is not
  a strict requirement. The default is to not have this option.

* Suggest a timeout to the Remote Endpoint:
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
  useful to enable / disable / configure a checksum when sending data,
  or decide whether to require a checksum or not when receiving data.
  This property applies to Connections and Connection Groups. This is not a
  strict requirement, as it signifies a reduction in reliability. The
  default is full checksum coverage without being able to change it, and
  requiring a checksum when receiving.

* Interface Type:
  This enumerated property specifies which kind of access network interface,
  e.g., WiFi, Ethernet, or LTE, to prefer over others for this connection, in
  case they are available. In general, Interface Types should be used only with
  the `prefer` and `prohibit` preference level. Specifically, using the
  `require` preference level for Interface Type may limit path selection in a
  way that is detrimental to connectivity. The default is to use the default
  interface configured in the system policy.

* Capacity Profile:
  This enumerated property specifies the application's expectation of the
  dominating traffic pattern for this connection. The Capacity Profile should
  only be used with the `prefer` preference level; other preference levels make
  no sense for profiles. The following values are valid for Capacity Profile:

  Default:
  : The application makes no representation about its expected
  capacity profile. No special optimizations of the tradeoff between
  delay, delay variation, and bandwidth efficiency should be made when selecting and
  configuring stacks.

  Interactive/Low Latency:
  : The application is interactive. Response time (latency) should be optimized at
  the expense of bandwidth efficiency and delay variation. This can be used by
  the system to disable the coalescing of multiple small Messages into larger
  packets (Nagle's algorithm), to prefer lower-latency paths, signal a
  preference for lower-latency, higher-loss treatment, and so on.

  Constant Rate:
  : The application expects to send/receive data at a constant rate after
  connection establishment. Delay and delay variation should be optimized at the
  expense of bandwidth efficiency.

  Scavenger/Bulk:
  : The application is not interactive. It expects to send/receive a large
  amount of data, without any urgency. This can be used to select protocol
  stacks with scavenger transmission control, to signal a preference for
  less-than-best-effort treatment, and so on.

In addition to protocol and path selection properties, the transport parameters
may also contain Generic and/or Specific Protocol Properties (see
{{protocol-props}}). These properties will be passed to the selected candidate
Protocol Stack(s) to configure them before candidate connection establishment.

### Transport Parameters Object

All transport parameters used in the pre-establishment phase are collected
in a TransportParameters object that is passed to the Preconnection object.

~~~
transportParameters := NewTransportParameters()
~~~

The Individual parameters are then added to the TransportParameters object.
While Protocol Properties and Application Intents use the `add` call,
Transport Preferences use special calls for the levels defined in {{transport-params}}.

~~~
transportParameters.add(parameter, value)

transportParameters.require(preference)
transportParameters.prefer(preference)
transportParameters.ignore(preference)
transportParameters.avoid(preference)
transportParameters.prohibit(preference)
~~~

For an existing connection, the Transport Parameters can be queried any time
by using the following call on the Connection object:

~~~
transportParameters := connection.getTransportParameters()
~~~

Note that most properties are only considered for connection establishment
and can not be changed later on. {{appendix-specify-query-params}} gives an
overview of what Transport Parameters can be specified and queried during which
phase.

A Connection gets its Transport Parameters either by being explicitly configured
via a Preconnection, or by inheriting them from an antecedent via cloning; see
{{groups}} for more.

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
identity to the Remote Endpoint. (Note, if private keys are not available, e.g., since they are
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
its use or has some protocol-specific meaning to the Remote Endpoint.

~~~
securityParameters.AddPreSharedKey(key, identity)
~~~

Security decisions, especially pertaining to trust, are not static. Thus, once configured,
parameters must also be supplied during live handshakes. These are best handled as
client-provided callbacks. Security handshake callbacks include:

- Trust verification callback: Invoked when a Remote Endpoint's trust must be validated before the
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

Like transport parameters, security parameters are inherited during cloning (see
{{groups}}).

# Establishing Connections

Before a Connection can be used for data transfer, it must be established.
Establishment ends the pre-establishment phase; all transport and
cryptographic parameter specification must be complete before establishment,
as these parameters will be used to select candidate Paths and Protocol Stacks
for the Connection. Establishment may be active, using the Initiate() Action;
passive, using the Listen() Action; or simultaneous for peer-to-peer, using
the Rendezvous() Action. These Actions are described in the subsections below.

## Active Open: Initiate {#initiate}

Active open is the action of establishing a connection to a Remote Endpoint presumed
to be listening for incoming connection requests. Active open is used by clients in
client-server interactions. Active open is supported by this interface through the
Initiate action:

~~~
Connection := Preconnection.Initiate()
~~~

Before calling Initiate, the caller must have populated a Preconnection
object with a Remote Endpoint specifier, optionally a Local Endpoint
specifier (if not specified, the system will attempt to determine a
suitable Local Endpoint), as well as all parameters
necessary for candidate selection. After calling Initiate, no further
parameters may be bound to the Connection. The Initiate() call consumes
the Preconnection and creates a Connection object. A Preconnection can
only be initiated once.

Once Initiate is called, the candidate Protocol Stack(s) may cause one or more
candidate transport-layer connections to be created to the specified remote
endpoint. The caller may immediately begin sending Messages on the Connection
(see {{sending}}) after calling Initate(); note that any idempotent data sent
while the Connection is being established may be sent multiple times or on
multiple candidates.

The following events may be sent by the Connection after Initiate() is called:

~~~
Connection -> Ready<>
~~~

The Ready event occurs after Initiate has established a transport-layer
connection on at least one usable candidate Protocol Stack over at least one
candidate Path. No Receive events (see {{receiving}}) will occur before
the Ready event for connections established using Initiate.

~~~
Connection -> InitiateError<>
~~~

An InitiateError occurs either when the set of transport and cryptographic
parameters cannot be fulfilled on a connection for initiation (e.g. the set of
available Paths and/or Protocol Stacks meeting the constraints is empty) or
reconciled with the local and/or remote endpoints; when the remote specifier
cannot be resolved; or when no transport-layer connection can be established
to the remote endpoint (e.g. because the remote endpoint is not accepting
connections, or the application is prohibited from opening a connection by the
operating system).

## Passive Open: Listen {#listen}

Passive open is the action of waiting for connections from remote endpoints,
commonly used by servers in client-server interactions. Passive open is
supported by this interface through the Listen action:

~~~
Preconnection.Listen()
~~~

Before calling Listen, the caller must have initialized the Preconnection
during the pre-establishment phase with a Local Endpoint specifier, as well
as all parameters necessary for Protocol Stack selection. A Remote Endpoint
may optionally be specified, to constrain what connections are accepted.
The Listen() action consumes the Preconnection. Once Listen() has been
called, no further parameters may be bound to the Preconnection, and no
subsequent establishment call may be made on the Preconnection.

~~~
Preconnection -> ConnectionReceived<Connection>
~~~

The ConnectionReceived event occurs when a Remote Endpoint has established a
transport-layer connection to this Preconnection (for connection-oriented
transport protocols), or when the first Message has been received from the
Remote Endpoint (for connectionless protocols), causing a new Connection to be
created. The resulting Connection is contained within the ConnectionReceived
event, and is ready to use as soon as it is passed to the application via the
event.

~~~
Preconnection -> ListenError<>
~~~

A ListenError occurs either when the Preconnection cannot be fulfilled for
listening, when the Local Endpoint (or Remote Endpoint, if specified) cannot
be resolved, or when the application is prohibited from listening by policy.

## Peer-to-Peer Establishment: Rendezvous {#rendezvous}

Simultaneous peer-to-peer connection establishment is supported by the
Rendezvous() action:

~~~
Preconnection.Rendezvous()
~~~

The Preconnection object must be specified with both a Local Endpoint and a
Remote Endpoint, and also the transport and security parameters needed for
protocol stack selection. The Rendezvous() action causes the Preconnection
to listen on the Local Endpoint for an incoming connection from the
Remote Endpoint, while simultaneously trying to establish a connection from
the Local Endpoint to the Remote Endpoint.
This corresponds to a TCP simultaneous open, for example.

The Rendezvous() action consumes the Preconnection. Once Rendezvous() has
been called, no further parameters may be bound to the Preconnection, and
no subsequent establishment call may be made on the Preconnection.

~~~
Preconnection -> RendezvousDone<Connection>
~~~

The RendezvousDone<> event occurs when a connection is established with the
Remote Endpoint. For connection-oriented transports, this occurs when the
transport-layer connection is established; for connectionless transports,
it occurs when the first Message is received from the Remote Endpoint. The
resulting Connection is contained within the RendezvousDone<> event, and is
ready to use as soon as it is passed to the application via the event.

~~~
Preconnection -> RendezvousError<msgRef, error>
~~~

An RendezvousError occurs either when the Preconnection cannot be fulfilled
for listening, when the Local Endpoint or Remote Endpoint cannot be resolved,
when no transport-layer connection can be established to the Remote Endpoint,
or when the application is prohibited from rendezvous by policy.

When using some NAT traversal protocols, e.g., ICE {{?RFC5245}}, it is
expected that the Local Endpoint will be configured with some method of
discovering NAT bindings, e.g., a STUN server. In this case, the
Local Endpoint may resolve to a mixture of local and server reflexive
addresses. The Resolve() method on the Preconnection can be used to
discover these bindings:

~~~
PreconnectionBindings := Preconnection.Resolve()
~~~

The Resolve() call returns a list of Preconnection objects, that represent
the concrete addresses, local and server reflexive, on which a Rendezvous()
for the Preconnection will listen for incoming connections. This list can
be passed to a peer via a signalling protocol, such as SIP or WebRTC, to
configure the remote.

\[NOTE: This API is sufficient for TCP-style simultaneous open, but should
  be considered experimental for ICE-like protocols.]


## Connection Groups {#groups}

Groups of Connections can be created using Clone action:

~~~
Connection := Connection.Clone()
~~~

Calling this once yields a group of two Connections: the parent Connection -- whose
Clone action was called -- and the resulting clone. Calling Clone on any of these two
Connections adds a third Connection to the group, and so on.
All Connections in a group are entangled. This means that they automatically share
all properties: changing a parameter for one of them also changes the parameter
for all others.

There is only one Protocol Property that is not entangled, i.e., it is a separate
per-Connection Property for individual Connections in the group: a priority.
This priority, which can be represented as a non-negative integer or float, expresses
a desired share of the Connection Group's available network capacity, such that an
ideal transport system implementation would assign the Connection the capacity
share P x C/sum_P, where P = priority, C = total available capacity and sum_P = sum
of all priority values that are used for the Connections in the same Connection Group.
The priority setting is purely advisory; no guarantees are given.

Connection Groups should be created by cloning as as early as possible in order
to aid the Transport System in choosing and configuring the right protocols (see
also {{transport-params}}).

# Sending Data {#sending}

Once a Connection has been established, it can be used for sending data. Data
is sent by passing a Message object and additional parameters
{{send-params}} to the Send action on an established connection:

~~~
Connection.Send(Message, sendParameters)
~~~

The type of the Message to be passed is dependent on the implementation, and
on the constraints on the Protocol Stacks implied by the Connection's
transport parameters. It may itself contain an array of octets to be
transmitted in the transport protocol payload, or be transformable to an array
of octets by a sender-side framer (see {{send-framing}}).

Messages may be arbitrarily large; however, there may be system and Protocol
Stack dependent limits on the size of a data object which can be transmitted
atomically. For that reason, the Message object passed to the Send action may
also be a partial Message, either representing the whole data object and
information about the range of bytes to send from it, or an object referring
back to the larger whole Message. The details of partial Message sending are
implementation-dependent.

If Send is called on a Connection which has not yet been established, an
Initiate action will be implicitly performed simultaneously with the Send.
Used together with the Idempotent property (see {{send-idempotent}}), this can
be used to send data during establishment for 0-RTT session resumption on
Protocol Stacks that support it.

Like all Actions in this interface, the Send action is asynchronous.

~~~
Connection -> Sent<msgRef>
~~~

The Sent event occurs when a previous Send action has completed, i.e., when the
data derived from the Message has been passed down or through the underlying
Protocol Stack and is no longer the responsibility of the implementation of
this interface. The exact disposition of the Message when the Sent event occurs is
specific to the implementation and the constraints on the Protocol Stacks
implied by the Connection's transport parameters. The Sent event contains an
implementation-specific reference to the Message to which it applies.

Sent events allow an application to obtain an understanding of the amount
of buffering it creates. That is, if an application calls the Send action multiple
times without waiting for a Sent event, it has created more buffer inside the
transport system than an application that only issues a Send after this event fires.

~~~
Connection -> Expired<msgRef>
~~~

The Expired event occurs when a previous Send action expired before completion;
i.e. when the Message was not sent before its Lifetime (see {{send-lifetime}})
expired. This is separate from SendError, as it is an expected behavior for
partially reliable transports. The Expired event contains an
implementation-specific reference to the Message to which it applies.

~~~
Connection -> SendError<msgRef>
~~~

A SendError occurs when a Message could not be sent due to an error condition:
an attempt to send a non-partial Message which is too large for the system and
Protocol Stack to handle, some failure of the underlying Protocol Stack, or a
set of send parameters not consistent with the Connection's transport
parameters. The SendError contains an implementation-specific reference to the
Message to which it applies.

## Send Parameters {#send-params}

The Send action takes per-Message send parameters which control how the
contents will be sent down to the underlying Protocol Stack and transmitted.

If Send Parameters should be overridden for a specific Message, an
empty sent parameter Object can be acquired and all desired Send Parameters
can be added to that object. A sendParameters object can be reused for
sending multiple contents with the same properties.

~~~
sendParameters := NewSendParameters()
sendParameters.add(parameter, value)
~~~

The Send Parameters are organized in *Message Properties* and *Application Intents*.
The Send Parameters share a single namespace with the Transport Parameters (see
{{transport-params}}). This allows to specify Protocol Properties and that can
be overridden on a per-Message basis or Application Intents that apply to a
specific Message.
See {{appendix-specify-query-params}} for an overview.

Send Parameters may be inconsistent with the properties of the Protocol Stacks
underlying the Connection on which a given Message is sent. For example,
infinite Lifetime is not possible on a Message over a Connection not providing
reliability. Sending a Message with Send Properties inconsistent with the
Transport Preferences on the connection yields an error.

### Message Properties

#### Lifetime {#send-lifetime}

Lifetime specifies how long a particular Message can wait to be sent to the
remote endpoint before it is irrelevant and no longer needs to be
(re-)transmitted. When a Message's Lifetime is infinite, it must be
transmitted reliably. The type and units of Lifetime are
implementation-specific.

#### Niceness {#send-niceness}

Niceness represents an unbounded hierarchy of priorities of Messages, relative
to other Messages sent over the same Connection and/or Connection Group (see
{{groups}}). It is most naturally represented as a non-negative integer.
A Message with Niceness 0 will yield to a Message with Niceness 1, which will
yield to a Message with Niceness 2, and so on. Niceness may be used as a
sender-side scheduling construct only, or be used to specify priorities on the
wire for Protocol Stacks supporting prioritization.

Note that this inversion of normal schemes for expressing priority has a
convenient property: priority increases as both Niceness and Lifetime
decrease.

#### Ordered {#send-ordered}

Ordered is a boolean property. If true, this Message should be delivered after
the last Message passed to the same Connection via the Send action; if false,
this Message may be delivered out of order.

#### Idempotent {#send-idempotent}

Idempotent is a boolean property. If true, the application-layer entity in the
Message is safe to send to the remote endpoint more than once for a single
Send action. It is used to mark data safe for certain 0-RTT establishment
techniques, where retransmission of the 0-RTT data may cause the remote
application to receive the Message multiple times.

#### Corruption Protection Length {#send-checksum}

This numeric property specifies the length of the section of the Message,
starting from byte 0, that the application assumes will be received without
corruption due to lower layer errors. It is used to specify options for simple
integrity protection via checksums. By default, the entire Message is protected
by checksum. A value of 0 means that no checksum is required, and a special
value (e.g. -1) can be used to indicate the default. Only full coverage is
guaranteed, any other requests are advisory.

#### Immediate Acknowledgement {#send-ackimmed}

This boolean property specifies, if true, that an application wants this
Message to be acknowledged immediately by the receiver. In case of reliable
transmission, this informs the transport protocol on the sender side faster
that it can remove the Message from its buffer; therefore this property can be
useful for latency-critical applications that maintain tight control over the
send buffer (see {{sending}}).

#### Instantaneous Capacity Profile

This enumerated property specifies the application's preferred tradeoffs for
sending this Message; it is a per-Message override of the Capacity Profile
protocol and path selection property (see {{transport-params}}).

The following values are valid for Instantaneous Capacity Profile:

  Default:
  :  No special optimizations of the tradeoff between delay, delay
  variation, and bandwidth efficiency should be made when sending this message.

  Interactive/Low Latency:
  : Response time (latency) should be optimized at the
  expense of bandwidth efficiency and delay variation when sending this message.
  This can be used by the system to disable the coalescing of multiple small
  Messages into larger packets (Nagle's algorithm), to signal a preference for
  lower-latency, higher-loss treatment, and so on.

  Constant Rate:
  : Delay and delay variation should be optimized at the
  expense of bandwidth efficiency.

  Scavenger/Bulk:
  : This Message may be sent at the system's leisure. This can
  be used to signal a preference for less-than-best-effort treatment, to delay
  sending until lower-cost paths are available, and so on.

## Sender-side Framing {#send-framing}

Sender-side framing allows a caller to provide the interface with a function
that takes a Message of an appropriate application-layer type and returns an
array of octets, the on-the-wire representation of the Message to be handed down
to the Protocol Stack. It consists of a Framer object with a single Action,
Frame. Since the Framer depends on the protocol used at the application layer,
it is bound to the Preconnection during the pre-establishment phase:

~~~
Preconnection.FrameWith(Framer)

OctetArray := Framer.Frame(Message)
~~~

Sender-side framing is a convenience feature of the interface, for parity with
receiver-side framing (see {{receive-framing}}).

# Receiving Data {#receiving}

Once a Connection is established, Messages may be received on it. The application can indicate that it is ready to receive Messages by calling Receive() on the connection.

~~~
Connection.Receive(ReceiveHandler)
~~~

Receive takes a single object, a ReceiveHandler which can handle the Received
event and the ReceiveError error. Each call to Receive will result in at most
one Received event being sent to the handler, though implementations may provide
convenience functions to indicate readiness to receive a larger but finite
number of Messages with a single call. This allows an application to provide
backpressure to the transport stack when it is temporarily not ready to receive
messages.

~~~
Connection -> Received<Message>
~~~

As with sending, the type of the Message to be passed is dependent on the
implementation, and on the constraints on the Protocol Stacks implied by the
Connection's transport parameters. The Message may also contain metadata from
protocols in the Protocol Stack; which metadata is available is Protocol Stack
dependent. In particular, when this information is available, the value of the
Explicit Congestion Notification (ECN) field is contained in such metadata.
This information can be used for logging and debugging purposes, and for
building applications which need access to information about the transport
internals for their own operation.

The Message object must provide some method to retrieve an octet array
containing application data, corresponding to a single message within the
underlying Protocol Stack's framing.  See {{receive-framing}} for handling
framing in situations where the Protocol Stack provides octet-stream transport
only.

The Message object passed to Received is complete and atomic, unless one of the following
conditions holds:

* the underlying protocol stack supports message boundary preservation, and
  the size of the Message is larger than the buffers available for a single
  message;
* the underlying protocol stack does not support message boundary
  preservation, and the deframer (see {{receive-framing}}) cannot determine
  the end of the message using the buffer space it has available; or
* the underlying protocol stack does not support message boundary
  preservation, and no deframer was supplied by the application

In this case, the Message object passed to Received contains an indication
that the object received is partial, the byte offset of the data in the
partial Message within the full Message, an indication whether this is the
last (highest-offset) partial Message in the full Message, and an optional
reference to the full Message it belongs to.

Note that in the degenerate case -- no message boundary preservation and no
deframing -- the entire connection is represented as one large message of
indeterminate length.

~~~
Connection -> ReceiveError<>
~~~

A ReceiveError occurs when data is received by the underlying Protocol Stack
that cannot be fully retrieved or deframed, or when some other indication is
received that reception has failed. Such conditions that irrevocably lead the
the termination of the Connection are signaled using ConnectionError instead
(see {{termination}}).


## Receiver-side De-framing over Stream Protocols {#receive-framing}

The Receive event is intended to be fired once per application-layer Message
sent by the remote endpoint; i.e., it is a desired property of this interface
that a Send at one end of a Connection maps to exactly one Receive on the
other end. This is possible with Protocol Stacks that provide
message boundary preservation, but is not the case over Protocol Stacks that
provide a simple octet stream transport.

For preserving message boundaries over stream transports, this interface
provides receiver-side de-framing. This facility is based on the observation
that, since many of our current application protocols evolved over TCP, which
does not provide message boundary preservation, and since many of these protocols
require message boundaries to function, each application layer protocol has
defined its own framing. A Deframer allows an application to push this
de-framing down into the interface, in order to transform an octet stream into
a sequence of Messages.

Concretely, receiver-side de-framing allows a caller to provide the interface
with a function that takes an octet stream, as provided by the underlying
Protocol Stack, reads and returns a single Message of an appropriate type for
the application and platform, and leaves the octet stream at the start of the
next Message to deframe. It consists of a Deframer object with a single Action,
Deframe. Since the Deframer depends on the protocol used at the application
layer, it is bound to the Preconnection during the pre-establishment phase:

~~~
Preconnection.DeframeWith(Deframer)

Message := Deframer.Deframe(OctetStream, ...)
~~~

# Setting and Querying of Connection Properties {#introspection}


At any point, the application can set and query the properties of a
Connection. Depending on the phase the connection is in, the connection
properties will include different information.

~~~
connectionProperties := Connection.getProperties()
~~~

~~~
Connection.setProperties()
~~~

Connection properties include:

* The status of the Connection, which can be one of the following: Establishing, Established, Closing, or Closed.
* Transport Features of the protocols that conform to the Required and Prohibited Transport Preferences, which might be selected by the transport system during Establishment. These features correspond to the properties given in {{transport-params}} and can only be queried.
* Transport Features of the protocols that were selected, once the Connection has been established. These features correspond to the properties given in {{transport-params}} and can only be queried.
* Protocol Properties of the Protocol Stack in use (see {{protocol-props}} below). These can be set or queried. Certain specific procotol queries may be read-only, on a protocol and property specific basis.
* Path Properties of the path(s) in use, once the Connection has been established. These  properties can be derived from the local provisioning domain, measurements by the protocol stack, or other sources. They can only be queried.

{{appendix-specify-query-params}} gives a more detailed overview of the different types of properties that can be set and queried at different times.

## Protocol Properties {#protocol-props}

Protocol Properties represent the configuration of the selected Protocol Stacks
backing a connection. Some properties apply generically across multiple
transport protocols, while other properties only apply to specific protocols.
The default settings of these properties will vary based on the specific
protocols being used and the system's configuration.

Note that Protocol Properties are also set during pre-establishment, as
transport parameters, to preconfigure Protocol Stacks during establishment.

Generic Protocol Properties include:

* Timeout for aborting Connection: This numeric property specifies how long to
  wait before aborting a Connection during establishment, or before deciding
  that a connection has failed after establishment. It is given in seconds.

* Abort timeout to suggest to the Remote Endpoint:
  This numeric property
  specifies the timeout to propose to the Remote Endpoint. It is given in
  seconds.

* Retransmission threshold before excessive retransmission notification:
  This numeric property specifies after how many retransmissions to inform
  the application about "Excessive Retransmissions".

* Required minimum coverage of the checksum for receiving:
  This numeric property specifies the part of the received data that needs
  to be covered by a checksum. It is given in Bytes. A value of 0 means
  that no checksum is required, and a special value (e.g., -1) indicates
  full checksum coverage.

* Connection group transmission scheduler:
  This enumerated property specifies which
  scheduler should be used among Connections within a Connection Group. It
  applies to connection groups; the set of schedulers can be taken from
  {{I-D.ietf-tsvwg-sctp-ndata}}.

* Maximum message size concurrent with connection establishment:
  This numeric property represents the maximum Message size that can be sent
  before or during Connection establishment, see also {{send-idempotent}}.
  It is given in Bytes. This property is read-only.

* Maximum Message size before fragmentation or segmentation:
  This numeric property, if applicable, represents the maximum Message size
  that can be sent without incurring network-layer fragmentation and/or transport layer segmentation at the sender. This property is read-only.

* Maximum non-partial Message size on send: This numeric property represents
  the maximum Message size that can be sent as a non-partial Message. This
  property is read-only.

* Maximum non-partial Message size on receive: This numeric property
  represents the maximum Message size that can be received as a non-partial
  Message. This property is read-only.

In order to specify Specific Protocol Properties, Transport System
implementations may offer applications to attach a set of options to the
Preconnection object, associated with a specific protocol. For example, an
application could specify a set of TCP Options to use if and only if TCP is
selected by the system. Such properties must not be assumed to apply across
different protocols. Attempts to set specific protocol properties on a
protocol stack not containing that specific protocol are simply ignored, and
do not raise an error..

# Connection Termination {#termination}

Close terminates a Connection after satisfying all the requirements that were
specified regarding the delivery of Messages that the application has already
given to the transport system. For example, if reliable delivery was requested
for a Message handed over before calling Close, the transport system will ensure
that this Message is indeed delivered. If the Remote Endpoint still has data to
send, it cannot be received after this call.

~~~
Connection.Close()
~~~

The Closed event can inform the application that the Remote Endpoint has closed the
Connection; however, there is no guarantee that a remote close will be
signaled.

~~~
Connection -> Closed<>
~~~

Abort terminates a Connection without delivering remaining data:

~~~
Connection.Abort()
~~~

A ConnectionError can inform the application that the other side has aborted
the Connection; however, there is no guarantee that an abort will be signaled:

~~~
Connection -> ConnectionError<>
~~~

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

This work has been supported by Leibniz Prize project funds of DFG - German
Research Foundation: Gottfried Wilhelm Leibniz-Preis 2011 (FKZ FE 570/4-1).

Thanks to Stuart Cheshire, Josh Graessley, David Schinazi, and Eric Kinnear for
their implementation and design efforts, including Happy Eyeballs, that heavily
influenced this work. Thanks to Laurent Chuat and Jason Lee for initial work on
the Post Sockets interface, from which this work has evolved.

--- back

# Additional Properties {#appendix-non-consensus}

The interface specified by this document represents the minimal common interface
to an endpoint in the transport services architecture {{TAPS-ARCH}}, based upon
that architecture and on the minimal set of transport service features
elaborated in {{I-D.ietf-taps-minset}}. However, the interface has been designed with
extension points to allow the implementation of features beyond those in the
minimal common interface: Protocol Selection Properties, Path Selection
Properties, and options on Message send are open sets. Implementations of the
interface are free to extend these sets to provide additional expressiveness to
applications written on top of them.

This appendix enumerates a few additional parameters and properties that could
be used to enhance transport protocol and/or path selection, or the transmission
of messages given a Protocol Stack that implements them. These are not part of
the interface, and may be removed from the final document, but are presented
here to support discussion within the TAPS working group as to whether they
should be added to a future revision of the base specification.

## Protocol and Path Selection Properties

The following protocol and path selection properties might be made available in
addition to those specified in {{transport-params}}:

* Request not to delay acknowledgment of Message:
  This boolean property specifies whether an application considers it
  useful to request for Message that its acknowledgment be sent out as
  early as possible instead of potentially being bundled with other
  acknowledgments. This property applies to connections and connection
  groups. This is not a strict requirement. The default is to not have this
  option.

### Application Intents {#intents}

Application Intents are a group of transport properties expressing what an
application wants to achieve, knows, assumes or prefers regarding its
communication. They are not strict requirements. In particular, they should not
be used to express any Quality of Service expectations that an application might
have. Instead, an application should express its intentions and its expected
traffic characteristics in order to help the transport system make decisions
that best match it, but on a best-effort basis. Even though Application Intents
do not represent Quality of Service requirements, a transport system may use
them to determine a DSCP value, e.g. similar to Table 1 in
{{I-D.ietf-tsvwg-rtcweb-qos}}.

Application Intents can influence protocol selection, protocol configuration,
path selection, and endpoint selection. For example, setting the "Timeliness"
Intent to "Interactive" may lead the transport system to disable the Nagle
algorithm for a connection, while setting the "Timeliness" to "Background" may
lead it to setting the DSCP value to "scavenger". If the "Size to be Sent"
Intent is set on an individual Message, it may influence path selection.

Specifying Application Intents is not mandatory. An application can specify any
combination of Application Intents. If specified, Application Intents are
defined as parameters passed to the Preconnection object, and may influence the
Connection established from that Preconnection. If a Connection is cloned to
form a Connection Group, and associated Application Intents are cloned along
with the other transport parameters. Some Intents have also corresponding
Message Properties, similar to the properties in {{send-params}}.

Application Intents can be added to this interface as Transport Preferences with
the "prefer" preference level.

#### Traffic Category

This Intent specifies what the application expect the dominating traffic
pattern to be.

Possible Category values are:

Query:
: Single request / response style workload, latency bound

Control:
: Long lasting low bandwidth control channel, not bandwidth bound

Stream:
: Stream of data with steady data rate

Bulk:
: Bulk transfer of large Messages, presumably bandwidth bound

The default is to not assume any particular traffic pattern. Most categories
suggest the use of other intents to further describe the traffic pattern
anticipated, e.g., the bulk category suggesting the use of the Message Size
intents or the stream category suggesting the Stream Bitrate and Duration
intents.

#### Size to be Sent / Received

This Intent specifies what the application expects the size of a transfer to be.
It is a numeric property and given in Bytes.

#### Duration

This Intent specifies what the application expects the lifetime of a transfer
to be. It is a numeric property and given in milliseconds.

#### Send / Receive Bit-rate

This Intent specifies what the application expects the bit-rate of a transfer to
be. It is a numeric property and given in Bytes per second.

#### Cost Preferences

This Intent describes what an application prefers regarding monetary costs,
e.g., whether it considers it acceptable to utilize limited data volume. It
provides hints to the transport system on how to handle trade-offs between cost
and performance or reliability. This Intent can also apply to an individual
Messages.

No Expense:
: Avoid transports associated with monetary cost

Optimize Cost:
: Prefer inexpensive transports and accept service degradation

Balance Cost:
: Use system policy to balance cost and other criteria

Ignore Cost:
: Ignore cost, choose transport solely based on other criteria

The default is "Balance Cost".

## Send Parameters

The following send parameters might be made available in addition to those
specified in {{send-params}}:

* Immediate:
  Immediate is a boolean property. If true, the caller prefers immediacy to
  efficient capacity usage for this Message. For example, this means that
  the Message should not be bundled with other
  Message into the same transmission by the underlying Protocol Stack.

* Send Bitrate:
  This numeric property in Bytes per second specifies at what
  bitrate the application wishes the Message to be sent. A transport supporting
  this feature will not exceed the requested Send Bitrate even if flow-control
  and congestion control allow higher bitrates. This helps to avid bursty
  traffic pattern on busy video streaming servers.

# Sample API definition in Go {#appendix-api-sketch}

\[EDITOR'S NOTE: rewrite following discussion and copy back here; see
https://github.com/taps-api/drafts/issues/39]

This document defines an abstract interface. To illustrate how this would map concretely into a programming language, this appendix contains an API interface definition in Go, using callbacks for event handling. The documentation for the API sketch is available online at https://godoc.org/github.com/mami-project/postsocket.

~~~~~~~
// Package postsocket specifies a Go interface for the TAPS API. This abstract
// interface to transport-layer service is described in
// https://taps-api.github.io/drafts/draft-trammell-taps-interface.html. For
// now, read that document to understand what's going on in this package.
// Eventually, this package will grow to contain a demonstration
// implementation of the API.
//
// A Note on Error Handling
//
// This API provides two ways for its client to learn of errors: through the
// Error event passed to a connection's EventHandler, and through error
// returns on various calls. In general, errors in networking are
// asynchronous, so almost every error involving the network will be passed
// through the Error event. The error returns on calls are therefore only used
// for immediately detectable errors, such as inconsistent arguments or
// states.
package postsocket

import (
  "crypto"
  "crypto/tls"
  "io"
  "net"
  "time"
)

// TransportContext encapsulates all the state kept by the API at a single
// endpoint, and is the "root" of the API. It can be used to create new
// default transport and security parameters, locals and remotes bound to the
// available provisioning domains and resolution, as well as new Connections
// with these properties. It stores path (address pair) and association
// (endpoint pair) properties and ephemeral as well as durable state. An
// application will generally create a single TransportContext instance on
// startup, optionally Load() state from disk, and can checkpoint in-memory
// state to disk with Save().
type TransportContext interface {

  // NewTransportParameters creates a new TransportParameters object with
  // system and user defaults for this TransportContext. The specification
  // of system and user defaults is implementation-specific.
  NewTransportParameters() TransportParameters

  // NewSecurityParameters creates a new SecurityParameters object with
  // system and user defaults for this TransportContext. The specification
  // of system and user defaults is implementation-specific.
  NewSecurityParameters() SecurityParameters

  // NewRemore creates a new, empty Remote specifier.
  NewRemote() Remote

  // NewLocal creates a new Local specifier initialized with defaults for
  // this TransportContext.
  NewLocal() Local

  // DefaultSendParameters returns a SendParameters object with default values.
  DefaultSendParameters() SendParameters

  // SetEventHandler sets the default connection handler for all
  // Connections created within this TransportContext.
  SetEventHandler(evh EventHandler)

  // SetFramingHandler sets the default framing handler for all
  // Connections created within this TransportContext.
  SetFramingHandler(fh FramingHandler)

  // Preconnect creates a Preconnection, which binds a connection and
  // framing handler to sets of related remote, local, transport and
  // security parameters (a Connection specifier) for Connection
  // instantiation. Any of Preconnect's arguments may  be nil, in which case
  // the default values for this TransportContext are used. Preconnection
  // allows the specification of multiple, disjoint sets of related
  // parameters for candidate transport protocol selection, as well as the
  // ability to initiate and send atomically for 0-RTT connection. The
  // Preconnection is initialized with one set of parameters; use
  // Preconnection.AddSpecifier to add more.

  Preconnect(evh EventHandler, fh FramingHandler, rem Remote, loc Local, tp TransportParameters, sp SecurityParameters) (Preconnection, error)

  // Initiate a connection with given remote, local, and parameters, and the
  // default connection and framing handlers. Any of these except Remote may
  // be nil, in which case context defaults will be used. Once the
  // Connection is initiated, the EventHandler's Ready callback will be
  // called with this connection and a nil antecedent. This is a shortcut
  // for creating a Preconnection with a single connection specifier and
  // initiating it.
  Initiate(rem Remote, loc Local, tp TransportParameters, sp SecurityParameters) (Connection, error)

  // Rendezvous with a given Remote using an appropriate peer to peer
  // rendezvous method, with
  // optional local, transport parameters, and security parameters. Each of
  // the optional arguments may be passed as nil; if so, the Context
  // defaults are used. Returns a Connection in the process of being
  // rendezvoused. The EventHandler's Ready callback will be called
  // with any established Connection(s), with a nil antecedent.
  // This is a shortcut for creating a Preconnection with a single
  // connection specifier and rendezvousing with it.
  Rendezvous(evh EventHandler, rem Remote, loc Local, tp TransportParameters, sp SecurityParameters) (Connection, error)

  // Listen on a given Local with a given Handler for connection events,
  // with optional transport and security parameters. Each of the optional
  // arguments may be passed as nil; if so, the Context defaults are used.
  // Returns a Listener in the process of being started. The
  // EventHandler's Ready callback will be called with any accepted
  // Connection(s), with this Connection as antecedent.
  Listen(evh EventHandler, loc Local, tp TransportParameters, sp SecurityParameters) (Connection, error)

  // Save this context's state to a file on disk. The format of this state
  // file is not specified and not necessarily portable across
  // implementations of the API.
  Save(filename string) error

  // Replace this context's state with state loaded from a file on disk. The
  // format of this state file is not specified and not necessarily portable
  // across implementations of the API.
  Restore(filename string) error
}

// Remote specifies a remote endpoint by hostname, address, port, and/or
// service name. Multiple of each of these may be given; this will result in a
// set of candidate endpoints assumed to be equivalent from the application's
// standpoint to be resolved and connected to. Resolution of the remote need
// not occur until a connection is created; any resolution error will be
// reported via the EventHandler when Intiate, Listen, or Rendezvous is
// called.
type Remote interface {
  // Return a remote specifier with the given hostname added to this specifier.
  WithHostname(hostname string) Remote

  // Return a remote specifier with the given IPv4 or IPv6 address added to this specifier
  WithAddress(address net.IP) Remote

  // Return a remote specifier with the given transport port added to this specifier
  WithPort(port uint16) Remote

  // Return a remote specifier with the given service name added to this specifier
  WithServiceName(svc string) Remote
}

// Local specifies a remote endpoint by interface name, hostname, address,
// port, and/or service name. Multiple of each of these may be given; this
// will result in a set of candidate endpoints assumed to be equivalent from
// the application's standpoint to be connected from or listened on. Any
// resolution error will be reported via the EventHandler when Intiate,
// Listen, or Rendezvous is called.
type Local interface {
  // Return a local specifier with the given local network interface name or alias added to this specifier
  WithInterface(iface string) Local

  // Return a local specifier with the given hostname added to this specifier
  WithHostname(hostname string) Local

  // Return a local specifier with the given IPv4 or IPv6 address added to this specifier
  WithAddress(address net.IP) Local

  // Return a local specifier with the given transport port added to this specifier
  WithPort(port uint16) Local

  // Return a local specifier with the given service name added to this specifier
  WithServiceName(svc string) Local
}

// ParameterIdentifier identifies a Transport or Security Parameter
type ParameterIdentifier int

// List of transport and security parameter names.
const (
  TransportFullyReliable = iota
  TransportOrderPreserved
  TransportPerMessageReliable
  TransportIdempotent0RTT
  TransportMultistreaming
  TransportTimeoutNegotiationSupport
  TransportExtendedErrorSupport
  TransportChecksumControl
  TransportInterfaceType
  TransportCapacityProfile
  TransportTimeout
  TransportSuggestTimeout
  TransportRetransmissionThreshold
  TransportMinimumReceiveChecksumCoverage
  TransportGroupTransmissionScheduler
  TransportMaxIdempotent0RTT
  TransportMaxNoFragment
  TransportMaxNonpartialSend
  TransportMaxNonpartialReceive
  SecuritySupportedGroup
  SecurityCiphersuite
  SecuritySignatureAlgorithm
  SecuritySessionCacheCapacity
  SecuritySessionCacheLifetime
  SecuritySessionCacheReuse
)

// CapacityProfile identifies a capacity profile value
type CapacityProfile int

// List of supported CapacityProfile values
const (
  CapProfDefault = iota
  CapProfInteractive
  CapProfConstantRate
  CapProfBulk
)

// TransportParameters contains a set of parameters used in the selection of
// transport protocol stacks and paths during connection pre-establishment.
// Get a new TransportParameters bound to a context with
// NewTransportParameters(), then set preferences through the Require(),
// Prefer(), Avoid(), and Prohibit() methods.
type TransportParameters interface {

  // Require protocols and paths selected to fulfill this parameter. Returns
  // a new TransportParameters with this requirement added. If no protocols
  // and paths available fulfill this parameter, then no connection is
  // possible. v is an optional value whose meaning is parameter-specific.
  Require(p ParameterIdentifier, v interface{}) TransportParameters

  // Prefer protocols and paths selected to fulfill this parameter. Returns
  // a new TransportParameters with this preference added. Preferences are
  // considered after requirements and prohibitions. v is an optional value
  // whose meaning is parameter-specific.
  Prefer(p ParameterIdentifier, v interface{}) TransportParameters

  // Ignore this parameter when selecting protocols and paths. Returns a new
  // TransportParameters with this ignorance added. This is used to cancel
  // system defaults that may consider this parameter.
  Ignore(p ParameterIdentifier) TransportParameters

  // Avoid protocols and paths selected to fulfill this parameter. Returns a
  // new TransportParameters with this avoidance added. Avoidences are
  // considered after requirements and prohibitions. v is an optional value
  // whose meaning is parameter-specific.
  Avoid(p ParameterIdentifier, v interface{}) TransportParameters

  // Prohibit the selection of protocols and paths that fulfill this
  // parameter. Returns a new TransportParameters with this prohibition
  // added.  If the protocols and paths available all fulfill this
  // parameter, then no connection is possible. v is an optional value whose
  // meaning is parameter-specific.
  Prohibit(p ParameterIdentifier, v interface{}) TransportParameters

  // Get retrieves the current value of a given transport parameter by
  // parameter identifier. Returns an error if the parameter identifier is
  // not gettable for this set of transport parameters. This is used to
  // query read-only and read-write properties on established Connections,
  // as well as to determine which Parameters were selected during
  // connection establishment.
  Get(p ParameterIdentifier) (interface{}, error)

  // Set sets the value of a given settable transport parameter by parameter
  // identifier. Returns an error if this parameter identifier is not
  // settable for this set of transport parameters, or if the type of the
  // given value is not appropriate for the transport parameter.
  Set(p ParameterIdentifier, v interface{}) error
}

// SecurityMetadata specifies information about a security association for
// trust verification and identity challenge callbacks.
type SecurityMetadata struct {
  Certificate tls.Certificate
  // and so on...
}

// SecurityParameters contains a set of parameters used in the establishment
// of security associations. Get a new SecurityParameters bound to a context
// with NewSecurityParameters(), then set preferences and associate identity
// and callbacks through the methods on the object.
type SecurityParameters interface {

  // AddIdentity adds an local identity (as a TLS certificate) to this parameter set.
  AddIdentity(c tls.Certificate) SecurityParameters

  // AddPrivateKey adds a public/private key pair to this parameter set.
  AddPrivateKey(sk crypto.PrivateKey, pk crypto.PublicKey) SecurityParameters

  // AddPSK adds an preshared key associated with a given identity (as a string) to
  // the parameter set.
  AddPreSharedKey(key []byte, identity string) SecurityParameters

  // VerifyTrustWith registers a callback to verify trust. This callback
  // takes metadata about a security association and returns true if the
  // association is trusted.
  VerifyTrustWith(func(m SecurityMetadata) (bool, error)) SecurityParameters

  // HandleChallengeWith registers a callback to handle identity challenges.
  // This callback takes metadata about a security association and returns
  // true if the association is trusted.
  HandleChallengeWith(func(m SecurityMetadata) (bool, error)) SecurityParameters

  // Get retrieves the current value of a given security parameter by
  // parameter identifier. Returns an error if the parameter identifier is
  // not gettable for this set of security parameters. This is used to
  // query read-only and read-write properties on established Connections,
  // as well as to determine which Parameters were selected during
  // session establishment.
  Get(p ParameterIdentifier) (interface{}, error)

  // Set sets the value of a given settable security parameter by parameter
  // identifier. Returns an error if this parameter identifier is not
  // settable for this set of security parameters, or if the type of the
  // given value is not appropriate for the transport parameter.
  Set(p ParameterIdentifier, v interface{}) error
}

// SendParameters contains a set of parameters used for sending Messages.
// DefaultSendParameters() returns the defaults for this context.
type SendParameters struct {
  // Lifetime after which the object is no longer relevant. Used for
  // unreliable and partially reliable transports; set to zero or less to
  // specify fully reliable transport, if available.
  Lifetime time.Duration
  // Niceness is the inverse priority of this Message relative to others on
  // this Connection or within this ConnectionGroup. Niceness 0 messages are
  // the highest priority.
  Niceness uint
  // Ordered is true if the Message must be sent before the next Message
  // sent on this Connection
  Ordered bool
  // Immediate is true if this Message should not be held for coalescing
  // with other Messages in a transport-layer datagram.
  Immediate bool
  // Idempotent is true if this Message may be sent to the application more
  // than once without ill effects. Use this together with SendInitial() for
  // 0RTT session resumption.
  Idempotent bool
  // CorruptionTolerant is true if this Message may be sent to the
  // application even if checksums fail; it is used to explicitly disable
  // checksums on sent Message.
  CorruptionTolerant bool
  // CapacityProfile is the capacity profile to use for this message only
  CapacityProfile CapacityProfile
}

// Preconnection is a container for sets of related remote, local, transport
// and security parameters (a Connection specifier), which can be instantiated
// into a Connection. Use Preconnect() to create one with an initial
type Preconnection interface {

  // AddSpecifier adds a related set of remote, local, transport and
  // security parameters to this Preconnection.
  AddSpecifier(rem Remote, loc Local, tp TransportParameters, sp SecurityParameters)

  // Initiate a Connection with a Remote specified by this Preconnection,
  // using the Local and parameters supplied. Returns a connection in the
  // initiation process. Once the Connection is initiated, the
  // EventHandler's Ready callback will be called with this connection
  // and a nil antecedent.
  Initiate() (Connection, error)

  // Initiate a Connection with a Remote specified by this Preconnection,
  // using the Local and parameters supplied, while simultaneously sending a
  // Message with the given SendParameters. Returns a connection in the
  // initiation process. IntialSend may be called more than once on a given
  // connection to send multiple Messages during initiation. Once the
  // Connection is initiated, the EventHandler's Ready callback will be
  // called with this connection and a nil antecedent.
  InitialSend(message interface{}, sp SendParameters) (Connection, error)

  // Rendezvous  using an appropriate peer to peer rendezvous method with a
  // Remote specified by this Preconnection, using the Local and parameters
  // supplied. Returns a connection in the rendezvous process. The
  // EventHandler's Ready callback will be called with any established
  // Connection(s), with a nil antecedent.
  Rendezvous() (Connection, error)

  // Listen for connections on the Local specified by this Preconnection
  // using the Local and parameters supplied. Returns a Listener in the
  // process of being started. The EventHandler's Ready callback will
  // be called with any accepted Connection(s), with this Connection as
  // antecedent.
  Listen() (Connection, error)
}

// Connection encapsulates a connection to another endpoint. All events on the
// connection will be passed to its associated EventHandler.
type Connection interface {

  // Send sends a Message on this connection, with an optional message
  // reference, an object that will be used to refer to the Message on any
  // event related to it, and a set of send parameters to govern how it will
  // be sent. If msg is a []byte, the bytes it contains will be sent to over
  // the connection as a single Message. If msg implements the Message
  // interface, the Bytes() method will be invoked and the resulting []byte
  // will be transmitted. Otherwise, msg will be passed to the framing
  // handler's Frame() method to convert it to a []byte for transmission.
  Send(msg interface{}, msgref interface{}, sp SendParameters) error

  // Receive informs this Connection that the application is ready to
  // receive the next message. The receiver argument is a function that will
  // be called once per call to Receive and passed a successfully received
  // Message on this Connection; this callback takes the Message received
  // and the Connection upon which it was received. Note that when multiple
  // Receive calls are pending, there is no guarantee on the order in which
  // the callbacks will be invoked.
  Receive(receiver func(msg Message, conn Connection))

  // Clone clones this Connection, creating a new Connection to the same
  // remote endpoint. If the underlying protocol stack supports
  // multistreaming, then this will create a new stream; otherwise, a new
  // transport connection (flow) will be created.
  Clone() (Connection, error)

  // Close closes this connection.
  Close() error

  // GetEventHandler returns this connection's event handler.
  GetEventHandler() EventHandler

  // SetEventHandler replaces this connection's event handler.
  SetEventHandler(evh EventHandler)

  // GetEventHandler returns this connection's framing handler.
  GetFramingHandler() FramingHandler

  // SetEventHandler replaces this connection's framing handler.
  SetFramingHandler(fh FramingHandler)

  // GetTransportParameters returns this connection's current transport parameter set.
  GetTransportParameters() TransportParameters
}

// Message provides the interface implemented by received Messages passed to a
// Received event.
type Message interface {
  // Bytes returns the content of a Message as a byte array
  Bytes() []byte

  // Partial allows a caller to determine whether this Message represents a
  // complete message, or part of an incomplete message. If the message is
  // complete, Partial returns false. If the message is partial, Partial
  // returns true, the offset into the full message of the first byte in the
  // slice returned by Bytes(), and true if this message is not yet complete
  // (i.e., there are Partial messages with higher offsets).
  Partial() (bool, int, bool)
}

// EventHandler defines the interface for connection event handlers. Every
// asynchronous event that can occur on a Connection, except message
// reception, is passed to an instance of this interface.
type EventHandler interface {

  // Ready occurs when a new connection is ready for use. The conn argument
  // contains the new connection, and the ante argument contains the
  // connection from which this connection was created. In the case of
  // Connections opened with Initate() and Rendezvous(), ante will be nil.
  // In the case of passively-opened Connections (i.e., created after
  // Listen()), ante will contain the listening Connection. In the case of
  // Connections created by Clone(), ante will contain the connection on
  // which Clone() was called. In the case of Connections created because a
  // remote endpoint created new streams with a multistreaming transport
  // protocol, ante will contain a Connection wrapped around one of the
  // other streams.
  Ready(conn Connection, ante Connection)

  // Sent occurs when a Message has been sent. The msgref argument
  // contains the message reference given on Send().
  Sent(conn Connection, msgref interface{})

  // Expired occurs when a Message's lifetime expires without the Message
  // having been sent. The msgref argument contains the message reference
  // given on Send().
  Expired(conn Connection, msgref interface{})

  // Error occurs when an error occurs on a connection. If the error refers
  // to an attempt to send a Message, the msgref argument contains the
  // content reference given on Send(); otherwise, it is nil. The Error
  // event only occurs for errors which do not also cause the connection to
  // close; Connection-closing errors will cause the Closed event to occur.
  Error(conn Connection, msgref interface{}, err error)

  // Closed occurs when a connection is closed, either actively through
  // Close(), passively because the remote side ended the connection, or
  // because a connection-ending error occurred. In this case, the
  // error is passed as the err argument.
  Closed(conn Connection, err error)
}

// FramingHandler defines the interface for application-assisted framing and deframing
type FramingHandler interface {
  // Frame converts a message object as passed to the Send() call into a
  // []byte to be passed down to the protocol stack.
  Frame(msg interface{}) ([]byte, error)

  // Deframe reads the next object from a given reader, and returns it as an
  // object of a type implementing Message. Deframe will only be called when
  // receiving data via a transport protocol which does not provide its own
  // framing (e.g. TCP)
  Deframe(in io.Reader) (Message, error)
}

~~~~~~~

# Transport Parameters {#appendix-transport-params}

\[EDITOR'S NOTE: move into main document as necessary; find a way to move the
rest into Appendix A or cut it; see
https://github.com/taps-api/drafts/issues/69]

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
transport protocol that allows to suggest a timeout to the Remote Endpoint, this
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
| Request immediate ACK  | Yes      | Yes    | Yes    | Yes     | None    |
| Use 0-RTT with Idempotent Content | Yes | Yes | Yes | Yes     | None    |
| Use Connection Groups with priorities | Yes | Yes | No | No   | None    |
| Suggest timeout to Remote Endpoint | Yes     | Yes    | No    | No       | None    |
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
