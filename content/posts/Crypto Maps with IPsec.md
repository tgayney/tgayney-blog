---
title: Crypto Maps with IPsec
date: 2025-11-05T12:00:00-05:00
draft: false
---
**IPsec with Crypto Maps** provides a method for securing IP communication between networks through encryption and authentication. **ISAKMP (Internet Security Association and Key Management Protocol)** establishes the **IKE Phase 1 tunnel** used to authenticate peers and negotiate the parameters for further communication. Crypto Maps act as the policy framework that binds IPsec settings such as peers, transform sets, and access lists to a physical (sub)interface. When traffic matches the defined crypto ACL, it is encrypted according to the transform set and sent to the specified peer through an IPsec tunnel.

This approach enables administrators to selectively protect traffic, maintain flexible security policies, and ensure confidentiality and integrity across untrusted networks. By combining ISAKMP/IKE for key exchange with Crypto Maps for policy enforcement, IPsec delivers a robust and standards-based foundation for secure site-to-site connectivity.

#### Configuring Crypto Maps With IPsec
##### Configuring ISAKMP Policy / IKE Phase 1
Before we can begin configuring **crypto maps**, we must first define the four key parameters that make up the **ISAKMP (IKE Phase 1) policy**:
- **Encryption**
- **Hashing**
- **Authentication**
- **Diffie–Hellman Group**

While higher bit-length encryption and stronger algorithms generally provide better security, they also introduce greater processing overhead. When selecting these parameters, it’s important to strike a balance between **security, performance, and compatibility**. Configuring the most secure ISAKMP policy is possible, but we must also consider whether our VPN peers support the chosen algorithms and whether our device’s CPU can sustain the cryptographic load especially as the number of IPsec tunnels increases. 

For this lab we only be configuring one ISAKMP policy but in a real world environment multiple policies would be configured to match the parameters of initiator's/peer's proposal of IKE Phase 1. We will run AES 192 and sha256 across the **Control Plane** then select pre-shared key "cisco123" because running the alternative being PKI would require additional configuration outside the scope of this post. Finally we will set DH group to 14:
```IOSv
R1(config)#crypto isakmp policy 10
R1(config-isakmp)# encryption aes 192
R1(config-isakmp)# hash sha256
R1(config-isakmp)# authentication pre-share
R1(config-isakmp)# group 14
!
R1(config)# crypto isakmp key cisco123 address 2.2.2.2
```


##### Configuring IPsec Policy / IKE Phase 2
Before we're able to proceed we must configure our **Transform-Set** this will specify the encryption algorithm and hashing in our **Data Plane**, additional the mode of our encapsulation being **Tunnel** vs **Transport** by default encapsulation will operate in Tunnel mode so we will leave alone. 
(Control Plane and Data Plane encryption algorithms do not have to match but ISAKMP & IPsec policies must match between peers)

After the preceding has been accomplished we will configure an **Extended ACL** to specify what source traffic should be encrypted to a destination:
```IOSv
R1(config)# crypto ipsec transform-set TS_NAME esp-aes 192 esp-sha-hmac 
!
R1(config)# ip access-list extended IPSEC_ACL
R1(config-ext-nacl)#10 permit ip 10.1.1.0 0.0.0.255 10.2.5.0 0.0.0.255
```

##### Defining The Crypto Map 
The other options available here include GETVPN and manual keying which doesn't scale well, we will select `ipsec-isakmp`. Here we will define the Crypto map to use the previously created **Transform-Set** and encrypt traffic specified by our Extended ACL.

For path redundancy we will also configure the tunnel source as our loopback

Finally we apply our previously created Crypto Map to any given physical (sub)interface egress of our peer:
```
R1(config)#crypto map CM_NAME 10 ipsec-isakmp 
R1(config-crypto-map)#set transform-set TS_NAME
R1(config-crypto-map)#set peer 2.2.2.2
R1(config-crypto-map)#match address IPSEC_ACL
!
R1(config)# crypto map CM_NAME local-address loopback0
!
R1(config-if)# crypto map CM_NAME
```

##### IKE Phase 1 & 2 Packet Inspection
| Step | Direction             | Payloads                   | Purpose                                             |
| ---- | --------------------- | -------------------------- | --------------------------------------------------- |
| 1    | Initiator → Responder | SA proposal                | Proposes encryption, hash, auth, DH group, lifetime |
| 2    | Responder → Initiator | SA response                | Chooses acceptable policy                           |
| 3    | Initiator → Responder | Key Exchange + Nonce       | Begins Diffie-Hellman key exchange                  |
| 4    | Responder → Initiator | Key Exchange + Nonce       | Completes Diffie-Hellman exchange                   |
| 5    | Initiator → Responder | Identification + Hash/Auth | Authenticates initiator                             |
| 6    | Responder → Initiator | Identification + Hash/Auth | Authenticates responder                             |


Overview: Here we can see the ISAKMP SA be negotiated and initial tunnel form the Control-Plane for further negotiation of the IPsec SA.
![Image Description](/images/IPsec-w-CryptoMap.png)
1\. Router2 makes the because source traffic from 10.2.5.2 to 10.1.1.10 initiated the IPsec SA. It then provides ISAKMP policy parameters to Router1.
![Image Description](/images/IPsec-w-CryptoMap-Proposal2.png)
3\. Router2 starts the DH key exchange and provides the Nonce. After Router1 responds and completes the DH exchange the rest of the ISAKMP SA and IPsec SA take place in an encrypted tunnel and cannot be observed.
![Image Description](/images/IPsec-w-CryptoMap-Key-n-Nonce2.png)
##### IPsec NAT-D & NAT-T
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

**NAT-D** !![Image Description](/images/IPsec-w-CryptoMap-NAT-D.png)
 
 **NAT-T**!![Image Description](/images/IPsec-w-CryptoMap-NAT-T.png)
 
Lab & Packet Captures: [here](/downloads/crypto-maps-w-ipsec.tar)
sha256sum: 47c4eeebe5c177e8e856618f0cd42fff12195acddced7ab83cb0d4a399480c06