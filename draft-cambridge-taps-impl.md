---
title: Implementing Interfaces to Transport Services
abbrev: TAPS Implementation
docname: draft-cambridge-taps-impl-latest
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
    role: editor
    name: Brian Trammell
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland


--- abstract

this will be the TAPS API Implementation Considerations document

--- middle

# Introduction

note that these are basically just brian's idea here; further structure to be
determined on monday afternoon.

# Implementing Pre-Establishment

# General Considerations for Racing During Establishment

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
