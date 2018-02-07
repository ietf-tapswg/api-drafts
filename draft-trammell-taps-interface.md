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
calls, for instance, and Events could be implemented though via callback
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

Name resolution is no explicit step of an transport service API.
Therefore, name resolution may be deferred until connection establishment
to incorporate transport parameters.
Instead, a remoteSpecifier object representing the remote endpoint is created
providing an appropriate endpoint representation, which include ip-addresses,
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

Implementations may also support add additional endpoint representations and
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

\[TASK: write me. list parameters to bind to a connection before establishing
it. look at minset for this. note that the API should have sensible and
well-defined defaults.]

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
transport-layer connection to this Connection, causing a new Connection to be
created. The resulting Connection is contained within the ConnectionReceived
event, and is ready to use as soon as it is passed to the application via the
event.

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

Like all Actions in this interface, the Send action is asynchronous. However,
a Send call may block until there is sufficient buffer space in the
implementation and/or the underlying Protocol Stack to handle the Content, in
order to provide sender-side backpressure to the application when transmission
is limited by transport channel capacity.

Connection -> Sent&lt;contentRef>

The Sent event occurs when a previous Send action has completed, i.e. when the
data derived from the Content has been passed down or through the underlying
Protocol Stack and is no longer the responsbility of the implementation of
this interface. The exact disposition of Content when the Sent event occurs is
specific to the implementation and the constraints on the Protocol Stacks
implied by the Connection's transport parameters. The Sent event contains an
implementation-specific reference to the Content to which it applies.

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
this Content may be delivered before the last Content passed to the same
Connection.

### Immediate {#send-immediate}

Immediate is a boolean property. If true, the caller prefers immediacy to
efficiency for this Content, and the Content should not be bundled with other
Content into the same transmission by the underlying Protocol Stack.

### Idempotent {#send-idempotent}

Idempotent is a boolean property. If true, the application-layer entity in the
Content is safe to send to the remote endpoint more than once for a single
Send action. It is used to mark data safe for certain 0-RTT establishment
techniques, where retransmission of the 0-RTT data may cause the remote
application to receive the Content multiple times.

\[NOTE: we need some way to signal to the transport that we want to wait for
0RTT data on Initiate. Probably a transport parameter]

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
recieved that reception has failed. Such conditions that irrevocably lead the
the termination of the Connection are signaled using ConnectionError instead
(see {{termination}}).

## Application-Layer Backpressure at the Receiver {#receive-backpressure}

Implementations of this interface must provide some way for the application to
indicate that it is temporarily not ready to receive new Content. Since the
mechanisms of event handling are implementation-platform specific, this
document does not specify the exact nature of this

## Receiver-side Deframing over Stream Protocols {#receive-framing}

The Receive event is intended to be fired once per application-layer Content
sent by the remote endpoint; i.e., it is a desired property of this interface
that a Send at one end of a Connection maps to exactly one Receive on the
other end. This is possible with Protocol Stacks that provide a mechanism
message boundary preservation, but is not the case over Protocol Stacks that
provide a simple octet stream transport.

For preserving message boundaries over stream transports, this interface
provides receiver-side deframing. This facility is based on the observation
that, since many of our current application protocols evolved over TCP, which
does not provide message boundary preservation, and since many these protocols
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

Connection.Close()

Connection -> Finished&lt;>

Connection.Abort()

Connection -> ConnectionError&lt;>

# Event and Error Handling

\[NOTE: point out that events and errors may be handled differently, although
they are the modeled the same in this specification.]

# IANA Considerations

RFC-EDITOR: Please remove this section before publication.

This document has no actions for IANA.

# Security Considerations

be paranoid
