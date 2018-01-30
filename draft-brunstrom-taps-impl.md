---
title: Implementing Interfaces to Transport Services
abbrev: TAPS Implementation
docname: draft-brunstrom-taps-impl-latest
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
    ins: A. Brunstrom
    name: Anna Brunstrom
    role: editor
    email: anna.brunstrom@kau.se
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
    ins: K-J. Grinnemo
    name: Karl-Johan Grinnemo
    org: Karlstad University
    email: karl-johan.grinnemo@kau.se
  -
    ins: T. Jones
    name: Tom Jones
    org: University of Aberdeen
    email: tom@erg.abdn.ac.uk
  -
    ins: C. Perkins
    name: Colin Perkins
    org: University of Glasgow
    street: School of Computing Science
    city: Glasgow  G12 8QQ
    country: United Kingdom
    email: csp@csperkins.org

--- abstract

this will be the TAPS API Implementation Considerations document

--- middle

# Introduction

note that these are basically just brian's idea here; further structure to be
determined on monday afternoon.

# Implementing Pre-Establishment

# General Considerations for Racing During Establishment

don't race when you can't race.

# Specific Transport Protocol Considerations

## TCP

## UDP

## SCTP

## QUIC

## HTTP over TLS as a pseudotransport

# Rendezvous and Environment Discovery

## ICE and STUN

# IANA Considerations

RFC-EDITOR: Please remove this section before publication.

This document has no actions for IANA.

# Security Considerations

explicitly point out that implementations need to be careful about downgrade.
