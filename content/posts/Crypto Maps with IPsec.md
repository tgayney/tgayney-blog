---
title: Crypto Maps with IPsec
date: 2025-11-05T12:00:00-05:00
draft: false
---
**IPsec with Crypto Maps** provides a method for securing IP communication between networks through encryption and authentication commonly referred to as site-to-site. **ISAKMP (Internet Security Association and Key Management Protocol)** establishes the **IKE Phase 1 tunnel** used to authenticate peers and negotiate the parameters for further communication. Crypto Maps act as the policy framework that binds **IPsec/IKE Phase 2** settings such as peers, transform sets, and access lists to a physical (sub)interface. When traffic matches the defined crypto ACL, it is encrypted according to the transform set and sent to the specified peer through an IPsec tunnel.

This approach enables administrators to selectively protect traffic, maintain flexible security policies, and ensure confidentiality and integrity across untrusted networks. By combining ISAKMP/IKE for key exchange with Crypto Maps for policy enforcement, IPsec delivers a robust and standards-based foundation for secure site-to-site connectivity.

#### Configuring Crypto Maps With IPsec
##### Configuring The ISAKMP Policy That Is IKE Phase 1
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


##### Configuring IPsec Transform-Set and Crypto Map That Make Up IKE Phase 2
Before we're able to proceed we must configure our **Transform-Set** this will specify the encryption algorithm and hashing in our **Data Plane**, additional the mode of our encapsulation being **Tunnel** vs **Transport** by default encapsulation will operate in Tunnel mode so we will leave alone. 
(Control Plane and Data Plane encryption algorithms do not have to match but ISAKMP & IPsec policies must match between peers)
- Encryption
- Hashing
- Encapsulation Mode


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

Lab: [here](/downloads/ipsec_with_cryptomap.tar)

sha256sum: 