---
title: An Overview of IPsec
date: 2025-11-12
draft: false
tags:
  - networking
  - VPN
  - IPsec
  - IKE
---
IPsec (Internet Protocol Security) is a suite of protocols that secures IP communications by providing encryption, authentication, and integrity directly at the network layer. IPsec requires both peers to agree on how to secure traffic, it uses a management framework called ISAKMP (Internet Security Association and Key Management Protocol), which defines how peers negotiate and manage Security Associations (SAs) the policies and keys used to protect control plane traffic to continue negotiation to secure data plane traffic. ISAKMP provides the structure, IKEv1 performs the key exchange and negotiation for the control plane, and IPsec enforces the security for the data plane.
### ISAKMP / IKEv1 Phase 1
Phase 1 will be performed in one of two modes being **Main Mode (MM)** or **Aggressive Mode (AM)**.  **MM** is more preferred today as the 3 additional messages of overhead is negligible for modern day processing. This doesn't mean **AM** is obsolete as it still find use in Remote-Access with key-based authentication solutions, where IKE Identities are sent in the clear. 

**MM** is more secure due to that it secures the entire Authentication Exchange including IKE Identities by using 6 messages for all three exchanges described below:
1. Initiator sends the first message to propose its ISAKMP policy. (Cleartext)
2. Responder replies with a selected policy that its capable of and agrees with based on priority. (Cleartext)
3. Initiator begin the Diffie-Hellman Group exchange. (Cleartext)
4. Responder begin the Diffie-Hellman Group exchange. (Cleartext)
5.  Initiator begins authentication exchange including the hash and IKE identity. (Encrypted)
6. Responder begins authentication exchange including the hash and IKE identity. (Encrypted)

**AM** is less secure due to that Authentication exchange & IKE identities go unprotected using only 3 messages described below:
1. Initiator sends the first message containing ISAKMP Policy, Diffie-Hellman Group, and IKE identity. (Cleartext)
2. Responder replies with a selected policy that its capable of and agrees with based on priority. Additionally the Responder includes an authentication hash for integrity. (Cleartext)
3. Initiator sends it's authentication hash. (Depends on implementation)

IKEv1 Phase 1 Policies must match between the VPN peers for the following parameters:
- Encryption: DES, 3DES, AES (128, 192, 256)
- Hash: MD5, SHA-1, SHA-2 (256, 384, 512)
- Diffie-Hellman Group: 1,2,5,14,15,16, 19, 20, and 24
- Authentication Method: PSK, PKI (RSA or ECDSA)
- (OPTIONAL) Lifetime
### IPsec / IKEv1 Phase 2
Phase 2 will be always be performed in **Quick Mode (QM)** to define the specific encryption and integrity parameters used to protect the data plane.

IKEv1 Phase 2 negotiation requires the following parameters must match between peers:
- Transform-Set 
	- Encryption: ESP 
	- Hashing: ESP or AH 
	- Encapsulation Mode: Tunnel (default) or Transport (Requires end to end communication)
- Proxy Identities: Describes the traffic to be encrypted typically via an ACL (0.0.0.0/0, GRE/47, etc)
	- This should be a mirror image between peers.
- (OPTIONAL) Perfect Forward Secrecy (PFS): Enables DH exchange to derive a fresh set of symmetric keys.

### IKE Packet Capture
Overview: Here we can see the ISAKMP SA be negotiated and initial tunnel form the Control-Plane for further negotiation of the IPsec SA.
![Image Description](/images/IPsec-w-CryptoMap.png)
1\. Router2 makes the because source traffic from 10.2.5.2 to 10.1.1.10 initiated the IPsec SA. It then provides ISAKMP policy parameters to Router1.
![Image Description](/images/IPsec-w-CryptoMap-Proposal2.png)
3\. Router2 starts the DH key exchange and provides the Nonce. After Router1 responds and completes the DH exchange the rest of the ISAKMP SA and IPsec SA take place in an encrypted tunnel and cannot be observed.
![Image Description](/images/IPsec-w-CryptoMap-Key-n-Nonce2.png)

### NAT-D & NAT-T
 During the **NAT-Discovery (NAT-D)** exchange, each peer calculates a **hash** (typically SHA-1) of its own and the peer’s **IP address + UDP port** information as it knows it locally.

Each side sends these hashes to the other inside NAT-D payloads.  
Then, upon receiving them, each peer recomputes what the hashes **should** be based on the IP/port values **it actually sees in the received packet headers**.

If those hashes **don’t match**, it means something in the path (a NAT device) has changed the source or destination IP/port. Once this is detected the peers unable NAT Traversal (UDP 4500)his can be observed below:

```
R2# show crypto isakmp sa detail

Codes: N - NAT-traversal

IPv4 Crypto ISAKMP SA

C-id  Local           Remote          I-VRF  Status Encr Hash   Auth DH Lifetime Cap.

1013  2.2.2.2         1.1.1.1                ACTIVE aes  sha384 psk  14 23:47:45 N   
       Engine-id:Conn-id =  SW:13

R2# show crypto ips sa detail   

interface: GigabitEthernet0/1
    Crypto map tag: CM1, local addr 2.2.2.2

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (10.2.5.0/255.255.255.0/0/0)
   remote ident (addr/mask/prot/port): (10.1.1.0/255.255.255.0/0/0)
   current_peer 1.1.1.1 port 4500
```

**NAT-D Packet Capture**
 UDP PORT 500![[IPsec-w-CryptoMap-NAT-D.png]]
 **NAT-T Packet Capture**
 UDP PORT 4500![Image Description](/images/IPsec-w-CryptoMap-NAT-T.png)


Packet Captures: [here](/downloads/an_overview_of_ipsec.tar)

SHA256: 