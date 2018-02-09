---
title: An Architecture for Transport Services
abbrev: TAPS Architecture
docname: draft-pauly-taps-arch-latest
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
    ins: T. Pauly
    name: Tommy Pauly
    role: editor
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: tpauly@apple.com
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
    ins: A. Brunstrom
    name: Anna Brunstrom
    org: Karlstad University
    email: anna.brunstrom@kau.se
  -
    ins: G. Fairhurst
    name: Gorry Fairhurst
    org: University of Aberdeen
    email: gorry@erg.abdn.ac.uk
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
    ins: C. Wood
    name: Chris Wood
    org: Apple Inc.
    street: One Apple Park Way
    city: Cupertino, California 95014
    country: United States of America
    email: cawood@apple.com

normative:
    RFC8095:
    I-D.ietf-taps-minset:
    I-D.pauly-taps-transport-security:
    I-D.brunstrom-taps-impl:
      title: Implementing Interfaces to Transport Services
      url: https://taps-api.github.io/drafts/draft-brunstrom-taps-impl.html
      authors:
        -
          ins: Anna Brunstrom
	-
          ins: Tommy Pauly
    I-D.trammell-taps-interface:
      title: An Abstract Application Layer Interface to Transport Services
      url: https://taps-api.github.io/drafts/draft-trammell-taps-interface.html
      authors:
        -
          ins: Brian Trammell
	-
          ins: Michael Welzl

--- abstract

This document provides an overview of the architecture of Transport Services, a system for exposing the features of transport protocols to applications. This architecture serves as a basis for Application Programming Interfaces (APIs) and implementations that provide flexibile transport networking services. It defines the common set of terminology and concepts to be used in more detailed discussion of transport services.

--- middle

# Introduction

This document provides an overview of the architecture of Transport Services, a system for exposing the features of transport protocols to applications. This architecture serves as a basis for Application Programming Interfaces (APIs) and implementations that provide flexibile transport networking services. It defines the common set of terminology and concepts to be used in more detailed discussion of transport services.

Many APIs to perform transport networking have been deployed, perhaps the most widely known and imitated being the BSD socket() interface. The names for calls into these interfaces and events returned from these interfaces vary depending on the intended application, even when the functions are analogous. Similarly, terminology for the implementation of protocols offering transport services vary based on the context of the protocols themselves. This variety can lead to confusion when trying to define the fundamental commonalities between APIs and distill the nuanced differences.

The goal of the Transport Services architecture is to provide a common, flexible, and reusable interface for transport protocols. As applications adopt this interface, they will benefit from a wide set of transport features that can evolve over time, and ensure that the system providing the interface can optimize is behavior based on the application requirements and network conditions.

This document is developed in parallel with the specification of the Transport Services API {{I-D.trammell-taps-interface}} and Implementation {{I-D.brunstrom-taps-impl}} documents.

# Background

The architecture of Transport Services is based on the survey of Services Provided by IETF Tranport Protocols and Congestion Control Mechanisms {{RFC8095}}, and the distilled minimal set of the features offered by trasport protocols {{I-D.ietf-taps-minset}}. This work identified the common features and patterns across all transport protocols developed thus far.

Since transport security is an increasingly relevant aspect of using transport protocols on the Internet, this architecture also considers the impact of transport security protocols on the feature set exposed by transport services {{I-D.pauly-taps-transport-security}}.

One of the key insights from the defining the minimal set of transport features {{I-D.ietf-taps-minset}} was to identify how features either required application interaction and guidance (referred to as Functional) or else could be handled automatically by a system implementing Transport Services (referred to as Automatable). Among the Functional features, some were common across all or nearly all transport protocols, while others could be seen as features that, if specified, would only be useful to \[MICHAEL: with?] a single transport protocol, but not harm the functionality of other protocols.
\[MICHAEL: I think adding a simple example may be good here. I suggest: For example, some protocols can deliver messages faster for applications that do not require them to arrive in the order in which they were sent. However, this functionality can never be automatically provided without the application being involved.]

# Design Principles

The goal of the Transport Services architecture is to redefine the interface between applications and transports in a way that allows the transport layer to evolve and improve without fundamentally changing the contract with the application. This requires a careful consideration of how to expose the capabilities of protocols.

There are several degrees in which a Transport Services system can offer flexibility to an application: it can provide access to multiple sets of protocols and protocol features, it can use these protocols across multiple networks that may have different performance and functional characteristics, and it can communicate with different addresses for a peer to optimize performance. Beyond these, if the API for the system remains the same over time, new protocols and features may be added to the system's implementation without requiring significant changes in applications for adoption.

The following considerations were used in the design of this architecture.

## Common APIs for Common Features

