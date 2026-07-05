# WraithLab
Production-grade documentation, architecture design, and deployment writeups for WraithLab — my enterprise-mimicking homelab sandbox.

## 🌐 Network Architecture & Topology

The core of the network is powered by OPNsense running on a Protectli Vault, which handles routing, firewall rules, and core network security straight from the ONT. Traffic is distributed via an 8-port managed switch using targeted VLAN segmentation.

```
VLAN ID  |  Name        |  Subnet Scope |  Services
---------------------------------------------------------------------------------------------

VLAN 10  | MGMT         | Main PC       | Secure access to network infrastructure management

VLAN 20  | Labs         | 2x Pi 3B+     | OWASP Juice Shop

VLAN 30  | IDS          | Pi 4B         | Suricata, Log2Ram, Cron, Webhook script

VLAN 40  | WiFi         | Router        | Router in AP-only mode
```

## Phases
* **Phase 1:** OPNSense
* **Phase 2:** Managed Switch
* **Phase 3:** Suricata/Pi 4B Setup
* **Phase 4:** Pi 3B+/OWASP Juice Shop
