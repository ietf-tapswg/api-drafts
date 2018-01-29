---
title: An Abstract Application Layer Interface to Transport Services
abbrev: TAPS Interface
docname: draft-cambridge-taps-interface-latest
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

--- abstract

this will be the TAPS Abstract API definition document.

--- middle

# Introduction

why does this document exist...

# Design Principles

take some first principles (beyond those in the architecture) from post sockets. asynchronicity and message orientation are key here.

# API Summary

# Pre-Establishment

## Resolving Remote Endpoints

note here: resolution should be flexible, and should accept URLS and URL-like
things. binding to transport or pseudotransport happens via pre-establishment
properties.

## Transport Parameters

list parameters to bind to a connection before establishing it

## Cryptographic Parameters

# Establishing Connections

## Connect Parameters

# Sending Data

## Send Parameters

## Sender-side Framing over Stream Protocols

# Receiving Data

## Receiver-side Deframing over Stream Protocols

# Responding to Events

# Error Handling

# IANA Considerations

RFC-EDITOR: Please remove this section before publication.

This document has no actions for IANA.

# Security Considerations

be paranoid