Functionality that all identified transport protocols share in common should be accessible through a unified set of API calls. An application should be able to implement logic for its basic use of transport networking (establishing the transport, and sending and receiving content) once, and expect that implementation to continue to function as the transports change.

Any Transport Services API must expose the basic set of transport networking features. This implies, at a minimum, the ability to use the equivalent of both TCP and UDP socket interfaces.

## Access to Specialized Features

Since applications will often need to control fine-grained details of transport protocols in order to optimize their behavior and ensure compatibility with remote peers, a Transport Services system also needs to allow more specialized protocol features to be used. The interface for these specialized options should be treated differently from the common options in order to ensure flexibility.

A specialized feature may be required by an application only when using a specific protocol, and not when using others. For example, if an application is using UDP, it may require control over the checksum or fragmentation behavior for UDP; if it used a protocol to frame its content over a bytestream like TCP, it would not need these options. In such cases, the API should expose the features in such a way that they take effect when a particular protocol is selected, but do not imply that only that protocol may be used if there are equivalent options.

Other specialized features, however, may be strictly required by an application and thus constrain the set of protocols that can be used. For example, if an application requires encryption of its transport content, it must be guaranteed that only combinations of protocols that include a transport security protocol.\[MICHAEL: "...are used?" - but also the "combinations that include" seems strange here. Suggest: "it must be guaranteed that a chosen transport protocol that can encrypt data."] A Transport Services API must allow applications to define such requirements and constrain the system's options. Since such options are not part of the core, common features, however, it should be simple for an application to modify its set of constraints and change the set of allowable protocol features without changing the core implementation.

## Scope for API and Implementation Definitions

The Transport Services API is envisioned as the abstract model for a family of APIs that all share a common way to expose transport features and encourage flexibility. The API definition {{I-D.trammell-taps-interface}} is aimed at explaining to application developers how to use this interface.

Implementations that provide the Transport Services API {{I-D.brunstrom-taps-impl}}, on the other hand, will vary due to system-specific support and the needs of the deployment scenario. It is expected that all implementations of Transport Services will offer the entire mandatory API, but that some features will not be functional in certain implementations. For example, all implementations should offer sufficient APIs to use basic transport protocols like TCP and UDP, but it is possible that a very constrained device may not have a full TCP implementation.

In order to preserve flexibility and compatibility with future protocols, top-level features in the Transport Services API should avoid referencing particular transport protocols. Mappings of these API features in the Implementation document, on the other hand, must explain the ramifications of each feature on existing protocols. It is expected that the Implementation document will be updated and supplemented as new protocols and protocol features are developed.

It is important to note that neither the Transport Services API nor the Implementation document defines new protocols that require any changes on peers. The Transport Services system must be deployable on one side only, as a way to allow an application to make better use of available capabilities on a system, and protocols features that may be supported by peers across the network.

# Transport Services Architecture and Concepts

The concepts defined in this document are intended primarily for use in Internet Drafts and specifications. While the specific terminology may be used in specific implementations, it is expected that there will remain a variety of terms used by running code.

The architecture divides the concepts for Transport Services into two categories:
- API concepts, which are meant to be exposed to applications
- System-implementation concepts, which are meant to be internally used when building systems that implement Transport Services.

The following diagram summarizes the top-level concepts in the architecture and how they relate to one another.

~~~~~~~~~~

  +------------------------------------------------------+
  |                    Application                       |
  +-+------------------^----------+----------^-----------+
    |                  |          |          |
  pre-               data         |        events
  establishment    transfer       |          |
    |                  |      terminate      |
    |                  |          |          |
    |             +----v----------v--+       |
  +-v-------------+      Objects     +-------+----------+
  |  Transport    +--------+---------+                  |
  |  Services              |                            |
  |  API                   |                            |
  +------------------------|----------------------------+
                           |
  +------------------------|----------------------------+
  |  Transport             |                            |
  |  System                |        +-----------------+ |
  |  Implementation        |        |   Association   | |
  |                        |        |      Cache      | |
  |  (Candidate Gathering) |        +-----------------+ |
  |                        |                            |
  |  (Candidate Racing)    |        +-----------------+ |
  |                        |        |     System      | |
  |                        |        |     Policy      | |
  |             +----------v-----+  +-----------------+ |
  |             |    Protocol    |                      |
  +-------------+     Stack      +----------------------+
                |    Instance    |
                +-------+--------+
                        V
              Network Layer Interface
