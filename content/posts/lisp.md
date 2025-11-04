---
title: LISP Fundamentals
date: 2025-11-04
draft: false
---

Advantages and benefits LISP provides highlighted in this post are **simplified traffic engineering**, **multihoming**, **overlay tunnel**, and **AF over AF**. 

**Locator/ID Separation Protocol (LISP)** is an IP in UDP tunnel protocol that separates the identity of a device from its location on the network. It allows sites to create tunnels between them and identifies devices using **Endpoint Identifiers (EIDs)**. Traffic then enters and exits the network through **Routing Locators (RLOCs)**.

For more specific information please check out the sources this post was based off of:
[Cisco Configuring LISP]({{% relref "https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_lisp/configuration/xe-3s/irl-xe-3s-book/irl-cfg-lisp.html" %}})
[RFC 6830]({{% relref "https://datatracker.ietf.org/doc/html/rfc6830" %}})

## Private LISP Minimal Configuration
**Relevant Terminology**
- Endpoint Identifiers (EIDs): Server/VM/Host IP address.
- Routing Locators (RLOCs): Router's link IP address.
- Mapping Database (M-DB): Association between EID and RLOC.
- MS/MR: The combined roles seen below.
	- Map Server (MS): Holds EID to RLOC mappings.
	- Map Resolver (MR): Responds to map-request messages
- xTR: The combined roles seen below.
	- Ingress Tunnel Router (iTR): Asks Map Resolve for EID to RLOC mapping and is responsible for LISP/UDP packet encapsulation.
	- Egress Tunnel Router (eTR): Registers EID/RLOC mapping to Map Server and is responsible for LISP/UDP decapsulation and forwarding the packet to the destination.
##### Setting the MS/MR and expected EID registration
```IOSv
MS-MR(config)# router lisp
MS-MR(config-router-lisp)# ipv4 map-server
MS-MR(config-router-lisp)# ipv4 map-resolver
MS-MR(config-router-lisp)# site A
MS-MR(config-router-lisp-site)# eid-prefix x.x.x.x/24 accept-more-specifics
MS-MR(config-router-lisp-site)# authentication-key APASS
MS-MR(config-router-lisp)# site B
MS-MR(config-router-lisp-site)# eid-prefix x.x.x.x/24 accept-more-specifics
MS-MR(config-router-lisp-site)# authentication-key BPASS
```
- - The use of `eid-prefix` lets our MS/MR know who to expect registration from `accept-more-specifics` allows our MS/MR to accept registration for more specific prefixes such as /24-/32 within the specific range.
- Passwords must be unique from site to site.
##### Setting the xTR For A site

## Ingress Load Balancing


## LISP to Non-LISP
**Relevant Terminology**
- PxTR: The combined roles seen below.
	- Proxy Ingress Tunnel Router (PiTR): iTR that connects LISP to Non LISP
	- Proxy Egress Tunnel Router (PeTR): eTR that connects LISP to Non LISP


## AF over AF
