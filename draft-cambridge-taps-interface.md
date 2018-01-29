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
    role: editor
    name: Brian Trammell
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland


--- abstract

this will be the TAPS Abstract API definition document.

--- middle

# Introduction

why does this document exist...

# Design Principles

take some first principles (beyond those in the architecture) from post sockets. asynchronicity and message orientation are key here.

# Pre-Establishment

## Resolving Remote Endpoints

note here: resolution should be flexible, and should accept URLS and URL-like
things. It should be possible to place

# Establishing Connections

# Sending Data

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