~~~~~~~~~~
{: #fig-abstractions title="Concepts and Relationships in the Transport Services Architecture \[MICHAEL: I think it's bad that establishment is missing from the figure - but I understand it could get messy. Maybe writing '(pre-)establishment' could help, to indicate that there is both pre-establishment and establishment ?]"}

## Transport Services API Concepts

Fundamentally, a Transport Services API needs to provide basic objects that allow applications to establish communication and send and receive data {{objects}}. These may be exposed as handles or referenced objects, depending on the language.

Beyond the basic objects, there are several high-level groups of actions that any Transport Services API must provide:

Pre-Establishment {{preestablishment}}

>> Pre-Establishment encompasses the parameters that an application can pass to describe its intent, requirements, prohibitions, and preferences for its networking operations. For any system that provides generic Transport Services, these properties should primarily offer knobs that apply across multiple transports. Properties may have a large impact on the rest of the the aspects of the interface: they can modify how establishment occurs, they can influence the expectations around data transfer, and they determine the set of events that will be supported.

Establishment {{establishment}}

>> Establishment focuses on the actions that an application takes \[MICHAEL: on? with?] the transport objects to prepare for data transfer.

Data Transfer {{datatransfer}}

>> Data Transfer consists of how an application represents data to be sent and received, the functions required to send and receive that data, and how the application is notified of the status of its data transfer.

Event Handling {{events}}

>> Events define the set of properties that an application may be notified of during the lifetime of transport objects. These may also provide opportunities for the application to interact with the underlying transport.

Termination {{termination}}

>> Termination focuses on the methods by which data transmission is ceased, and state is torn down in the transport.

### Objects {#objects}

Connection

>> A Connection is the fundamental object used by an application for all interaction with a peer and data transfer. It is generally capable of bi-directional communication. It has state that represents the capability of being able to send or receive content between its local and remote endpoints. This capability may be based merely on the existence of a route between the two endpoints and an application's permission to send and receive on its local endpoint; or may represent successful protocol handshakes at one or more layers that are pre-requistes to receiving content.

Listener

>> A Listener is any object that can be used to prepare a Connection with a remote endpoint without initiation by the local application. That is, it can passively accept initiations for Connections made by another endpoint.

### Pre-Establishment {#preestablishment}

Endpoint

>> An Endpoint represents one side of a transport connection. It is a concept that represents how an application views itself or a peer that may vary in levels of specificity. Examples include "IP address + port", "hostname + port", and "DNS service name".

Remote Endpoint

>> The Remote Endpoint in a properties \[MICHAEL: I can't parse "in a properties" here. Should this be removed?] represents the application's view of a peer that can participate in a transport connection.

Local Endpoint

>> The Local Endpoint in a properties \[MICHAEL: I can't parse "in a properties" here. Should this be removed?] represents the application's view of itself that it wants to use for transport connections. This may be optional, or may be generic (a wildcard address, for example).

Path Selection Properties

>> Path Selection Properties consist of the options that an application may set to influence the selection of path between itself and the Remote Endpoint. These options can come in the form of requirements, prohibitions, or preferences. Examples of options which may influence path selection include the interface type (such as a Wi-Fi Ethernet connection, or a Cellular LTE connection), characteristics of the path that are locally known like Maximum Transmission Unit (MTU), or expected throughput or latency.

Protocol Selection Properties

>> Protocol Selection Properties consist of the options that an application may set to influence the selection of transport protocol, configure the behavior of generic transport protocol features. These options come in the form of requirements, prohibitions, and preferences. Examples include reliability, service class, multipath support, and fast open support. 

Specific Protocol Properties

>> Specific Protocol Properties refers to the subset of Protocol Properties options that apply to a single protocol (transport protocol, IP, or security protocol). The presence of such Properties does not necessarily require that a specific protocol must be used, but that if this protocol is employed, a particular set of options should be used. This is critical to allow compatibility with Protocol Properties on peers.

### Establishment {#establishment}

Initiate

>> Initiate is the primary action that an application can take to create a Connection to a remote endpoint, and prepare any required local or remote state to be able to send and/or receive content. For some protocols, this may initiate a server-to-client style handshake; for other protocols, this may just establish local state; and for peer-to-peer protocols, this may begin the process of a simulatenous open.

Listen

>> Listen is the action of marking a Listener as willing to accept incoming Connections.

### Data Transfer {#datatransfer}

Content

>> Content is a unit of data that can be respresented as bytes that can be transferred between two endpoints over a transport connection. The bytes within a unit of Content are assumed to be ordered within the unit itself. That is, if an application does not care about the order in which a peer receives two distinct spans of bytes, those spans of bytes are considered independent units of Content. Content may or may not be usable if incomplete or corrupted. Boundaries of Content may or may not be understood or transmitted by transport protocols. That is, what one application considers to be two units of Content sent on a stream-based transport may be treated as a single unit of Content by the application on the other side.

Send

>> Send is the action to transmit Content over a Connection to a remote endpoint. The interface to Send may include options specific to how this Content is to be sent. Status of the Send operation may be delivered back to the application in an event {{events}}.

Receive

>> Receive is the indication that the application is ready to asynchronously accept Content over a Connection from a Remote Endpoint, while the Content itself will be delivered in an event {{events}}. The interface to Receive may include options specific to the Content that is to be delivered to the application.

### Event Handling {#events}

This list of events that can be delivered to an application is not exhaustive, but gives the top-level categories of events. The API may expand this list.

Connection Ready

>> A Connection Ready event signals to an application that a given Connection is ready to send and/or receive Content. If the Connection relies on handshakes to establish state between peers, then it is assumed that these steps have been taken.

Connection Finished

>> A Connection Finished event signals to an application that a given Connection is no longer usable for sending or receiving Content. This should deliver a useful error to the application.

Connection Received

>> A Connection Received event signals to an application that a given Listener has passively received a Connection.

Content Received

>> A Content Received event delivers received content to the application, based on a Receive action. This may include an error if the action failed.

Content Sent

>> A Content Sent event notifies the application of the status of its Send action. This may be an error, an indication that Content has been processed by the protocol stack, or potentially \[MICHAEL: Why "potentially" - because you're not sure about it? If it may be such an indication, "potentially" should be removed (this seems like the right thing to do for me).] an indication that Content has been acknowledged by a peer.

Path Properties Changed

>> A Path Properties Changed event notifies the application that some property of the Connection has changed that may influence how and where data is sent and/or received.

### Termination {#termination}

Close

>> Close is the action an application may take on a Connection to indicate that it no longer intends to send data, is no longer willing to receive data, and that the protocol should signal this state to the remote endpoint if applicable.

Abort

>> Abort is an action the application may take on a Connection to indicate a Close, but with the additional indication that the transport system should not attempt to deliver any outstanding data.

## Transport System Implementation Concepts

The Transport System Implementation Concepts define the set of objects used internally to a system or library to provide the functionality of transport networking, as required by the abstract interface.

Connection Group

>> For multiplexing transport protocols, a Connection Group is a set of Connections that can be multiplexed together.

Path

>> A Path represents an available set of properties of a network route on which packets may be sent or received.

Protocol Instance

>> A Protocol Instance is a single instance of one protocol, including any state it has necessary to establish connectivity or send and receive content.

Protocol Stack

>> A Protocol Stack is a set of Protocol Instances (including relevant application, security, transport, or Internet protocols) that are used together to establish connectivity or send and receive content. A single stack may be simple (a single transport protocol instance over IP), or complex (multiple application protocol streams going through a single security and transport protocol, over IP; or, a multi-path transport protocol over multiple transport subflows).

System Policy

>> A Transport Service System Policy defines the algorithm it uses to take connection properties from the application, and determine how it will gather candidate paths and protocols {{gathering}} and race the candidates during establishment {{racing}}.

Association Cache

>> The Association Cache holds the state that the implementation keeps for each set of associated endpoints that have been used previously. This can include DNS results, TLS session state, previous success and quality of transport protocols over certain paths.

### Gathering {#gathering}

Path Candidate Identification/Gathering

>> Path Selection represents the act of choosing one or more paths that are available to use based on the Path Selection Properties provided by the application, and a Transport Services system's policies and heuristics.
\[MICHAEL: I find it awkward that the terms and headline here talk about "Gathering" but then this description doesn't. Maybe because "Path Selection" and "Protocol Selection" are indeed clearer, easier terms than this Identification/Gathering stuff?]

Protocol Candidate Identification/Gathering

>> Protocol Selection represents the act of choosing one or more sets of protocol options that are available to use based on the Protocol Properties provided by the application, and a Transport Services system's policies and heuristics.
\[MICHAEL: Same as above, here...]

### Racing {#racing}

Protocol Option Racing

\[MICHAEL: About all the definitions below: I believe you can't "attempt a stack", you can only attempt to *use* a stack. Not easy to fix in a nice way because of the "scheduling attempts for", but I think as it stands, this is grammatically broken. Then again, I'm no native speaker!]

>> Protocol Racing is the act of attempting, or scheduling attempts for, multiple Protocol Stacks that differ based on the composition of protocols or the options used for protocols.

Path Racing

>> Path Racing is the act of attempting, or scheduling attempts for, multiple Protocol Stacks that differ based on a selection from the available Paths.

Endpoint Racing

>> Endpoint Racing is the act of attempting, or scheduling attempts for, multiple Protocol Stacks that differ based on the specific representation of the remote and local endpoints, such as IP addresses resolved from a DNS hostname.

# IANA Considerations

RFC-EDITOR: Please remove this section before publication.

This document has no actions for IANA.

# Security Considerations

\[TASK: discuss security considerations]
