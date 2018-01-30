---
title: Transport Services Architecture
abbrev: TAPS Architecture
docname: draft-cambridge-taps-arch-latest
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

--- abstract

This document provides an overview of the the architecture of Transport Services. This architecture serves as a basis for Application Programming Interfaces (APIs) and implementations that provide flexibile transport networking services. It defines the common set of terminology and concepts to be used in more detailed discussion of transport services.

--- middle

# Introduction

This document provides an overview of the the architecture of Transport Services. This architecture serves as a basis for Application Programming Interfaces (APIs) and implementations that provide flexibile transport networking services. It defines the common set of terminology and concepts to be used in more detailed discussion of transport services.

Many APIs to perform transport networking have been deployed, perhaps the most widely known and imitated being the BSD socket() interface. The names for calls into these interfaces and events returned from these interfaces vary depending on the intended application, even when the functions are analogous. Similarly, terminology for the implementation of transport services vary based on the context of the protocols themselves. This variety can lead to confusion when trying to define the fundamental commonalities between APIs and distill the nuanced differences.

The effort of Transport Services is focused on providing a common, flexible, and reusable interface and implementation for transport protocols. This generic and flexible approach to transport interfaces requires a standardized set of concepts and terminology with which to describe functionality.

The terminology defined in this document is intended primarily for use in Internet Drafts and specifications. While the concepts and terms may be used in specific implementations, it is expected that there will remain a variety of terms used by running code.

# Terminology

glossary with forward references

# Design Principles

This document is developed in parallel with the specification of the Transport Services API and Transport System documents. The following are the guiding principles of what is and is not included in each.

1. Every feature described in the Transport Services API document MUST have a functional mapping to at least one protocol, which MUST be described in the Transport System document.

2. The capabilities that are common between all transports and secure transports (the minimum set) MUST be expressible in the Transport Services API. This guarantees support for the basic capabilities of TCP and UDP.

3. The Transport Services API MAY expose features or policy requirements that only apply to some transport protocols, and thus may limit the permissible transport protocols and transport protocol options on a per-connection basis.

4. Since Transport Services does not define its own transport protocols, but rather provides an interface to existing and future protools, an application using the Transport Services API MUST be able to communicate with peers or servers that use alternate implementations. 

5. In order to preserve flexibility and compatibility with future protocols, features in the Transport Services API SHOULD NOT reference particular transport protocols. Mappings of API features in the Transport System document, on the other, MUST explain the ramifications of each feature on each existing protocol (which will need to be supplemented and updated in the future).

Provide flexibility in protocol selection without changing the API

"Write once, configure many times, run anywhere"

