---
title: An Abstract Application Layer Interface to Transport Services
abbrev: TAPS Interface
docname: draft-ietf-taps-interface-latest
date:
category: std

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
    org: Independent
    role: editor
    email: ietf@trammell.ch
    street: ""
    city: 8304 Wallisellen
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
    street: Marchstrasse 23
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
    street: Marchstrasse 23
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
  I-D.ietf-tsvwg-sctp-ndata:
  I-D.ietf-tsvwg-rtcweb-qos:
  I-D.ietf-taps-arch:

informative:
  I-D.ietf-taps-minset:
  I-D.ietf-taps-transport-security:
  RFC7556:

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
applications, was a revolution in simplicity. The simplicity of this API is a
key reason the Internet won the protocol wars of the 1980s. SOCK_STREAM is
tied to the Transmission Control Protocol (TCP), specified in 1981
{{?RFC0793}}. TCP has scaled remarkably well over the past three and a half
decades, but its total ubiquity has hidden an uncomfortable fact: the network
is not really a file, and stream abstractions are too simplistic for many
modern application programming models.

In the meantime, the nature of Internet access, and the variety of Internet
transport protocols, is evolving. The challenges that new protocols and access
paradigms present to the sockets API and to programming models based on them
inspire the design principles of a new approach, which we outline in {{principles}}.

As a first step to realizing this design, {{I-D.ietf-taps-arch}}
describes a high-level architecture for transport services. This document
builds a modern abstract programming interface atop this architecture,
deriving specific path and protocol selection properties and supported
transport features from the analysis provided in {{?RFC8095}},
{{I-D.ietf-taps-minset}}, and {{I-D.ietf-taps-transport-security}}.

# Terminology and Notation

This API is described in terms of Objects, which an application can interact
with; Actions the application can perform on these Objects; Events, which an
Object can send to an application asynchronously; and Parameters associated
with these Actions and Events.

The following notations, which can be combined, are used in this document:

- An Action creates an Object:

~~~
Object := Action()
~~~

- An Action creates an array of Objects:

~~~
[]Object := Action()
~~~

- An Action is performed on an Object:

~~~
Object.Action()
~~~

- An Object sends an Event:

~~~
Object -> Event<>
~~~

- An Action takes a set of Parameters; an Event contains a set of Parameters.
  Action parameters whose names are suffixed with a question mark are optional.

~~~
Action(param0, param1?, ...) / Event<param0, param1, ...>
~~~

Actions associated with no Object are Actions on the abstract interface
itself; they are equivalent to Actions on a per-application global context.

How these abstract concepts map into concrete implementations of this API in a
given language on a given platform is largely dependent on the features of the
language and the platform. Actions could be implemented as functions or method
calls, for instance, and Events could be implemented via callbacks,
communicating sequential processes, or other asynchronous calling conventions.
The method for dispatching and handling Events is left as an implementation
detail, with the caveat that the interface for receiving Messages must require
the application to invoke the Connection.Receive() Action once per Message to be
received (see {{receiving}}).

