# Phase 1: Console-Based Initial Setup 

To eliminate any chance of network lockout, the initial configuration was perfrmed entirely offline using a monitor and keyboard plugged directly into the Protectli Vault. This allowed the core networking interfaces to be rebuilt via the OPNSense Console Menu before initializing the web-based graphical interface.

## 1. First Boot & Interface Assignment

Upon booting the hardware, the system loaded the interactive OPNSense Console Menu

* **Physical Recognition:** The console accurately identified the physical hardware interfaces, noting `igc0` as the designated WAN port and `igc1` as the physical LAN port layer.
* **Interface Assignment and Initial Setup:** `igc1` was provisioned with a static IPv4 address within the `VLAN 10` scope (e.g., `10.10.10.1/24`) to establish the dedicated Management (MGMT) gateway. The integrated DHCP server was explicitly disabled to enforce strict static IP assignment across administrative endpoints. Default factory credentials were immediately rotated to comply with access control hardening standards.
* **Administrative Access:** The administrative workstation was configured with a corresponding static IP within the `10.10.10.0/24` subnet, granting secure local access to the HTTPS WebGUI.

## 2. VLAN Segmentation & Layer 2/3 Assignment

Logical network segmentation was constructed natively via interface > devices and interface > assignments through the HTTPS WebGUI after initial configuration.

## 3. Firewall Policy & Inter-VLAN Routing Constraints

To enforce a zero-trust architecture, strict packet filtering rules were implemented at the firewall layer:

* **Floating Rules (Global Block):** High-priority Floating Rules were established to drop all inter-VLAN traffic targeting the VLAN 10 (MGMT) interface, preventing lateral privilege escalation from lower-privilege zones. An explicit default-deny rule was applied to VLAN 40 (WLAN), isolating wireless clients from all other local RFC 1918 subnets.

* **Interface-Specific Rules (Targeted Administration):** Explicit pass rules were defined on the VLAN 10 interface to permit out-of-band SSH and management traffic strictly to designated endpoints within VLAN 20 (Labs) and VLAN 30 (IDS).