[Uniform interface allows adoption and deployment of TAPS

[Mandatory to implement entire API, but not every protocol feature]

# Interface Architecture and Terminology

introduce architecture overview diagram

~~~~~~~~~~

  +------------------------------------------------------+
  |                    Application                       |
  +-|----------------+-----^------+----------^-----------+
    |                |     |      |          |
  pre-             send    |      |       events
  establishment      |  receive   v          |
    |                |     ^    terminate    |
    |                |     |      +          |
    |             +--v-----+------v--+       |
  +-v-------------+    Connection    +-------+----------+
  |  Transport    +--------+---------+                  |
  |  Services              |                            |
  |  API                   |                            |
  +------------------------|----------------------------+
                           |
  +------------------------|----------------------------+
  |  Transport             |                            |
  |  System                |        +-----------------+ |
  |                        |        | Path  / TS State| |
  |  (Path Info Gathering) |        | Cache / Cache   | |
  |                        |        +-----------------+ |
  |  (Connection Racing)   |                            |
  |                        |        +-----------------+ |
  |                        |        | System / User   | |
  |                        |        | Policy / Config | |
  |             +----------v-----+  +-----------------+ |
  |             |    Protocol    |                      |
  +-------------+     Stack      +----------------------+
                |    Instance    |
                +-------+--------+
                        V
              Network Layer Interface
~~~~~~~~~~
{: #fig-abstractions title="Concepts and Relationships in the Transport Services Interface Architecture"}

The Transport Services API defines the set of objects, parameters, and functions provided to an application when using a transport networking stack. The consumer of this interface is code in an application which can treat networking transports as a generic tool.

These concepts are defined to be protocol-independent and as forward-looking as possible. While it is a requirement that the concepts are applicable to the least common denominator for transport (today, TCP streams), they must also allow the use of a variety of protocols that provides various other benefits.

There are several high-level phases of functions that any Transport Services API must provide:

1. Pre-Establishment

>> Properties encompasses the parameters that an application can pass to describe its intent, requirements, prohibitions, and preferences for its networking operations. For any system that provides generic Transport Services, this properties should primarily offer knobs that apply across multiple transports. Properties may have a large impact on the rest of the the aspects of the interface: it can modify how establishment occurs, it can influence the expectations around data transfer, it determines the set of events that will be supported.

2. Establishment

>> Establishment focuses on the objects that an application interacts with to prepare to transfer data. These objects themselves are defined with specific terminology, along with the terms for the most important actions that can be taken on the objects.

3. Data Transfer

>> Data Transfer consists of how an application represents data to be sent and received, the functions required to send and receive that data, and how the application is notified of the status of its data transfer.

4. Event Handling

>> Events define the set of properties that an application may be notified of during the lifetime of transport objects. These may also provide opportunities for the application to interact with the underlying transport.

5. Termination

>> Termination focuses on the methods by which data transmission is ceased, and state is torn down in the transport.

## Objects

Connection

>> A Connection is the fundamental object used by an application for all interaction with a peer and data transfer. It is generally capable of bi-directional communication. It has state that represents the capability of being able to send or receive content between its local and remote endpoints. This capability may be based merely on the existence of a route between the two endpoints and an application's permission to send and receive on its local endpoint; or may represent successful protocol handshakes at one or more layers that are pre-requistes to receiving content. 

Listener

>> A Listener is any object that can be used to prepare a Connection with a remote endpoint without initiation by the local application. That is, it can passively accept initiations for Connections made by another endpoint.

## Pre-Establishment

Endpoint

>> An Endpoint represents one side of a transport connection. It is a concept that represents how an application views itself or a peer that may vary in levels of specificity. Examples include "IP address + port", "hostname + port", and "DNS service name".

Remote Endpoint

>> The Remote Endpoint in a properties represents the application's view of a peer that can participate in a transport connection.

Local Endpoint

>> The Local Endpoint in a properties represents the application's view of itself that it wants to use for transport connections. This may be optional, or may be generic (a wildcard address, for example).

Path Selection Properties

>> Path Selection Properties consists of any option that an application may set to influence the selection of path between itself and the Remote Endpoint. These options can come in the form of requirements, prohibitions, or preferences. Examples of options which may influence path selection include the interface type (such as a Wi-Fi Ethernet connection, or a Cellular LTE connection), characteristics of the path that are locally known like Maximum Transmission Unit (MTU), or expected throughput or latency.

Protocol Properties

>> Protocol Properties consists of any option that an application may set to influence the selection of transport protocol, configure the behavior of transport protocols, or influence the behavior of other protocols such as IP and security protocols.

> Protocol Selection Properties

>>> Generic Protocol Properties refers to the subset of Protocol Properties options that apply across multiple or all transport protocols. These options come in the form of requirements, prohibitions, and preferences. Examples include reliability, service class, multipath support, and fast open support.

> Specific Protocol Properties

>>> Specific Protocol Properties refers to the subset of Protocol Properties options that apply to a single protocol (transport protocol, IP, or security protocol). The presence of such a properties does not necessarily require that a specific protocol must be used, but that if this protocol is employed, a particular set of options should be used. This is critical to allow compatibility with protocol propertiess on peers. 

## Establishment

The terminology of Establishment is divided into two sections: Objects that are the focus of an application's interaction, and Actions that the application can take on these Objects to establish connectivity.

Initiate

>> Initiate is the primary action that an application can take to create a Connection to a remote endpoint, and prepare any required local or remote state to be able to send and/or receive content. [NOTE: Make clear that this works for simulataneous open for peer to peer]

Listen

>> Listen is the action of marking a Listener as willing to accept incoming Connections.

## Data Transfer

Content

>> Content is a unit of data that can be respresented as bytes that can be transferred between two endpoints over a transport connection. The bytes within a unit of Content are assumed to be ordered within the unit itself. That is, if an application does not care about the order in which a peer receives two distinct spans of bytes, those spans of bytes are considered independent units of Content. Content may or may not be usable if incomplete or corrupted. Boundaries of Content may or may not be understood or transmitted by transport protocols. That is, what one application considers to be two units of Content sent on a stream-based transport may be treated as a single unit of Content by the application on the other side.

Send

>> Send is the action to transmit Content over a Connection to a remote endpoint. The interface to sending may include options specific to how this content is to be sent, as well as ways to notify the application of the status of the sending operation.

Receive

>> Receive is the indication that the application is ready to asynchronously accept Content over a Connection from a remote endpoint [reference the event that is delivered upon reception]. The interface to receiving may include options specific to the content that is to be delivered to the application, as well as ways to notify the application of errors.

## Event Handling

Connection Ready

>> A Ready event signals to an application that a given Connection is ready to send and/or receive Content. If the Connection relies on handshakes to establish state between peers, then it is assumed that these steps have been taken.

Connection Finished

>> A Failed event signals to an application that a given Connection is unable to be used for sending or receiving Content. This should deliver a useful error to the application.

Connection Received

>> An Accept event signals to an application that a given Listener has passively received a Connection.

Content Received

Content Sent

>> Sent from stack, or acked, or whatever. May be send error.

Path Properties Changed

[API may expand this list]

## Termination

Close

>> Close is the action an application may take on a Connection to indicate that it no longer intends to send data.

Abort

>> Abort is the action an application may take on a Connection to indicate that it no longer intends to send data, will not be willing to receive data, and that the protocol should signal this state to the remote endpoint if applicable.

## Transport System Terminology

The Transport System defines the set of objects used internally to a system or library to provide the functionality of transport networking, as required by the abstract interface.

Connection Group

>> Multiplexing! A group of connections.

Path

>> A Path represents an available set of properties of a network route on which packets may be sent or received.

Protocol Instance

>> A Protocol Instance is a single instance of one protocol, including any state it has necessary to establish connectivity or send and receive content.

Protocol Stack

>> A Protocol Stack is a set of Protocol Instances (including relevant application, security, transport, or Internet protocols) that are used together to establish connectivity or send and receive content. A single stack may simple (a single transport protocol instance over IP), or complex (multiple application protocol streams going through a single security and transport protocol, over IP; or, a multi-path transport protocol over multiple transport subflows).

Policy

>> Decides gathering and racing

Persistent State

>> History and cache (DNS, TLS, routes)

### Gathering

Path Candidate Identification/Gathering

>> Path Selection represents the act of choosing one or more paths that are available to use based on the Path Selection Properties provided by the application, and a Transport Services system's policies and heuristics.

Protocol Candidate Identification/Gathering

>> Protocol Selection represents the act of choosing one or more sets of protocol options that are available to use based on the Protocol Properties provided by the application, and a Transport Services system's policies and heuristics.

### Racing

Protocol Option Racing

>> Protocol Racing is the act of attempting, or scheduling attempts for, multiple Protocol Stacks that differ based on the composition of protocols or the options used for protocols.

Path Racing

>> Path Racing is the act of attempting, or scheduling attempts for, multiple Protocol Stacks that differ based on a selection from the available Paths.

Endpoint Racing

>> Endpoint Racing is the act of attempting, or scheduling attempts for, multiple Protocol Stacks that differ based on the specific representation of the remote and local endpoints, such as IP addresses resolved from a DNS hostname.

# IANA Considerations

RFC-EDITOR: Please remove this section before publication.

This document has no actions for IANA.

# Security Considerations

be paranoid