This specification treats Events and errors similarly. Errors, just as any
other Events, may occur asynchronously in network applications. However, it is
recommended that implementations of this interface also return errors
immediately, according to the error handling idioms of the implementation
platform, for errors which can be immediately detected, such as inconsistency
in Transport Properties.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Interface Design Principles {#principles}

The design of the interface specified in this document is based on a set of
princples, themselves an elaboration on the architectural design principles
defined in {{I-D.ietf-taps-arch}}. The interface defined in this document
provides:

- A single interface to a variety of transport protocols to be
  used in a variety of application design patterns, independent of the
  properties of the application and the Protocol Stacks that will be used at
  runtime, such that  all common specialized features of these protocol
  stacks are made available to the application as necessary in a
  transport-independent way, to enable applications written to a single API
  to make use of transport protocols in terms of the features they provide;

- Message- as opposed to stream-orientation, using application-assisted framing
  and deframing where the underlying transport does not provide these;

- Asynchronous Connection establishment, transmission, and reception, allowing
  concurrent operations during establishment and supporting event-driven
  application interactions with the transport layer, in line with developments
  in modern platforms and programming languages;

- Explicit support for security properties as first-order transport features,
  and for long-term caching of cryptographic identities and parameters for
  associations among endpoints; and

- Explicit support for multistreaming and multipath transport protocols, and
  the grouping of related Connections into Connection Groups through cloning
  of Connections, to allow applications to take full advantage of new
  transport protocols supporting these features.


# API Summary

The Transport Services Interface is the basic common abstract application
programming interface to the Transport Services Architecture defined in
{{I-D.ietf-taps-arch}}.

An application primarily interacts with this interface through two Objects,
Preconnections and Connections. A Preconnection represents a set of properties
and constraints on the selection and configuration of paths and protocols to
establish a Connection with a remote endpoint. A Connection represents a
transport Protocol Stack on which data can be sent to and/or received from a
remote endpoint (i.e., depending on the kind of transport, connections can be
bi-directional or unidirectional). Connections can be created from
Preconnections in three ways: by initiating the Preconnection (i.e., actively
opening, as in a client), through listening on the Preconnection (i.e.,
passively opening, as in a server), or rendezvousing on the Preconnection (i.e.
peer to peer establishment).

Once a Connection is established, data can be sent on it in the form of
Messages. The interface supports the preservation of message boundaries both
via explicit Protocol Stack support, and via application support through a
deframing callback which finds message boundaries in a stream. Messages are
received asynchronously through a callback registered by the application.
Errors and other notifications also happen asynchronously on the Connection.

{{pre-establishment}}, {{establishment}}, {{sending}}, {{receiving}}, and
{{termination}} describe the details of application interaction with Objects
through Actions and Events in each phase of a Connection, following the phases
described in {{I-D.ietf-taps-arch}}.

## Transport Properties

Each application using the Transport Services Interface declares its preferences
for how the transport service should operate using properties at each stage of
the lifetime of a connection. During pre-establishment, Selection Properties
(see {{selection-props}}) are used to specify which paths and protocol stacks can be
used and are preferred by the application, and Connection Properties
(see {{connection-props}}) can be used to fine-tune the eventually established
connection. These Connection Properties can also be used later, to monitor and
fine-tune established connections -- but configuring Connection Properties on
Preconnections is preferred because they can then can influence
decisions made during establishment. The behavior of the selected protocol
stack(s) when sending Messages is controlled by Message Properties
(see {{message-props}}).

Collectively, Selection, Connection, and Message Properties can be
referred to as Transport Properties. All Transport Properties, regardless of the
phase in which they are used, are organized within a single namespace. This
enables setting them as defaults in earlier stages and querying them in later
stages:
 - Connection Properties can be set on Preconnections
 - Message Properties can be set on Preconnections and Connections
 - The effect of Selection Properties can be queried on Connections and Messages

Transport Properties can have one of a set of data types:

- Boolean: can take the values "true" and "false"; representation is
  implementation-dependent.
- Integer: can take positive or negative numeric values; range and
  representation is implementation-dependent.
- Enumeration: can take one value of a finite set of values, dependent on the
  property itself. The representation is implementation dependent; however,
  implementations MUST provide a method for the application to determine the
  entire set of possible values for each property.
- Preference: can take one of five values (Prohibit, Avoid, Ignore, Prefer,
  Require) for the level of preference of a given property during protocol
  selection; see {{selection-props}}.

## Scope of the Interface Definition

This document defines a language- and platform-independent interface to a
Transport Services system. Given the wide variety of languages and language
conventions used to write applications that use the transport layer to connect
to other applications over the Internet, this independence makes this interface
necessarily abstract. While there is no interoperability benefit to tightly
defining how the interface be presented to application programmers in diverse
platforms, maintaining the "shape" of the abstract interface across these
platforms reduces the effort for programmers who learn the transport services
interface to apply their knowledge in multiple platforms. We therefore make the
following recommendations:

- Actions, Events, and Errors in implementations of this interface SHOULD carry
  the names given for them in the document, subject to capitalization and
  punctuation conventions in the language of the implementation, unless the
  implementation itself uses different names for substantially equivalent
  objects for networking by convention.
- Implementations of this interface SHOULD implement each Selection Property,
  Connection Property, and Message Context Property specified in this document,
  exclusive of appendices, even if said implementation is a non-operation, e.g.
  because transport protocols implementing a given Property are not available on
  the platform.

# Pre-Establishment Phase {#pre-establishment}

The pre-establishment phase allows applications to specify properties for
the Connections they are about to make, or to query the API about potential
connections they could make.

A Preconnection Object represents a potential Connection. It has state that
describes properties of a Connection that might exist in the future.  This state
comprises Local Endpoint and Remote Endpoint Objects that denote the endpoints
of the potential Connection (see {{endpointspec}}), the Selection Properties
(see {{selection-props}}), any preconfigured Connection Properties
({{connection-props}}), and the security parameters (see
{{security-parameters}}):

~~~
   Preconnection := NewPreconnection(LocalEndpoint,
                                     RemoteEndpoint,
                                     TransportProperties,
                                     SecurityParams)
~~~

The Local Endpoint MUST be specified if the Preconnection is used to Listen()
for incoming Connections, but is OPTIONAL if it is used to Initiate()
connections. The Remote Endpoint MUST be specified if the Preconnection is used
to Initiate() Connections, but is OPTIONAL if it is used to Listen() for
incoming Connections.
The Local Endpoint and the Remote Endpoint MUST both be specified if a
peer-to-peer Rendezvous is to occur based on the Preconnection.

Framers (see {{send-framing}}) and deframers (see {{receive-framing}}), if
necessary, should be bound to the Preconnection during pre-establishment.

## Specifying Endpoints {#endpointspec}

The transport services API uses the Local Endpoint and Remote Endpoint types
to refer to the endpoints of a transport connection.
Subtypes of these represent various different types of endpoint identifiers,
such as IP addresses, DNS names, and interface names, as well as port numbers
and service names.

~~~
RemoteSpecifier := NewRemoteEndpoint()
RemoteSpecifier.WithHostname("example.com")
RemoteSpecifier.WithService("https")
~~~

~~~
RemoteSpecifier := NewRemoteEndpoint()
RemoteSpecifier.WithIPv6Address(2001:db8:4920:e29d:a420:7461:7073:0a)
RemoteSpecifier.WithPort(443)
~~~

~~~
RemoteSpecifier := NewRemoteEndpoint()
RemoteSpecifier.WithIPv4Address(192.0.2.21)
RemoteSpecifier.WithPort(443)
~~~

~~~
LocalSpecifier := NewLocalEndpoint()
LocalSpecifier.WithInterface("en0")
LocalSpecifier.WithPort(443)
~~~

~~~
LocalSpecifier := NewLocalEndpoint()
LocalSpecifier.WithStunServer(address, port, credentials)
~~~

Implementations may also support additional endpoint representations and
provide a single NewEndpoint() call that takes different endpoint representations.

Multiple endpoint identifiers can be specified for each Local Endpoint and
Remote Endpoint.  For example, a Local Endpoint could be configured with two
interface names, or a Remote Endpoint could be specified via both IPv4 and
IPv6 addresses. These multiple identifiers refer to the same transport
endpoint.

The transport services API resolves names internally, when the Initiate(),
Listen(), or Rendezvous() method is called establish a Connection. The API
explicitly does not require the application to resolve names, though there is
a tradeoff between early and late binding of addresses to names. Early binding
allows the API implementation to reduce connection setup latency, at the cost
of potentially limited scope for alternate path discovery during Connection
establishment, as well as potential additional information leakage about
application interest when used with a resolution method (such as DNS without
TLS) which does not protect query confidentiality.

The Resolve() action on Preconnection can be used by the application to force
early binding when required, for example with some Network Address Translator
(NAT) traversal protocols (see {{rendezvous}}).

## Specifying Transport Properties {#selection-props}

A Preconnection Object holds properties reflecting the application's
requirements and preferences for the transport. These include Selection
Properties for selecting protocol stacks and paths, as well as Connection
Properties for configuration of the detailed operation of the selected Protocol
Stacks.

The protocol(s) and path(s) selected as candidates during establishment are
determined and configured using these properties. Since there could be paths
over which some transport protocols are unable to operate, or remote endpoints
that support only specific network addresses or transports, transport protocol
selection is necessarily tied to path selection. This may involve choosing
between multiple local interfaces that are connected to different access
networks.

Selection properties are represented as preferences, which can have one of five
preference levels:

   | Preference | Effect                                                                 |
   |------------|------------------------------------------------------------------------|
   | Require    | Select only protocols/paths providing the property, fail otherwise     |
   | Prefer     | Prefer protocols/paths providing the property, proceed otherwise       |
   | Ignore     | No preference                                                          |
   | Avoid      | Prefer protocols/paths not providing the property, proceed otherwise   |
   | Prohibit   | Select only protocols/paths not providing the property, fail otherwise |
   |------------|------------------------------------------------------------------------|

In addition, the pseudo-level ``Default`` can be used to reset the property to the default
level used by the implementation. This level will never show up when queuing the value of
a preference - the effective preference must be returned instead.

Internally, the transport system will first exclude all protocols and paths that
match a Prohibit, then exclude all protocols and paths that do not match a
Require, then sort candidates according to Preferred properties, and then use
Avoided properties as a tiebreaker. Selection Properties which select paths take
preference over those which select protocols. For example, if an application
indicates a preference for a specific path by specifying an interface, but also a
preference for a protocol not available on this path, the transport system will
try the path first, ignoring the preference.

Both Selection and Connection Properties can be added to a Preconnection to
configure the selection process, and to further configure the eventually
selected protocol stack(s). They are collected into a TransportProperties object
to be passed into a Preconnection object:

~~~
TransportProperties := NewTransportProperties()
~~~

Individual properties are then added to the TransportProperties Object:

~~~
TransportProperties.Add(property, value)
~~~

Selection Properties can be added to a TransportProperties object using special actions for each preference level i.e, `TransportProperties.Add(some_property, avoid)` is equivalent to `TransportProperties.Avoid(some_property)`:

~~~
TransportProperties.Require(property)
TransportProperties.Prefer(property)
TransportProperties.Ignore(property)
TransportProperties.Avoid(property)
TransportProperties.Prohibit(property)
TransportProperties.Default(property)
~~~

For an existing Connection, the Transport Properties can be queried any time
by using the following call on the Connection Object:

~~~
TransportProperties := Connection.GetTransportProperties()
~~~

A Connection gets its Transport Properties either by being explicitly configured
via a Preconnection, by configuration after establishment, or by inheriting them
from an antecedent via cloning; see {{groups}} for more.

{{connection-props}} provides a list of Connection Properties, while Selection
Properties are listed in the subsections below. Note that many properties are
only considered during establishment, and can not be changed after a Connection
is established; however, they can be queried. Querying a Selection Property
after establishment yields the value Required for properties of the selected
protocol and path, Avoid for properties avoided during selection, and Ignore for
all other properties.

An implementation of this interface must provide sensible defaults for Selection
Properties. The recommended defaults given for each property below represent a
configuration that can be implemented over TCP. An alternate set of default
Protocol Selection Properties would represent a configuration that can be
implemented over UDP.


### Reliable Data Transfer (Connection) {#prop-reliable}

This property specifies whether the application needs to use a transport
protocol that ensures that all data is received on the other side without
corruption. This also entails being notified when a Connection is closed or
aborted. The recommended default is to Require Reliable Data Transfer.

### Preservation of Message Boundaries {#prop-boundaries}

This property specifies whether the application needs or prefers to use a transport
protocol that preserves message boundaries. The recommended default
is to Prefer Preservation of Message Boundaries.

### Configure per-Message reliability {#prop-partially-reliable}

This property specifies whether an application considers it useful to indicate
its reliability requirements on a per-Message basis. This property applies to
Connections and Connection Groups. The recommended default is to Ignore
this option.

### Preservation of data ordering {#prop-ordering}

This property specifies whether the application wishes to use a transport
protocol that can ensure that data is received by the application on the other
end in the same order as it was sent. The recommended default is to Require
Preservation of data ordering.

### Use 0-RTT session establishment with an idempotent Message {#prop-0rtt}

This property specifies whether an application would like to supply a Message to
the transport protocol before Connection establishment, which will then be
reliably transferred to the other side before or during Connection
establishment, potentially multiple times. See also {{msg-idempotent}}. The
recommended default is to Prefer this option.

### Multistream Connections in Group {#prop-multistream}

This property specifies that the application would prefer multiple Connections
within a Connection Group to be provided by streams of a single underlying
transport connection where possible. The recommended default is to Prefer have
this option.

### Control checksum coverage on sending {#prop-checksum-control-send}

This property specifies whether the application considers it useful to enable,
disable, or configure a checksum when sending a Message.  The recommended default
is to Ignore this option.

### Control checksum coverage on receiving {#prop-checksum-control-receive}

This property specifies whether the application considers it useful configure whether to
require a checksum or not when receiving.  The recommended default is to Ignore
this option.

### Congestion control {#prop-cc}

This property specifies whether the application would like the Connection to be
congestion controlled or not. Note that if a Connection is not congestion
controlled, an application using such a Connection should itself perform
congestion control in accordance with {{?RFC2914}}. Also note that reliability
is usually combined with congestion control in protocol implementations,
rendering "reliable but not congestion controlled" a request that is unlikely to
succeed. The recommended default is to Require that the Connection is congestion
controlled.


### Interface Instance or Type {#prop-interface}

This property allows the application to select which specific network interfaces
or categories of interfaces it wants to `Require`, `Prohibit`, `Prefer`, or
`Avoid`.

In contrast to other Selection Properties, this property is a tuple of an
(Enumerated) interface identifier and a preference, and can either be
implemented directly as such, or for making one preference available for each
interface and interface type available on the system.

Note that marking a specific interface as `Required` strictly limits path
selection to a single interface, and leads to less flexible and resilient
connection establishment.

The set of valid interface types is implementation- and system-specific. For
example, on a mobile device, there may be `Wi-Fi` and `Cellular` interface types
available; whereas on a desktop computer, there may be `Wi-Fi` and `Wired
Ethernet` interface types available. Implementations should provide all types
that are supported on some system to all systems, in order to allow applications
to write generic code. For example, if a single implementation is used on both
mobile devices and desktop devices, it should define the `Cellular` interface
type for both systems, since an application may want to always `Prohibit
Cellular`. Note that marking a specific interface type as `Required` limits path
selection to a small set of interfaces, and leads to less flexible and resilient
connection establishment.

The set of interface types is expected to change over time as new access
technologies become available.

Interface types should not be treated as a proxy for properties of interfaces
such as metered or unmetered network access. If an application needs to prohibit
metered interfaces, this should be specified via Provisioning Domain attributes
(see {{prop-pvd}}) or another specific property.


### Provisioning Domain Instance or Type {#prop-pvd}

Similar to interface instances and types (see {{prop-interface}}), this property
allows the application to control path selection by selecting which specific
Provisioning Domains or categories of Provisioning Domains it wants to
`Require`, `Prohibit`, `Prefer`, or `Avoid`. Provisioning Domains define
consistent sets of network properties that may be more specific than network
interfaces {{RFC7556}}.

As with interface instances and types, this property is a tuple of an (Enumerated)
PvD identifier and a preference, and can either be implemented directly as such,
or for making one preference available for each interface and interface type
available on the system.

The identification of a specific Provisioning Domain (PvD) is defined to be
implementation- and system-specific, since there is not a portable standard
format for a PvD identitfier. For example, this identifier may be a string name
or an integer. As with requiring specific interfaces, requiring a specific PvD
strictly limits path selection.

Categories or types of PvDs are also defined to be implementation- and
system-specific. These may be useful to identify a service that is provided by a
PvD. For example, if an application wants to use a PvD that provides a
Voice-Over-IP service on a Cellular network, it can use the relevant PvD type to
require some PvD that provides this service, without needing to look up a
particular instance. While this does restrict path selection, it is broader than
requiring specific PvD instances or interface instances, and should be preferred
over these options.


### Parallel Use of Multiple Paths

This property specifies whether an application considers it useful to
transfer data across multiple paths between the same end hosts. Generally,
in most cases, this will improve performance (e.g., achieve greater throughput).
One possible side-effect is increased jitter, which may be problematic for
delay-sensitive applications. The recommended default is to Prefer this option.


### Direction of communication

This property specifies whether an application wants to use the connection for sending and/or receiving data.  Possible values are:

Bidirectional (default):
: The connection must support sending and receiving data

Unidirectional send:
: The connection must support sending data

Unidirectional receive:
: The connection must support receiving data

In case a unidirectional connection is requested, but unidirectional connections are not supported by the transport protocol,
the system should fall back to bidirectional transport.


### Notification of excessive retransmissions {#prop-establish-retrans-notify}

This property specifies whether an application considers it useful to be
informed in case sent data was retransmitted more often than a certain
threshold. The recommended default is to Ignore this option.


### Notification of ICMP soft error message arrival {#prop-soft-error}

This property specifies whether an application considers it useful to be
informed when an ICMP error message arrives that does not force termination of a
connection. When set to true, received ICMP errors will be available as
SoftErrors. Note that even if a protocol supporting this property is selected,
not all ICMP errors will necessarily be delivered, so applications cannot rely
on receiving them. The recommended default is to Ignore this option.


## Specifying Security Parameters and Callbacks {#security-parameters}

Most security parameters, e.g., TLS ciphersuites, local identity and private key, etc.,
may be configured statically. Others are dynamically configured during connection establishment.
Thus, we partition security parameters and callbacks based on their place in the lifetime
of connection establishment. Similar to Transport Properties, both parameters and callbacks
are inherited during cloning (see {{groups}}).

### Pre-Connection Parameters

Common parameters such as TLS ciphersuites are known to implementations. Clients should
use common safe defaults for these values whenever possible. However, as discussed in
{{I-D.ietf-taps-transport-security}}, many transport security protocols require specific
security parameters and constraints from the client at the time of configuration and
actively during a handshake. These configuration parameters are created as follows:

~~~
SecurityParameters := NewSecurityParameters()
~~~

Security configuration parameters and sample usage follow:

- Local identity and private keys: Used to perform private key operations and prove one's
identity to the Remote Endpoint. (Note, if private keys are not available, e.g., since they are
stored in hardware security modules (HSMs), handshake callbacks must be used. See below for details.)

~~~
SecurityParameters.AddIdentity(identity)
SecurityParameters.AddPrivateKey(privateKey, publicKey)
~~~

- Supported algorithms: Used to restrict what parameters are used by underlying transport security protocols.
When not specified, these algorithms should default to known and safe defaults for the system. Parameters include:
ciphersuites, supported groups, and signature algorithms.

~~~
SecurityParameters.AddSupportedGroup(secp256k1)
SecurityParameters.AddCiphersuite(TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256)
SecurityParameters.AddSignatureAlgorithm(ed25519)
~~~

- Session cache management: Used to tune cache capacity, lifetime, re-use,
and eviction policies, e.g., LRU or FIFO. Constants and policies for these interfaces
are implementation-specific.

~~~
SecurityParameters.SetSessionCacheCapacity(MAX_CACHE_ELEMENTS)
SecurityParameters.SetSessionCacheLifetime(SECONDS_PER_DAY)
SecurityParameters.SetSessionCachePolicy(CachePolicyOneTimeUse)
~~~

- Pre-Shared Key import: Used to install pre-shared keying material established
out-of-band. Each pre-shared keying material is associated with some identity that typically identifies
its use or has some protocol-specific meaning to the Remote Endpoint.

~~~
SecurityParameters.AddPreSharedKey(key, identity)
~~~

### Connection Establishment Callbacks

Security decisions, especially pertaining to trust, are not static. Once configured,
parameters may also be supplied during connection establishment. These are best
handled as client-provided callbacks. Security handshake callbacks that may be
invoked during connection establishment include:

- Trust verification callback: Invoked when a Remote Endpoint's trust must be validated before the
handshake protocol can proceed.

~~~
TrustCallback := NewCallback({
  // Handle trust, return the result
})
SecurityParameters.SetTrustVerificationCallback(trustCallback)
~~~

- Identity challenge callback: Invoked when a private key operation is required, e.g., when
local authentication is requested by a remote.

~~~
ChallengeCallback := NewCallback({
  // Handle challenge
})
SecurityParameters.SetIdentityChallengeCallback(challengeCallback)
~~~

# Establishing Connections {#establishment}

Before a Connection can be used for data transfer, it must be established.
Establishment ends the pre-establishment phase; all transport properties and
cryptographic parameter specification must be complete before establishment,
as these will be used to select candidate Paths and Protocol Stacks
for the Connection. Establishment may be active, using the Initiate() Action;
passive, using the Listen() Action; or simultaneous for peer-to-peer, using
the Rendezvous() Action. These Actions are described in the subsections below.

## Active Open: Initiate {#initiate}

Active open is the Action of establishing a Connection to a Remote Endpoint presumed
to be listening for incoming Connection requests. Active open is used by clients in
client-server interactions. Active open is supported by this interface through the
Initiate Action:

~~~
Connection := Preconnection.Initiate(timeout?)
~~~

The timeout parameter specifies how long to wait before aborting Active open.
Before calling Initiate, the caller must have populated a Preconnection
Object with a Remote Endpoint specifier, optionally a Local Endpoint
specifier (if not specified, the system will attempt to determine a
suitable Local Endpoint), as well as all properties
necessary for candidate selection.

The Initiate() Action consumes the Preconnection. Once Initiate() has
been called, no further properties may be added to the Preconnection, and
no subsequent establishment call may be made on the Preconnection.

Once Initiate is called, the candidate Protocol Stack(s) may cause one or more
candidate transport-layer connections to be created to the specified remote
endpoint. The caller may immediately begin sending Messages on the Connection
(see {{sending}}) after calling Initate(); note that any idempotent data sent
while the Connection is being established may be sent multiple times or on
multiple candidates.

The following Events may be sent by the Connection after Initiate() is called:

~~~
Connection -> Ready<>
~~~

The Ready Event occurs after Initiate has established a transport-layer
connection on at least one usable candidate Protocol Stack over at least one
candidate Path. No Receive Events (see {{receiving}}) will occur before
the Ready Event for Connections established using Initiate.

~~~
Connection -> InitiateError<>
~~~

An InitiateError occurs either when the set of transport properties and security
parameters cannot be fulfilled on a Connection for initiation (e.g. the set of
available Paths and/or Protocol Stacks meeting the constraints is empty) or
reconciled with the local and/or remote endpoints; when the remote specifier
cannot be resolved; or when no transport-layer connection can be established to
the remote endpoint (e.g. because the remote endpoint is not accepting
connections, the application is prohibited from opening a Connection by the
operating system, or the establishment attempt has timed out for any other reason).

See also {{initiate-and-send}} to combine Connection establishment
and transmission of the first message in a single action.

## Passive Open: Listen {#listen}

Passive open is the Action of waiting for Connections from remote endpoints,
commonly used by servers in client-server interactions. Passive open is
supported by this interface through the Listen Action:

~~~
Preconnection.Listen()
~~~

Before calling Listen, the caller must have initialized the Preconnection
during the pre-establishment phase with a Local Endpoint specifier, as well
as all properties necessary for Protocol Stack selection. A Remote Endpoint
may optionally be specified, to constrain what Connections are accepted.
The Listen() Action consumes the Preconnection. Once Listen() has been
called, no further properties may be added to the Preconnection, and no
subsequent establishment call may be made on the Preconnection.

Listening continues until the global context shuts down, or until the Stop
action is performed on the same Preconnection:

~~~
Preconnection.Stop()
~~~

After Stop() is called, the preconnection can be disposed of.

~~~
Preconnection -> ConnectionReceived<Connection>
~~~

The ConnectionReceived Event occurs when a Remote Endpoint has established a
transport-layer connection to this Preconnection (for Connection-oriented
transport protocols), or when the first Message has been received from the
Remote Endpoint (for Connectionless protocols), causing a new Connection to be
created. The resulting Connection is contained within the ConnectionReceived
event, and is ready to use as soon as it is passed to the application via the
event.

~~~
Preconnection -> ListenError<>
~~~

A ListenError occurs either when the Preconnection cannot be fulfilled for
listening, when the Local Endpoint (or Remote Endpoint, if specified) cannot
be resolved, or when the application is prohibited from listening by policy.

~~~
Preconnection -> Stopped<>
~~~

A Stopped event occurs after the Preconnection has stopped listening.

## Peer-to-Peer Establishment: Rendezvous {#rendezvous}

Simultaneous peer-to-peer Connection establishment is supported by the
Rendezvous() Action:

~~~
Preconnection.Rendezvous()
~~~

The Preconnection Object must be specified with both a Local Endpoint and a
Remote Endpoint, and also the transport properties and security parameters
needed for Protocol Stack selection.

The Rendezvous() Action causes the Preconnection to listen on the Local
Endpoint for an incoming Connection from the Remote Endpoint, while
simultaneously trying to establish a Connection from the Local Endpoint to the
Remote Endpoint. This corresponds to a TCP simultaneous open, for example.

The Rendezvous() Action consumes the Preconnection. Once Rendezvous() has
been called, no further properties may be added to the Preconnection, and
no subsequent establishment call may be made on the Preconnection.

~~~
Preconnection -> RendezvousDone<Connection>
~~~

The RendezvousDone<> Event occurs when a Connection is established with the
Remote Endpoint. For Connection-oriented transports, this occurs when the
transport-layer connection is established; for Connectionless transports,
it occurs when the first Message is received from the Remote Endpoint. The
resulting Connection is contained within the RendezvousDone<> Event, and is
ready to use as soon as it is passed to the application via the Event.

~~~
Preconnection -> RendezvousError<msgRef, error>
~~~

An RendezvousError occurs either when the Preconnection cannot be fulfilled
for listening, when the Local Endpoint or Remote Endpoint cannot be resolved,
when no transport-layer connection can be established to the Remote Endpoint,
or when the application is prohibited from rendezvous by policy.

When using some NAT traversal protocols, e.g., Interactive Connectivity
Establishment (ICE) {{?RFC5245}}, it is expected that the Local Endpoint will
be configured with some method of discovering NAT bindings, e.g., a Session
Traversal Utilities for NAT (STUN) server. In this case, the Local Endpoint
may resolve to a mixture of local and server reflexive addresses. The
Resolve() action on the Preconnection can be used to discover these bindings:

~~~
[]Preconnection := Preconnection.Resolve()
~~~

The Resolve() call returns a list of Preconnection Objects, that represent the
concrete addresses, local and server reflexive, on which a Rendezvous() for
the Preconnection will listen for incoming Connections. These resolved
Preconnections will share all other Properties with the Preconnection from
which they are derived, though some Properties may be made more-specific by
the resolution process. This list can be passed to a peer via a signalling
protocol, such as SIP {{?RFC3261}} or WebRTC {{?RFC7478}}, to configure the
remote.

## Connection Groups {#groups}

Groups of Connections can be created using the Clone Action:

~~~
Connection := Connection.Clone()
~~~

Calling Clone on a Connection yields a group of two Connections: the parent
Connection on which Clone was called, and the resulting cloned Connection. These
connections are "entangled" with each other, and become part of a Connection
Group. Calling Clone on any of these two Connections adds a third Connection to
the Connection Group, and so on. Connections in a Connection Group share all
Protocol Properties that are not applicable to a Message.

Changing one of these Protocol Properties on one Connection in the group changes it for all others. Per-Message Protocol Properties, however, are not entangled.
For example, changing "Timeout for aborting Connection" (see {{conn-timeout}}) on one Connection in a group will automatically change this Protocol Property for all Connections in the group in the same way. However, changing "Lifetime" (see {{msg-lifetime}}) of a Message will only affect a single Message on a single Connection, entangled or not.

If the underlying protocol supports multi-streaming, it is natural to use this functionality to implement Clone. In that case, entangled Connections are multiplexed together, giving them similar treatment not only inside endpoints but also across
the end-to-end Internet path.

If the underlying Protocol Stack does not support cloning, or cannot create a
new stream on the given Connection, then attempts to clone a connection will
result in a CloneError:

~~~
Connection -> CloneError<>
~~~

The Protocol Property "Priority" operates on entangled Connections as in {{msg-priority}}:
when allocating available network
capacity among Connections in a Connection Group, sends on Connections with
higher Priority values will be prioritized over sends on Connections with
lower Priority values. An ideal transport system implementation would assign
each Connection the capacity share (M-N) x C / M, where N is the Connection's
Priority value, M is the maximum Priority value used by all Connections in the
group and C is the total available capacity. However, the Priority setting is
purely advisory, and no guarantees are given about the way capacity is shared.
Each implementation is free to implement a way to share
capacity that it sees fit.

# Sending Data {#sending}

Once a Connection has been established, it can be used for sending data. Data is
sent in terms of Messages, which allow the application to communicate the boundaries
of the data being transferred. By default, Send enqueues a complete Message,
and takes optional per-Message properties (see {{send-basic}}). All Send actions
are asynchronous, and deliver events (see {{send-events}}). Sending partial
Messages for streaming large data is also supported (see {{send-partial}}).

Messages are sent on a Connection using the Send action:

~~~
Connection.Send(messageData, messageContext?, endOfMessage?)
~~~

where messageData is the data object to send. The optional messageContext
parameter supports per-message properties and is described in {{message-props}}.
The optional endOfMessage parameter supports partial sending and is described in
{{send-partial}}.

## Basic Sending {#send-basic}

The most basic form of sending on a connection involves enqueuing a single Data
block as a complete Message, with default Message Properties. Message data is
created as an array of octets, and the resulting object contains both the byte
array and the length of the array.

~~~
messageData := "hello".octets()
Connection.Send(messageData)
~~~

The interpretation of a Message to be sent is dependent on the implementation, and
on the constraints on the Protocol Stacks implied by the Connection’s transport properties.
For example, a Message may be a single datagram for UDP Connections; or an HTTP
Request for HTTP Connections.

Some transport protocols can deliver arbitrarily sized Messages, but other
protocols constrain the maximum Message size. Applications can query the
protocol property Maximum Message Size on Send to determine the maximum size
allowed for a single Message. If a Message is too large to fit in the Maximum Message
Size for the Connection, the Send will fail with a SendError event ({{send-error}}). For
example, it is invalid to send a Message over a UDP connection that is larger than
the available datagram sending size.

## Send Events {#send-events}

Like all Actions in this interface, the Send Action is asynchronous. There are
several events that can be delivered in response to Sending a Message.

Note that if partial Sends are used ({{send-partial}}), there will still be exactly
one Send Event delivered for each call to Send. For example, if a Message
expired while two requests to Send data for that Message are outstanding,
there will be two Expired events delivered.

### Sent

~~~
Connection -> Sent<msgRef>
~~~

The Sent Event occurs when a previous Send Action has completed, i.e., when
the data derived from the Message has been passed down or through the
underlying Protocol Stack and is no longer the responsibility of the
implementation of this interface. The exact disposition of the Message (i.e.,
whether it has actually been transmitted, moved into a buffer on the network
interface, moved into a kernel buffer, and so on) when the Sent Event occurs
is implementation-specific. The Sent Event contains an implementation-specific
reference to the Message to which it applies.

Sent Events allow an application to obtain an understanding of the amount
of buffering it creates. That is, if an application calls the Send Action multiple
times without waiting for a Sent Event, it has created more buffer inside the
transport system than an application that always waits for the Sent Event before
calling the next Send Action.

### Expired

~~~
Connection -> Expired<msgRef>
~~~

The Expired Event occurs when a previous Send Action expired before completion;
i.e. when the Message was not sent before its Lifetime (see {{msg-lifetime}})
expired. This is separate from SendError, as it is an expected behavior for
partially reliable transports. The Expired Event contains an
implementation-specific reference to the Message to which it applies.

### SendError {#send-error}

~~~
Connection -> SendError<msgRef>
~~~

A SendError occurs when a Message could not be sent due to an error condition:
an attempt to send a Message which is too large for the system and
Protocol Stack to handle, some failure of the underlying Protocol Stack, or a
set of Message Properties not consistent with the Connection's transport
properties. The SendError contains an implementation-specific reference to the
Message to which it applies.

## Message Properties {#message-props}

Applications may need to annotate the Messages they send with extra information
to control how data is scheduled and processed by the transport protocols in the
Connection. A MessageContext object contains properties for sending Messages,
and can be passed to the Send Action. Note that these properties are
per-Message, not per-Send if partial Messages are sent ({{send-partial}}). All
data blocks associated with a single Message share properties. For example, it
would not make sense to have the beginning of a Message expire, but allow the
end of a Message to still be sent.

~~~
messageData := "hello".octets()
messageContext := NewMessageContext()
messageContext.add(parameter, value)
Connection.Send(messageData, messageContext)
~~~

The simpler form of Send that does not take any messageContext is equivalent
to passing a default MessageContext with not values added.

If an application wants to override Message Properties for a specific message,
it can acquire an empty MessageContext Object and add all desired Message
Properties to that Object. It can then reuse the same messageContext Object
for sending multiple Messages with the same properties.

Properties may be added to a MessageContext object only before the context is used
for sending. Once a messageContext has been used with a Send call, modifying any
of its properties is invalid.

Message Properties may be inconsistent with the properties of the Protocol Stacks
underlying the Connection on which a given Message is sent. For example,
a Connection must provide reliability to allow setting an infinitie value for the
lifetime property of a Message. Sending a Message with Message Properties
inconsistent with the Selection Properties of the Connection yields an error.

The following Message Properties are supported:

### Lifetime {#msg-lifetime}

Type:
: Integer

Lifetime specifies how long a particular Message can wait to be sent to the
remote endpoint before it is irrelevant and no longer needs to be
(re-)transmitted. This is a hint to the transport system -- it is not guaranteed
that a Message will not be sent when its Lifetime has expired.

Setting a Message's Lifetime to infinite indicates that the application does
not wish to apply a time constraint on the transmission of the Message, but it does not express a need for
reliable delivery; reliability is adjustable per Message via the "Reliable Data Transfer (Message)"
property (see {{msg-reliable-message}}). The type and units of Lifetime are implementation-specific.

### Priority {#msg-priority}

Type:
: Integer (non-negative)

This property represents a hierarchy of priorities.
It can specify the priority of a Message, relative to other Messages sent over the
same Connection.

A Message with Priority 0 will yield to a Message with Priority 1, which will
yield to a Message with Priority 2, and so on. Priorities may be used as a
sender-side scheduling construct only, or be used to specify priorities on the
wire for Protocol Stacks supporting prioritization.

Note that this property is not a per-message override of the connection Priority
- see {{conn-priority}}. Both Priority properties may interact, but can be used
independently and be realized by different mechanisms.

### Ordered {#msg-ordered}

Type:
: Boolean

If true, it specifies that the receiver-side transport protocol stack only deliver the Message to the receiving application after the previous ordered Message which was passed to the same Connection via the Send
Action, when such a Message exists. If false, the Message may be delivered to the receiving application out of order.
This property is used for protocols that support preservation of data ordering,
see {{prop-ordering}}, but allow out-of-order delivery for certain messages.


### Idempotent {#msg-idempotent}

Type:
: Boolean

If true, it specifies that a Message is safe to send to the remote endpoint
more than once for a single Send Action. It is used to mark data safe for
certain 0-RTT establishment techniques, where retransmission of the 0-RTT data
may cause the remote application to receive the Message multiple times.

### Final {#msg-final}

Type:
: Boolean

If true, this Message is the last one that
the application will send on a Connection. This allows underlying protocols
to indicate to the Remote Endpoint that the Connection has been effectively
closed in the sending direction. For example, TCP-based Connections can
send a FIN once a Message marked as Final has been completely sent,
indicated by marking endOfMessage. Protocols that do not support signalling
the end of a Connection in a given direction will ignore this property.

Note that a Final Message must always be sorted to the end of a list of Messages.
The Final property overrides Priority and any other property that would re-order
Messages. If another Message is sent after a Message marked as Final has already
been sent on a Connection, the Send Action for the new Message will cause a SendError Event.

### Corruption Protection Length {#msg-checksum}

Type:
: Integer (non-negative with -1 as special value)

This property specifies the length of the section of the Message,
starting from byte 0, that the application requires to be delivered without
corruption due to lower layer errors. It is used to specify options for simple
integrity protection via checksums. By default, the entire Message is protected
by a checksum. A value of 0 means that no checksum is required, and a special
value (e.g. -1) can be used to indicate the default. Only full coverage is
guaranteed, any other requests are advisory.

### Reliable Data Transfer (Message) {#msg-reliable-message}

Type:
: Boolean

When true, this property specifies that a message should be sent in such a way
that the transport protocol ensures all data is received on the other side
without corruption. Changing the ´Reliable Data Transfer´ property on Messages
is only possible for Connections that were established with the Selection Property 'Reliable Data Transfer (Connection)' enabled.
When this is not the case, changing it will generate an error.
Disabling this property indicates that the transport system may disable retransmissions
or other reliability mechanisms for this particular Message, but such disabling is not guaranteed.


### Transmission Profile {#send-profile}

Type:
: Enumeration

This enumerated property specifies the application's preferred tradeoffs for
sending this Message; it is a per-Message override of the Capacity Profile
protocol and path selection property (see {{prop-cap-profile}}).

The following values are valid for Transmission Profile:

  Default:
  :  No special optimizations of the tradeoff between delay, delay
  variation, and bandwidth efficiency should be made when sending this message.

  Low Latency:
  : Response time (latency) should be optimized at
  the expense of efficiently using the available capacity when sending this
  message. This can be used by the system to disable the coalescing of
  multiple small Messages into larger packets (Nagle's algorithm); to prefer
  immediate acknowledgment from the peer endpoint when supported by the
  underlying transport; to signal a preference for lower-latency, higher-loss
  treatment; and so on.

\[TODO: This is inconsistent with {prop-cap-profile}} - needs to be fixed]

### Singular Transmission {#send-singular}

Type:
: Boolean

This property specifies that a message should be sent and received as a single
packet without transport-layer segmentation or network-layer fragmentation.
Attempts to send a message with this property set with a size greater to the
transport's current estimate of its maximum transmission segment size will
result in a `SendError`. When used with transports supporting this functionality
and running over IP version 4, the Don't Fragment bit will be set.

## Partial Sends {#send-partial}

It is not always possible for an application to send all data associated with
a Message in a single Send Action. The Message data may be too large for
the application to hold in memory at one time, or the length of the Message
may be unknown or unbounded.

Partial Message sending is supported by passing an endOfMessage boolean
parameter to the Send Action. This value is always true by default, and
the simpler forms of Send are equivalent to passing true for endOfMessage.

The following example sends a Message in two separate calls to Send.

~~~
messageContext := NewMessageContext()
messageContext.add(parameter, value)

messageData := "hel".octets()
endOfMessage := false
Connection.Send(messageData, messageContext, endOfMessage)

messageData := "lo".octets()
endOfMessage := true
Connection.Send(messageData, messageContext, endOfMessage)
~~~

All data sent with the same MessageContext object will be treated as belonging
to the same Message, and will constitute an in-order series until the
endOfMessage is marked. Once the end of the Message is marked, the
MessageContext object may be re-used as a new Message with identical parameters.

## Batching Sends {#send-batching}

In order to reduce the overhead of sending multiple small Messages on a Connection, the
application may want to batch several Send actions together. This provides a hint to
the system that the sending of these Messages should be coalesced when possible,
and that sending any of the batched Messages may be delayed until the last Message
in the batch is enqueued.

~~~
Connection.Batch(
    Connection.Send(messageData)
    Connection.Send(messageData)
)
~~~

## Send on Active Open: InitiateWithSend {#initiate-and-send}

For application-layer protocols where the Connection initiator also sends the
first message, the InitiateWithSend() action combines Connection initiation with
a first Message sent:

~~~
Connection := Preconnection.InitiateWithSend(messageData, messageContext?, timeout?)
~~~

Whenever possible, a messageContext should be provided to declare the message passed to InitiateWithSend
as idempotent. This allows the transport system to make use of 0-RTT establishment in case this is supported
by the available protocol stacks. When the selected stack(s) do not support transmitting data upon connection
establishment, InitiateWithSend is identical to Initiate() followed by Send().

Neither partial sends nor send batching are supported by InitiateWithSend().

The Events that may be sent after InitiateWithSend() are equivalent to those
that would be sent by an invocation of Initate() followed immediately by an
invocation of Send(), with the caveat that a send failure that occurs because
the Connection could not be established will not result in a
SendError separate from the InitiateError signaling the failure of Connection
establishment.

## Sender-side Framing {#send-framing}

Sender-side framing allows a caller to provide the interface with a function
that takes a Message of an appropriate application-layer type and returns an
array of octets, the on-the-wire representation of the Message to be handed down
to the Protocol Stack. It consists of a Framer Object with a single Action,
Frame. Since the Framer depends on the protocol used at the application layer,
it is bound to the Preconnection during the pre-establishment phase:

~~~
Preconnection.FrameWith(Framer)

OctetArray := Framer.Frame(messageData)
~~~

Sender-side framing is a convenience feature of the interface, for parity with
receiver-side framing (see {{receive-framing}}).

# Receiving Data {#receiving}

Once a Connection is established, it can be used for receiving data. As with
sending, data is received in terms of Messages. Receiving is an asynchronous
operation, in which each call to Receive enqueues a request to receive new
data from the connection. Once data has been received, or an error is encountered,
an event will be delivered to complete the Receive request (see {{receive-events}}).

As with sending, the type of the Message to be passed is dependent on the
implementation, and on the constraints on the Protocol Stacks implied by the
Connection's transport parameters.

## Enqueuing Receives

Receive takes two parameters to specify the length of data that an application
is willing to receive, both of which are optional and have default values if not
specified.

~~~
Connection.Receive(minIncompleteLength?, maxLength?)
~~~

By default, Receive will try to deliver complete Messages in a single event ({{receive-complete}}).

The application can set a minIncompleteLength value to indicates the smallest partial
Message data size in bytes that should be delivered in response to this Receive. By default,
this value is infinite, which means that only complete Messages should be delivered (see {{receive-partial}} and {{receive-framing}} for more information on how this is accomplished).
If this value is set to some smaller value, the associated receive event will be triggered
only when at least that many bytes are available, or the Message is complete with fewer
bytes, or the system needs to free up memory. Applications should always
check the length of the data delivered to the receive event and not assume
it will be as long as minIncompleteLength in the case of shorter complete Messages
or memory issues.

The maxLength argument indicates the maximum size of a Message in bytes
the application is currently prepared to receive. The default
value for maxLength is infinite. If an incoming Message is larger than the
minimum of this size and the maximum Message size on receive for
the Connection's Protocol Stack, it will be delivered via ReceivedPartial
events ({{receive-partial}}).

Note that maxLength does not guarantee that the application will receive that many
bytes if they are available; the interface may return ReceivedPartial events with less
data than maxLength according to implementation constraints.

## Receive Events {#receive-events}

Each call to Receive will be paired with a single Receive Event, which can be a success
or an error. This allows an application to provide backpressure to the transport stack
when it is temporarily not ready to receive messages.

### Received {#receive-complete}

~~~
Connection -> Received<messageData, messageContext>
~~~

A Received event indicates the delivery of a complete Message. It contains two objects,
the received bytes as messageData, and the metadata and properties of the received
Message as messageContext. See {{receive-context}} for details about the received context.

The messageData object provides access to the bytes that were received for this Message,
along with the length of the byte array.

See {{receive-framing}} for handling Message framing in situations where the Protocol
Stack provides octet-stream transport only.

### ReceivedPartial {#receive-partial}

~~~
Connection -> ReceivedPartial<messageData, messageContext, endOfMessage>
~~~

If a complete Message cannot be delivered in one event, one part of the Message
may be delivered with a ReceivedPartial event. In order to continue to receive more
of the same Message, the application must invoke Receive again.

Multiple invocations of ReceivedPartial deliver data for the same Message by
passing the same MessageContext, until the endOfMessage flag is delivered or a
ReceiveError occurs. All partial blocks of a single Message are delivered in
order without gaps. This event does not support delivering discontiguous partial
Messages.

If the minIncompleteLength in the Receive request was set to be infinite (indicating
a request to receive only complete Messages), the ReceivedPartial event may still be
delivered if one of the following conditions is true:

* the underlying Protocol Stack supports message boundary preservation, and
  the size of the Message is larger than the buffers available for a single
  message;
* the underlying Protocol Stack does not support message boundary
  preservation, and the deframer (see {{receive-framing}}) cannot determine
  the end of the message using the buffer space it has available; or
* the underlying Protocol Stack does not support message boundary
  preservation, and no deframer was supplied by the application

Note that in the absence of message boundary preservation or
deframing, all bytes received on the Connection will be represented as one
large message of indeterminate length.

### ReceiveError

~~~
Connection -> ReceiveError<messageContext>
~~~

A ReceiveError occurs when data is received by the underlying Protocol Stack
that cannot be fully retrieved or deframed, or when some other indication is
received that reception has failed. Such conditions that irrevocably lead to
the termination of the Connection are signaled using ConnectionError instead
(see {{termination}}).

The ReceiveError event passes an optional associated MessageContext. This may
indicate that a Message that was being partially received previously, but had not
completed, encountered an error and will not be completed.

## Message Receive Context {#receive-context}

Each Received Message Context may contain metadata from protocols in the Protocol Stack;
which metadata is available is Protocol Stack dependent. The following metadata
values are supported:

### ECN {#receive-ecn}

When available, Message metadata carries the value of the Explicit Congestion
Notification (ECN) field. This information can be used for logging and debugging
purposes, and for building applications which need access to information about
the transport internals for their own operation.

### Early Data {#receive-early}

In some cases it may be valuable to know whether data was read as part of early
data transfer (before connection establishment has finished). This is useful if
applications need to treat early data separately,
e.g., if early data has different security properties than data sent after
connection establishment. In the case of TLS 1.3, client early data can be replayed
maliciously (see {{!I-D.ietf-tls-tls13}}). Thus, receivers may wish to perform additional
checks for early data to ensure it is idempotent or not replayed. If TLS 1.3 is available
and the recipient Message was sent as part of early data, the corresponding metadata carries
a flag indicating as such. If early data is enabled, applications should check this metadata
field for Messages received during connection establishment and respond accordingly.

### Receiving Final Messages

The Received Message Context can indicate whether or not this Message is
the Final Message on a Connection. For any Message that is marked as Final,
the application can assume that there will be no more Messages received on the
Connection once the Message has been completely delivered. This corresponds
to the Final property that may be marked on a sent Message {{msg-final}}.

Some transport protocols and peers may not support signaling of the Final property.
Applications therefore should not rely on receiving a Message marked Final to know
that the other endpoint is done sending on a connection.

Any calls to Receive once the Final Message has been delivered will result in errors.

## Receiver-side De-framing over Stream Protocols {#receive-framing}

The Receive Event is intended to be fired once per application-layer Message
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
next Message to deframe. It consists of a Deframer Object with a single Action,
Deframe. Since the Deframer depends on the protocol used at the application
layer, it is bound to the Preconnection during the pre-establishment phase:

~~~
Preconnection.DeframeWith(Deframer)

{messageData} := Deframer.Deframe(OctetStream)
~~~

# Managing Connections {#introspection}

During pre-establishment and after establishment, connections can be configured and queried using Connection
Properties, and asynchronous information may be available about the state of the
connection via Soft Errors.

Connection Properties represent the configuration and state of the selected
Protocol Stack(s) backing a Connection. These Connection Properties may be
Generic, applying regardless of transport protocol, or Specific, applicable to a
single implementation of a single transport protocol stack. Generic Connection
Properties are defined in {{connection-props}} below. Specific Protocol
Properties are defined in a transport- and implementation-specific way, and must
not be assumed to apply across different protocols. Attempts to set Specific
Protocol Properties on a protocol stack not containing that specific protocol
are simply ignored, and do not raise an error; however, too much reliance by an
application on Specific Protocol Properties may significantly reduce the
flexibility of a transport services implementation.

The application can set and query Connection Properties on a per-Connection
basis. Connection Properties that are not read-only can be set during
pre-establishment (see {{selection-props}}), as well as on connections directly using
the SetProperty action:

~~~
Connection.SetProperty(property, value)
~~~

At any point, the application can query Connection Properties.

~~~
ConnectionProperties := Connection.GetProperties()
~~~

Depending on the status of the connection, the queried Connection
Properties will include different information:

* The connection state, which can be one of the following:
  Establishing, Established, Closing, or Closed.

* Whether the connection can be used to send data. A connection can not be used
  for sending if the connection was created with the Selection Property
  "Direction of Communication" set to "unidirectional receive" or if a Message
  marked as "Final" was sent over this connection, see {{msg-final}}.

* Whether the connection can be used to receive data. A connection can not be
  used for reading if the connection was created with the Selection Property
  "Direction of Communication" set to "unidirectional send" or if a Message
  marked as "Final" was received, see {{receiving-final-messages}}. The latter
  is only supported by certain transport protocols, e.g., by TCP as half-closed
  connection.

* For Connections that are Establishing: Transport Properties that the
  application specified on the Preconnection, see {{selection-props}}.

* For Connections that are Established, Closing, or Closed: Selection
  ({{selection-props}}) and Connection Properties ({{connection-props}}) of the
  actual protocols that were selected and instantiated. Selection Properties
  indicate whether or not the Connection has or offers a certain Selection
  Property. Note that the actually instantiated protocol stack may not match all
  Protocol Selection Properties that the application specified on the
  Preconnection. For example, a certain Protocol Selection Property that an
  application specified as Preferred may not actually be present in the chosen
  protocol stack because none of the currently available transport protocols had
  this feature.

* For Connections that are Established, additional properties of the path(s) in
  use. These properties can be derived from the local provisioning domain
  {{RFC7556}}, measurements by the Protocol Stack, or other sources.


## Generic Connection Properties {#connection-props}

The Connection Properties defined as independent, and available on all
Connections are defined in the subsections below. 

Note that many protocol properties have a corresponding selection property, which
prefers protocols providing a specific transport feature that controlled by
that protocol property. \[EDITOR'S NOTE: todo: add these cross-references up to {{selection-props}}]


### Retransmission threshold before excessive retransmission notification {#conn-excss-retransmit}

Type:
: Integer

This property specifies after how many retransmissions to inform the application
about "Excessive Retransmissions".


### Required minimum coverage of the checksum for receiving {#conn-recv-checksum}

Type:
: Integer

This property specifies the part of the received data that needs
to be covered by a checksum. It is given in Bytes. A value of 0 means
that no checksum is required, and a special value (e.g., -1) indicates
full checksum coverage.

### Priority (Connection) {#conn-priority}

Type:
: Integer

This Property is a non-negative integer representing the relative inverse
priority of this Connection relative to other Connections in the same
Connection Group. It has no effect on Connections not part of a Connection
Group. As noted in {{groups}}, this property is not entangled when Connections
are cloned.

### Timeout for aborting Connection {#conn-timeout}

This property specifies how long to wait before deciding that a Connection has
failed when trying to reliably deliver data to the destination. Adjusting this Property
will only take effect when the underlying stack supports reliability.


### Connection group transmission scheduler {#conn-scheduler}

Type:
: Enum

This property specifies which scheduler should be used among Connections within
a Connection Group, see {{groups}}. The set of schedulers can
be taken from {{I-D.ietf-tsvwg-sctp-ndata}}.

### Maximum message size concurrent with Connection establishment {#size-idempotent}

Type:
: Integer (read only)

This property represents the maximum Message size that can be sent
before or during Connection establishment, see also {{msg-idempotent}}.
It is given in Bytes. 

### Maximum Message size before fragmentation or segmentation {#conn-max-msg-notfrag}

Type:
: Integer (read only)

This property, if applicable, represents the maximum Message size that can be
sent without incurring network-layer fragmentation or transport layer
segmentation at the sender. 

### Maximum Message size on send {#conn-max-msg-send}

Type:
: Integer (read only)

This property represents the maximum Message size that can be sent. 


### Maximum Message size on receive {#conn-max-msg-recv}

Type:
: Integer (read only)

This numeric property represents the maximum Message size that can be received.

### Capacity Profile {#prop-cap-profile}

This property specifies the desired network treatment for traffic sent by the
application and the tradeoffs the application is prepared to make in path and
protocol selection to receive that desired treatment. When the capacity profile
is set to a value other than Default, the transport system should select paths
and profiles to optimize for the capacity profile specified. The following
values are valid for the Capacity Profile:

  Default: 
  : The application makes no representation about its expected capacity
  profile. No special optimizations of the tradeoff between delay, delay
  variation, and bandwidth efficiency should be made when selecting and
  configuring transport protocol stacks. Transport system implementations that
  map the requested capacity profile onto per-connection DSCP signaling without
  multiplexing SHOULD assign the DSCP Default Forwarding {{?RFC2474}} PHB; when
  the Connection is multiplexed, the guidelines in section 6 of {{?RFC7657}}
  apply.

  Scavenger: 
  : The application is not interactive. It expects to send
  and/or receive data without any urgency. This can, for example, be used to
  select protocol stacks with scavenger transmission control and/or to assign
  the traffic to a lower-effort service. Transport system implementations that
  map the requested capacity profile onto per-connection DSCP signaling without
  multiplexing SHOULD assign the DSCP Less than Best Effort
  {{?LE-PHB=I-D.ietf-tsvwg-le-phb}} PHB; when the Connection is multiplexed, the
  guidelines in section 6 of {{?RFC7657}} apply.

  Low Latency/Interactive: 
  : The application is interactive, and prefers loss to
  latency. Response time should be optimized at the expense of bandwidth
  efficiency and delay variation when sending on this connection. This can be
  used by the system to disable the coalescing of multiple small Messages into
  larger packets (Nagle's algorithm); to prefer immediate acknowledgment from
  the peer endpoint when supported by the underlying transport; and so on.
  Transport system implementations that map the requested capacity profile onto
  per-connection DSCP signaling without multiplexing SHOULD assign the DSCP
  Expedited Forwarding {{?RFC3246}} PHB; when the Connection is multiplexed, the
  guidelines in section 6 of {{?RFC7657}} apply.

  Low Latency/Non-Interactive: 
  : The application prefers loss to latency but is
  not interactive. Response time should be optimized at the expense of bandwidth
  efficiency and delay variation when sending on this connection.Transport
  system implementations that map the requested capacity profile onto
  per-connection DSCP signaling without multiplexing SHOULD assign a DSCP
  Assured Forwarding (AF21,AF22,AF23,AF24) {{?RFC2597}} PHB; when the Connection
  is multiplexed, the guidelines in section 6 of {{?RFC7657}} apply.

  Constant-Rate Streaming: 
  : The application expects to send/receive data at a
  constant rate after Connection establishment. Delay and delay variation should
  be minimized at the expense of bandwidth efficiency. This implies that the
  Connection may fail if the desired rate cannot be maintained across the Path.
  A transport may interpret this capacity profile as preferring a circuit
  breaker {{?RFC8084}} to a rate-adaptive congestion controller. Transport
  system implementations that map the requested capacity profile onto
  per-connection DSCP signaling without multiplexing SHOULD assign a DSCP
  Assured Forwarding (AF31,AF32,AF33,AF34) {{?RFC2597}} PHB; when the Connection
  is multiplexed, the guidelines in section 6 of {{?RFC7657}} apply.

  High Throughput Data: 
  : The application expects to send/receive data at the
  maximum rate allowed by its congestion controller over a relatively long
  period of time. Transport system implementations that map the requested
  capacity profile onto per-connection DSCP signaling without multiplexing
  SHOULD assign a DSCP Assured Forwarding (AF11,AF12,AF13,AF14) {{?RFC2597}} PHB
  per section 4.8 of {{?RFC4594}}. When the Connection is multiplexed, the
  guidelines in section 6 of {{?RFC7657}} apply.

The Capacity Profile for a selected protocol stack may be modified on a
per-Message basis using the Transmission Profile Message Property; see
{{send-profile}}.


### Bounds on Send or Receive Rate

This property specifies an upper-bound rate that a transfer is not expected to
exceed (even if flow control and congestion control allow higher rates), and/or a
lower-bound rate below which the application does not deem
a data transfer useful. It is given in bits per second.


### TCP-specific Property: User Timeout

This property specifies, for the case TCP becomes the chosen transport protocol:

Advertised User Timeout:
: a time value to be advertised via the User Timeout Option (UTO) for the TCP at the remote endpoint
to adapt its own "Timeout for aborting Connection" (see {{conn-timeout}}) value accordingly

User Timeout Enabled:
: a boolean (default false) to control whether the UTO option is enabled for a
connection. This applies to both sending and receiving.

Changeable:
: a boolean (default true) which controls whether the "Timeout for aborting Connection" (see {{conn-timeout}})
may be changed
based on a UTO option received from the remote peer. This boolean becomes false when
"Timeout for aborting Connection" (see {{conn-timeout}}) is used.

All of the above parameters are optional (e.g., it is possible to specify "User Timeout Enabled" as true,
but not specify an Advertised User Timeout value; in this case, the TCP default will be used).


## Soft Errors

Asynchronous introspection is also possible, via the SoftError Event. This event
informs the application about the receipt of an ICMP error message related to
the Connection. This will only happen if the underlying protocol stack supports
access to soft errors; however, even if the underlying stack supports it, there
is no guarantee that a soft error will be signaled.

~~~
Connection -> SoftError<>
~~~

## Excessive retransmissions {#conn-retrans-notify}

This event notifies the application of excessive retransmissions, based on a
configured threshold (see {{conn-excss-retransmit}}). This will only happen if
the underlying protocol stack supports reliability and, with it, such notifications.

~~~
Connection -> ExcessiveRetransmission<>
~~~


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

The Closed Event can inform the application that the Remote Endpoint has closed the
Connection; however, there is no guarantee that a remote Close will indeed be
signaled.

~~~
Connection -> Closed<>
~~~

Abort terminates a Connection without delivering remaining data:

~~~
Connection.Abort()
~~~

A ConnectionError informs the application that data to could not be delivered after a timeout, 
or the other side has aborted the Connection; however, there is no guarantee that an Abort will indeed be signaled.

~~~
Connection -> ConnectionError<>
~~~


# Connection State and Ordering of Operations and Events

As this interface is designed to be independent of an implementation's
concurrency model, the
details of how exactly actions are handled, and on which threads/callbacks
events are dispatched, are implementation dependent. 

Each transition of connection state is associated with one of more events:

- Ready<> occurs when a Connection created with Initiate() or
  InitiateWithSend() transitions to Established state.

- ConnectionReceived<> occurs when a Connection created with Listen()
  transitions to Established state.

- RendezvousDone<> occurs when a Connection created with Rendezvous()
  transitions to Established state.

- Closed<> occurs when a Connection transitions to Closed state without error.

- InitiateError<> occurs when a Connection created with Initiate() transitions
  from Establishing state to Closed state due to an error.

- ConnectionError<> occurs when a Connection transitions to Closed state due to
  an error in all other circumstances.

The interface provides the following guarantees about the ordering of
 operations:

- Sent<> events will occur on a Connection in the order in which the Messages
  were sent (i.e., delivered to the kernel or to the network interface,
  depending on implementation).

- Received<> will never occur on a Connection before it is Established; i.e.
  before a Ready<> event on that Connection, or a ConnectionReceived<> or
  RendezvousDone<> containing that Connection.

- No events will occur on a Connection after it is Closed; i.e., after a
  Closed<> event, an InitiateError<> or ConnectionError<> on that connection. To
  ensure this ordering, Closed<> will not occur on a Connection while other
  events on the Connection are still locally outstanding (i.e., known to the
  interface and waiting to be dealt with by the application). ConnectionError<>
  may occur after Closed<>, but the interface must gracefully handle all cases
  where application ignores these errors.


# IANA Considerations

RFC-EDITOR: Please remove this section before publication.

This document has no Actions for IANA.

# Security Considerations

This document describes a generic API for interacting with a transport services (TAPS) system.
Part of this API includes configuration details for transport security protocols, as discussed
in {{security-parameters}}. It does not recommend use (or disuse) of specific
algorithms or protocols. Any API-compatible transport security protocol should work in a TAPS system.

# Acknowledgements

This work has received funding from the European Union's Horizon 2020 research and
innovation programme under grant agreements No. 644334 (NEAT) and No. 688421 (MAMI).

This work has been supported by Leibniz Prize project funds of DFG - German
Research Foundation: Gottfried Wilhelm Leibniz-Preis 2011 (FKZ FE 570/4-1).

This work has been supported by the UK Engineering and Physical Sciences
Research Council under grant EP/R04144X/1.

Thanks to Stuart Cheshire, Josh Graessley, David Schinazi, and Eric Kinnear for
their implementation and design efforts, including Happy Eyeballs, that heavily
influenced this work. Thanks to Laurent Chuat and Jason Lee for initial work on
the Post Sockets interface, from which this work has evolved.

--- back

# Additional Properties {#appendix-non-consensus}

The interface specified by this document represents the minimal common interface
to an endpoint in the transport services architecture {{I-D.ietf-taps-arch}}, based upon
that architecture and on the minimal set of transport service features
elaborated in {{I-D.ietf-taps-minset}}. However, the interface has been designed with
extension points to allow the implementation of features beyond those in the
minimal common interface: Protocol Selection Properties, Path Selection
Properties, and Message Properties are open sets. Implementations of the
interface are free to extend these sets to provide additional expressiveness to
applications written on top of them.

This appendix enumerates a few additional properties that could
be used to enhance transport protocol and/or path selection, or the transmission
of messages given a Protocol Stack that implements them. These are not part of
the interface, and may be removed from the final document, but are presented
here to support discussion within the TAPS working group as to whether they
should be added to a future revision of the base specification.

## Experimental Transport Properties

The following Transport Properties might be made available in
addition to those specified in {{selection-props}}, {{connection-props}}, and {{message-props}}.


### Cost Preferences {#cost-preferences}

\[EDITOR'S NOTE: At IETF 103, opinions were that this property should stay,
but it was also said that this is maybe not "on the right level".
If / when moving it to the main text, note that this is meant to be
applicable to a Preconnection or a Message.]

This property describes what an application prefers regarding monetary costs,
e.g., whether it considers it acceptable to utilize limited data volume. It
provides hints to the transport system on how to handle trade-offs between cost
and performance or reliability.

Possible values are:

No Expense:
: Avoid transports associated with monetary cost

Optimize Cost:
: Prefer inexpensive transports and accept service degradation

Balance Cost:
: Use system policy to balance cost and other criteria

Ignore Cost:
: Ignore cost, choose transport solely based on other criteria

The default is "Balance Cost".



# Sample API definition in Go {#appendix-api-sketch}

This document defines an abstract interface. To illustrate how this would map
concretely into a programming language, an API interface definition in Go is
available online at https://github.com/mami-project/postsocket.  Documentation
for this API -- an illustration of the documentation an application developer
would see for an instance of this interface -- is available online at
https://godoc.org/github.com/mami-project/postsocket. This API definition will
be kept largely in sync with the development of this abstract interface
definition.


# Relationship to the Minimal Set of Transport Services for End Systems

{{I-D.ietf-taps-minset}} identifies a minimal set of transport services that end systems should offer. These services make all transport features offered by TCP, MPTCP, UDP, UDP-Lite, SCTP and LEDBAT available that 1) require interaction with the application, and 2) do not get in the way of a possible implementation over TCP or, with limitations, UDP. The following text explains how this minimal set is reflected in the present API. For brevity, this uses the list in Section 4.1 of {{I-D.ietf-taps-minset}}, updated according to the discussion in Section 5 of {{I-D.ietf-taps-minset}}.

\[EDITOR'S NOTE: This is early text. In the future, this section will contain backward references, which we currently avoid because things are still being moved around and names / categories etc. are changing.]

* Connect:  
"Initiate" Action.

* Listen:  
"Listen" Action.

* Specify number of attempts and/or timeout for the first establishment message:  
"Timeout for aborting Connection Establishment" Property, using a time value.

* Disable MPTCP:  
"Parallel Use of Multiple Paths" Property.

* Hand over a message to reliably transfer (possibly multiple times) before connection establishment:  
"InitiateWithSend" Action.

* Hand over a message to reliably transfer during connection establishment:  
"InitiateWithSend" Action.

* Change timeout for aborting connection (using retransmit limit or time value):  
"Timeout for aborting Connection" property, using a time value.

* Timeout event when data could not be delivered for too long:  
"ConnectionError" Event.

* Suggest timeout to the peer:  
TCP-specific Property: User Timeout.

* Notification of Excessive Retransmissions (early warning below abortion threshold):  
"Notification of excessive retransmissions" property.

* Notification of ICMP error message arrival:  
"Notification of ICMP soft error message arrival" property.

* Choose a scheduler to operate between streams of an association:  
"Connection group transmission scheduler" property.

* Configure priority or weight for a scheduler:  
"Priority (Connection)" property.

* "Specify checksum coverage used by the sender" and "Disable checksum when sending":  
"Corruption Protection Length" property (value 0 to disable).

* "Specify minimum checksum coverage required by receiver" and "Disable checksum requirement when receiving":  
"Required minimum coverage of the checksum for receiving" property (value 0 to disable).

* "Specify DF" field and "Request not to bundle messages:"  
The "Singular Transmission" Message property combines both of these requests, i.e. if a request not to bundle messages is made, this also turns off DF in case of protocols that allow this (only UDP and UDP-Lite, which cannot bundle messages anyway).

* Get max. transport-message size that may be sent using a non-fragmented IP packet from the configured interface:  
"Maximum Message size before fragmentation or segmentation" property.

* Get max. transport-message size that may be received from the configured interface:  
"Maximum Message size on receive" property.

* Obtain ECN field:  
"ECN" is a defined metadata value as part of the Message Receive Context.

* "Specify DSCP field", "Disable Nagle algorithm", "Enable and configure a 'Low Extra Delay Background Transfer'":  
As suggested in Section 5.5 of {{I-D.ietf-taps-minset}}, these transport features are collectively offered via the "Capacity profile" property.

* Close after reliably delivering all remaining data, causing an event informing the application on the other side:  
This is offered by the "Close" Action with slightly changed semantics in line with the discussion in Section 5.2 of {{I-D.ietf-taps-minset}}.

* "Abort without delivering remaining data, causing an event informing the application on the other side" and "Abort without delivering remaining data, not causing an event informing the application on the other side":  
This is offered by the "Abort" action without promising that this is signaled to the other side. If it is, a "ConnectionError" Event will fire at the peer.

* "Reliably transfer data, with congestion control", "Reliably transfer a message, with congestion control" and "Unreliably transfer a message":  
Reliability is controlled via the "Reliable Data Transfer (Message)" Message property. Transmitting data without delimiters is done by not using a Framer. The choice of congestion control is provided via the "Congestion control" property.

* Configurable Message Reliability:  
The "Lifetime" Message Property implements a time-based way to configure message reliability. 

* "Ordered message delivery (potentially slower than unordered)" and "Unordered message delivery (potentially faster than ordered)":  
The two transport features are controlled via the Message property "Ordered".

* Request not to delay the acknowledgement (SACK) of a message:  
Should the protocol support it, this is one of the transport features the transport system can use when an application uses the Capacity Profile Property with value "Low Latency/Interactive".

* Receive data (with no message delimiting):  
"Received" Event without using a Deframer.

* Receive a message:  
"Received" Event. Section 5.1 of {{I-D.ietf-taps-minset}} discusses how messages can be obtained from a bytestream in case of implementation over TCP. Here, this is dealt with by Framers and Deframers.

* Information about partial message arrival:  
"ReceivedPartial" Event.

* Notification of send failures:  
"Expired" and "SendError" Events.

* Notification that the stack has no more user data to send:  
Applications can obtain this information via the "Sent" Event.

* Notification to a receiver that a partial message delivery has been aborted:  
"ReceiveError" Event.
