---
title: An Architecture for Interfaces to Transport Services
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
    street: 1 Infinite Loop
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

--- abstract

this will be the TAPS Interface Architecture document.

--- middle

# Introduction

why does this document exist...

# Terminology

glossary with forward references

# Design Principles

what are the initial design principles

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

introduce, list phases

1. Pre-Establishment

2. Establishment

3. Data Transfer

4. Event Handling

5. Termination

## Objects

draw a picture

## Pre-Establishment

## Establishment

## Data Transfer

## Event Handling

## Termination

## Internal Terminology

...things you probably need to implement this, but is not

# IANA Considerations

RFC-EDITOR: Please remove this section before publication.

This document has no actions for IANA.

# Security Considerations

be paranoid
