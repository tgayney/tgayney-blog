---
title: Crypto Maps with IPsec
date: 2025-11-04T12:00:00-05:00
draft: false
---
Crypto Map deatils


## Configuring Crypto Maps With IPsec
##### Setting ISAKMP Policy / IKE Phase 1
Before we can begin configuring **crypto maps**, we must first define the four key parameters that make up the **ISAKMP (IKE Phase 1) policy**:
- **Encryption**
- **Hashing**
- **Authentication**
- **Diffie–Hellman Group**

While higher bit-length encryption and stronger algorithms generally provide better security, they also introduce greater processing overhead. When selecting these parameters, it’s important to strike a balance between **security, performance, and compatibility**. Configuring the most secure ISAKMP policy is possible, but we must also consider whether our VPN peers support the chosen algorithms and whether our device’s CPU can sustain the cryptographic load—especially as the number of IPsec tunnels increases.
```IOSv
Router(config-isakmp)# encryption aes 256
Router(config-isakmp)# hash sha256
Router(config-isakmp)# authentication pre-share
Router(config-isakmp)# group 14
```
