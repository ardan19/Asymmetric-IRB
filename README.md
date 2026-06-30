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
<img width="3693" height="3813" alt="image" src="https://github.com/user-attachments/assets/e8a6682d-c7c5-4f59-856a-8e0860f7a191" />


## Traffic Flow
### 1. Traffic Flow Host-A to Host-B
<img width="3609" height="1986" alt="image" src="https://github.com/user-attachments/assets/3a3e70d8-7c72-4014-808c-26bb88442d6c" />
* **Workflow**
Host-A and Host-B are in the different VLAN but are located across different leafs.
* 1. Host-A makes a LACP hash decision and forwards traffic to leaf1 or leaf2.
* 2. leaf1 or leaf2 does a layer 2 lookup and has the MAC address for Host-B, leaf1 or leaf2 performs routing from VLAN10 to VLAN20 and found Host B is on the other side of the fabric and associated with L2 VNI 1020.
* 3. leaf1 or leaf2 performs VXLAN encapsulation packet out VNI1020.
* 4. leaf1 or leaf2 does hashing based outer 5-tuple using underlay route through spine1->leaf4.
* 5. leaf3 or leaf4 receive the packet in and performs VXLAN decapsulation packet.
* 6. leaf3 or leaf4 does not performs any routing and has the MAC address for Host-B.
* 7. Finally, leaf3 or leaf4 does bridging the packet to physical port connected to the Host-B. 

### 2. Traffic Flow Host-A to Host-C
<img width="2151" height="4113" alt="image" src="https://github.com/user-attachments/assets/952b5758-f933-4077-af76-78e8ad5c6002" />
* **Workflow**
Host-A and Host-C are in the different Network.
* 1. Host-A makes a LACP hash decision and forwards traffic to leaf1 or leaf2.
* 2. leaf1 or leaf2 does a layer 3 lookup and has the MAC address for Host-C, leaf1 or leaf2 performs routing and found Host C is on the other side of the Network.
* 3. leaf1 or leaf2 performs VXLAN encapsulation packet out VNI1010.
* 4. leaf1 or leaf2 does hashing based outer 5-tuple using underlay route through spine1 or spine2->border1 or border2.
* 5. border1 or border2 receive the packet in and performs VXLAN decapsulation packet.
* 6. border1 or border2 transfer the packet out external-sw using OSPF routing protocol. 
* 6. external-sw does not performs any routing and has the MAC address for Host-B.
* 7. Finally, external-sw does transfer the packet to physical port connected to the Host-C. 

### 3. Traffic Flow Host-C to Host-B
<img width="2139" height="4053" alt="image" src="https://github.com/user-attachments/assets/6d9e555b-bc2c-474a-97b3-c324159eaec4" />
* **Workflow**
Host-C and Host-B are in the different Network.
* 1. Host-C transfer packet to external-sw.
* 2. external-sw does a layer 3 lookup and has the MAC address for Host-B on the other side of the Network.
* 3. external-sw does hashing based outer 5-tuple using underlay route through border1 or border2.
* 4. packet arrive in border1 or border2 and found the host-C is on the side of leaf3 and leaf4 fabric and associated with L2 VNI1020.
* 5. border1 or border2 performs VXLAN encapsulation packet and forward the packet through spine2 and spine2 makes a LACP hash decision and forwards traffic to leaf3 or leaf4.
* 6. leaf3 or leaf4 receive the packet in and performs VXLAN decapsulation packet.
* 7. Finally, leaf3 or leaf4 does bridging the packet to physical port connected to the Host-B.

## Configuration Detail
<img width="3663" height="4503" alt="image" src="https://github.com/user-attachments/assets/a318bacf-c4ff-41e3-b31d-5b06179e937f" />


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
* **Fabric ECMP:** Equal-Cost Multi-Path (ECMP) is fully enabled at the routing layer. Traffic between Leaves and Borders is hashed (5-tuple) and distributed evenly across both Spines, ensuring 100% link utilization with no Spanning Tree blocked ports.
* **All-Active Edge Multi-Homing:** Compute nodes connect to Leaf pairs using **MLAG** (Multi-Chassis Link Aggregation). Both links actively forward traffic simultaneously, providing maximum throughput and seamless failover.
