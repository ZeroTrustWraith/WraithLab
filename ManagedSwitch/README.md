# Phase 2: Layer 2 Managed Switch Configuration

To support the 802.1Q logical segmentation defined at the firewall layer, an 8-port managed switch was provisioned to handle the physical distribution of VLAN tags, port mirroring, and native ingress filtering.

## 1. Device Hardening & Management Plane Initialization

Before integrating the switch into the active network topology, initial provisioning was conducted via a local loopback connection to isolate the device during administrative bootstrapping:

Credential Hardening: Default factory administrative credentials were rotated to a cryptographically secure passphrase to prevent unauthorized local privilege escalation.

* **Management Interface Assignment:** The switch's default management IP was migrated from the vendor-default subnet to a static IPv4 assignment within VLAN 10 (e.g., 10.10.10.2/24). This locks down the administrative WebGUI/CLI, ensuring it is only accessible to authenticated hosts originating from the dedicated MGMT zone alongside the OPNsense management gateway.

## 2. 802.1Q VLAN Provisioning & PVID Mapping

To enforce strict traffic isolation at Layer 2, the switch fabric was carved into discrete broadcast domains using explicit ingress and egress tagging rules.

### Ingress Filtering and Port Mode Logic:

* **Trunk Configuration (Ports 1 & 8):** Ports 1 and 8 were configured as explicit trunk ports, accepting frames with valid 802.1Q tags for all 4 operational VLANs (10, 20, 30, 40).

* **Access Configuration:** Host-facing ports were stripped of tagging requirements. To prevent VLAN hopping and traffic bleeding, the PVID (Port VLAN ID) on each access port was explicitly locked to match its assigned untagged VLAN ID. Any untagged ingress frame entering these ports is automatically encapsulated with the corresponding VLAN tag at the hardware level.

## 3. Passive Monitoring via Switched Port Analyzer (SPAN)

To facilitate non-intrusive threat detection for the Suricata IDS node, a hardware-level Port Mirroring session was implemented within the switch fabric:

* **Source Port:** Port 1 (The core OPNsense trunk carrying all inter-VLAN and WAN-bound traffic).

* **Destination Port:** Port 8 (Connected directly to the Suricata network interface on the Raspberry Pi 4B).

* **Operational Behavior:** The switch duplicates all ingress and egress frame streams traversing Port 1 and pipes them directly out of Port 8 in promiscuous mode. This allows full-stack packet inspection across all 4 segmented VLANs without injecting latency or modifying production traffic vectors.
