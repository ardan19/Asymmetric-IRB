# EVPN-VXLAN Data Center Fabric: Asymmetric IRB Architecture

## 📖 Overview
This repository contains the High-Level Design (HLD), architectural decisions, and configuration templates for a modern, highly available Data Center fabric. The network is built on a Spine-Leaf (Clos) topology utilizing **EVPN-VXLAN** with an **Asymmetric Integrated Routing and Bridging (IRB)** model. 

Designed for maximum bandwidth utilization and fault tolerance, this fabric leverages **OSPF Area 0** for a robust underlay and an **All-Active** multi-homing strategy using EVPN ESI-LAG.

## 🏗️ Topology Architecture
The physical underlay is structured to separate East-West (server-to-server) and North-South (external) traffic, ensuring optimal performance and scalability:
* **Spine Layer:** 2x Spine switches (`spine1`, `spine2`) acting as the high-speed backplane and iBGP Route Reflectors.
* **Compute Leaf Layer:** 4x Leaf switches (`leaf1` to `leaf4`, deployed in pairs) providing connectivity to compute endpoints.
* **Border Leaf Layer:** 2x Dedicated Border switches (`border1`, `border2`) handling ingress/egress traffic to the external network (`external-sw`).

## Physical Topology
<img width="3693" height="3813" alt="image" src="https://github.com/user-attachments/assets/1d00770f-9801-42e9-b02f-481a7a6f4714" />

## Traffic Flow
### 1. Traffic Flow Host-A to Host-B
<img width="3609" height="1986" alt="image" src="https://github.com/user-attachments/assets/3a3e70d8-7c72-4014-808c-26bb88442d6c" />


### 2. Traffic Flow Host-A to Host-C
<img width="2151" height="4113" alt="image" src="https://github.com/user-attachments/assets/952b5758-f933-4077-af76-78e8ad5c6002" />


### 3. Traffic Flow Host-C to Host-B
<img width="2139" height="4053" alt="image" src="https://github.com/user-attachments/assets/6d9e555b-bc2c-474a-97b3-c324159eaec4" />


## Configuration Detail
<img width="3663" height="4503" alt="image" src="https://github.com/user-attachments/assets/00f17eaa-cc34-43ea-943a-a8c36b8a7bcf" />

## ⚙️ Key Technical Specifications

### 1. Data Plane: Asymmetric IRB
* **Ingress Routing & Egress Bridging:** Routing between different subnets occurs exclusively at the ingress VTEP. The packet is then encapsulated into the destination's Layer 2 VNI and bridged (Layer 2 forwarded) at the egress VTEP.
* **VLAN Everywhere:** All required VLANs and VNIs are provisioned across all participating Leaf switches to support localized routing, removing the need for a dedicated L3 Transit VNI.

### 2. Underlay Network: OSPF Area 0
* **Single-Area OSPF:** Provides simple, fast, and reliable IP reachability between all VTEP loopbacks.
* **Optimized Links:** All Fabric links (Spine-to-Leaf) are configured as OSPF `point-to-point` network types to eliminate DR/BDR election overhead and speed up convergence.
* **BFD Integration:** Bidirectional Forwarding Detection (BFD) is tied to OSPF for sub-second link failure detection.

### 3. Overlay Network: MP-BGP EVPN
* **Control Plane:** iBGP is utilized for the EVPN control plane to efficiently distribute MAC and IP reachability information across the fabric.
* **Route Reflectors:** To ensure scalability, the Spine switches are configured as BGP Route Reflectors, eliminating the need for a full-mesh BGP topology among the Leaves.

### 4. High Availability & ECMP (All-Active)
* **Fabric ECMP:** Equal-Cost Multi-Path (ECMP) is fully enabled at the routing layer. Traffic between Leaves is hashed (5-tuple) and distributed evenly across both Spines, ensuring 100% link utilization with no Spanning Tree blocked ports.
* **All-Active Edge Multi-Homing:** Compute nodes connect to Leaf pairs using **MLAG** (Multi-Chassis Link Aggregation). Both links actively forward traffic simultaneously, providing maximum throughput and seamless failover.
