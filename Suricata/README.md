# Phase 3: Building a Hardened, Low-Wear Network Intrusion Detection System (NIDS) Sensor

This technical brief documents the design, implementation, and verification of a low-wear, high-performance Network Intrusion Detection System (NIDS) sensor deployed on resource-constrained hardware (Raspberry Pi 4B). The system utilizes Suricata to parse mirrored network traffic from an infrastructure backbone tap (Protectli Vault), buffers all active input/output transactions to an ephemeral Log2Ram RAM disk to prevent physical storage wear, and pushes sanitized, real-time security alerts to an operations center dashboard via an automated Python and Discord Webhook pipeline.

🛠 Architecture & Component DesignThe sensor deployment is engineered across four structural layers:

* **Packet Capture & Inspection Layer (Suricata):** Binds to a local network interface to aggressively analyze frames against thousands of known malicious structural definitions. **Implemented via a dual-NIC configuration:** a stealth interface stripped of an IP allocation running in pure promiscuous mode to ingest SPAN mirror traffic, paired with a secondary isolated physical interface linked to the SEC_MONITOR management plane.

* **Hardware Protection Layer (Log2Ram):** Mounts an isolated temporary filesystem (tmpfs) in memory, intercepting all directory operations hitting /var/log to eliminate persistent write amplification on physical storage media (SD Cards/SSDs).

* **Telemetry Sanitization & Pipeline Layer (Python/Systemd):** Continuously reads raw JavaScript Object Notation (JSON) alert metadata directly from volatile memory, strips internal infrastructure data to protect topology privacy, and pipes telemetry to an operational endpoint.

* **Self-Healing Automation Layer (Bash/Cron):** Periodically reviews the volatile storage layout, executing memory-to-disk flushes and non-destructive log truncations if disk space thresholds are crossed.

```
  [ Mirrored Network Traffic (Protectli Vault Port 1) ]
                                             │
                                             ▼
  [ Hardened Suricata Packet Capture Engine (RAM) ]
                                             │
                                             ▼
   ┌─────────────────────────────────── Log2Ram Mount ───────────────────────────────────┐
   │                                                                                     │
   │  /var/log/suricata/eve.json  ◄─── [ Automated Bash Engine: Truncates if > 80% ]     │
   │            │                                                                        │
   └────────────┼────────────────────────────────────────────────────────────────────────┘
                │
                ▼
     [ Python Forwarder Script ] ───► (Strips 10.10.x.x Topologies) ───► [ Discord Webhook ]
```     
     
## 💻 Technical Implementation Steps

### 1. Ephemeral Storage Provisioning (Log2Ram)

To prevent physical media degradation from volatile log structures like eve.json, a 256M memory logging buffer was provisioned:

* **Fetch and apply the official Log2Ram deployment framework**

```curl -L https://github.com/azlux/log2ram/archive/master.tar.gz | tar zxf -```

```cd log2ram-master && chmod +x install.sh && sudo ./install.sh```

```cd .. && rm -rf log2ram-master```

* **Configure allocation boundary limits**

```sudo nano /etc/log2ram.conf```

Set: `SIZE=256M`

* **Apply filesystem mounts via reboot**

```sudo reboot```

### 2. High-Performance Hardening of Suricata

To prevent Suricata from exhausting the limited RAM disk footprint via background logging metrics, the orchestration layout inside /etc/suricata/suricata.yaml was hardened to suppress high-volume telemetry fields:

* **Global Performance Adjustments:** max-pending-packets: 1024 and mpm-algo: ac (optimized for ARM architectures).

* **Global Engine Tuning:** Global status logs were throttle-restricted to reduce polling overhead:


```
stats:
  enabled: yes
  interval: 3600  # Restricts updates to 1-hour windows to shield RAM disk space
```

* **Strict Alert Output Targeting:** The eve-log structural array was restricted to capture zero telemetry types except explicit alerts, explicitly dropping raw packets and Base64 payloads to preserve memory metrics:

```
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
      types:
        - alert:
            payload: no
            packet: no
```

* **Telemetry Suppression:** Duplicate fast logs and performance tracing logs (- fast, - stats, - http, - dns) were removed from the outputs: list to halt duplicate disk processing cycles.

### 3. Rule Optimization and Footprint Mitigation

Loading unneeded rules causes signature bloat and risks Out-of-Memory (OOM) kernel crashes. To mitigate this, a custom exclusion file was introduced to target heavy background telemetry:

```sudo nano /etc/suricata/disable.conf```

* **Pasted targeting arrays:**

```
group:emerging-info.rules
group:emerging-policy.rules
group:emerging-retired.rules
```

* **Synchronize and process the signatures via the update tool:**

```sudo suricata-update --disable-conf /etc/suricata/disable.conf```
```sudo systemctl restart suricata```

**Validation Trace Metrics:** 66,826 rules loaded, 4,891 successfully filtered and disabled.

### 4. Topology Sanitization & Discord Pipeline Configuration

A background daemon script was built in Python to tail the RAM log natively, split incoming telemetry logs, sanitize local IP groupings, and push metadata to an external endpoint:

```sudo nano /usr/local/bin/suricata_to_discord.py```

```
import time, os, json, requests

LOG_FILE = "/var/log/suricata/eve.json"
DISCORD_WEBHOOK_URL = "https://discord.com/api/webhooks/YOUR_WEBHOOK_ID_HERE/YOUR_WEBHOOK_TOKEN_HERE"

def sanitize_ip(ip_address):
    """Masks internal 10.10.x.x topologies to shield corporate infrastructure layout."""
    if ip_address.startswith("10.10."):
        parts = ip_address.split('.')
        return f"{parts[0]}.{parts[1]}.x.x"
    return ip_address

def send_to_discord(alert_data):
    alert_info = alert_data.get("alert", {})
    src_ip = sanitize_ip(alert_data.get("src_ip", "Unknown"))
    dest_ip = sanitize_ip(alert_data.get("dest_ip", "Unknown"))
    
    payload = {
        "embeds": [{
            "title": f"🚨 Suricata Alert: {alert_info.get('signature', 'Unknown Signature')}",
            "color": 15158332,
            "fields": [
                {"name": "Severity", "value": str(alert_info.get("severity", "Unknown")), "inline": True},
                {"name": "Category", "value": alert_info.get("category", "Unknown"), "inline": True},
                {"name": "Traffic Direction", "value": f"`{src_ip}` ➡️ `{dest_ip}`", "inline": False}
            ],
            "timestamp": alert_data.get("timestamp")
        }]
    }
    try:
        response = requests.post(DISCORD_WEBHOOK_URL, json=payload)
        if response.status_code != 204:
            print(f"Discord API error: {response.status_code}")
    except Exception as e:
        print(f"Network error sending to Discord: {e}")

def watch_log():
    """Continuously monitors the eve.json log using a trailing logic loop."""
    if not os.path.exists(LOG_FILE):
        print(f"Waiting for log file {LOG_FILE} to be created...")
        while not os.path.exists(LOG_FILE):
            time.sleep(2)

    with open(LOG_FILE, "r") as f:
        # Move pointer to the end of the file to catch only NEW logs
        f.seek(0, os.SEEK_END)
        
        while True:
            try:
                if os.path.getsize(LOG_FILE) < f.tell():
                    f.seek(0)
            except FileNotFoundError:
                time.sleep(2)
                continue

            line = f.readline()
            if not line:
                time.sleep(0.5)
                continue
                
            try:
                log_data = json.loads(line)
                if log_data.get("event_type") == "alert":
                    alert_obj = log_data.get("alert", {})
                    severity = alert_obj.get("severity")

                    # Allow Severity 1 and 2 while dropping severity 3
                    if severity and severity <= 2:
                        send_to_discord(log_data)
            except json.JSONDecodeError:
                continue

if __name__ == "__main__":
    watch_log()
```

The automated pipeline was containerized into a persistent background system service:

```sudo nano /etc/systemd/system/suricata-discord.service```

```
[Unit]
Description=Suricata to Discord Forwarder
After=suricata.service

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/suricata_to_discord.py
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
```

```sudo systemctl enable --now suricata-discord.service```

### 5. Automated, Non-Destructive Storage Self-Healing

To prevent unexpected logging spikes from filling the 256M memory allocations, an automation script flushes data down to persistent storage and truncates active files if memory thresholds are crossed:

```sudo nano /usr/local/bin/log2ram_monitor.sh```

```
#!/bin/bash
WEBHOOK_URL="https://discord.com/api/webhooks/YOUR_WEBHOOK_ID_HERE/YOUR_WEBHOOK_TOKEN_HERE"
THRESHOLD=80
USAGE=$(df -h | grep "log2ram" | awk '{print $5}' | sed 's/%//')

if [ -n "$USAGE" ] && [ "$USAGE" -gt "$THRESHOLD" ]; then
    # Flush logs cleanly from memory down to persistent hard storage disks
    log2ram write >/dev/null 2>&1
    
    # Non-destructively zero-out files without severing active running process descriptors
    if [ -f /var/log/suricata/eve.json ]; then
        truncate -s 0 /var/log/suricata/eve.json
    fi
    
    if [ -f /var/log/suricata/fast.log ]; then
            truncate -s 0 /var/log/suricata/fast.log
    fi
    
    NEW_USAGE=$(df -h | grep "log2ram" | awk '{print $5}' | sed 's/%//')
    PAYLOAD=$(cat <<EOF
{
  "embeds": [{
    "title": "⚡ Log2Ram Auto-Clearing Action",
    "color": 3066993,
    "description": "RAM disk exceeded safety parameters. Executed automated persistent flush and log truncation.",
    "fields": [
      {"name": "Trigger Space", "value": "\`${USAGE}%\`", "inline": true},
      {"name": "Post-Flush Space", "value": "\`${NEW_USAGE}%\`", "inline": true}
    ]
  }]
}
EOF
)
    curl -H "Content-Type: application/json" -X POST -d "$PAYLOAD" "$WEBHOOK_URL"
fi
```

```sudo chmod +x /usr/local/bin/log2ram_monitor.sh```

```sudo crontab -e```

Added line to evaluate metrics every 30 minutes:

```*/30 * * * * /bin/bash /usr/local/bin/log2ram_monitor.sh >/dev/null 2>&1```

### 6. 🔍 Validation & Pipeline Proof Verification

To prove the validity of the detection engine, filtering array parameters, and sanitization code strings under simulated stress conditions, an attack vector test signature was fired at the network adapter:

```curl http://testmynids.org/uid/index.html```

**1. Data Source Proof Verification via DNS**

To ensure traffic data metrics tracked by the sensor mapped directly onto legitimate threat endpoints, local mapping parameters were verified via standard lookups:

```nslookup testmynids.org```

```
Non-authoritative answer:
Name:   testmynids.org
Address: 3.168.2.70
Address: 3.168.2.54
```

**2. Live Pipeline Response Output (Operational Center Feed)**

Upon encountering the signature, Suricata parsed the exploit pattern and the script captured it instantly out of memory. The final sanitized alert popped up perfectly inside the operational channel dashboard:

```
Alert Title: 🚨 Suricata Alert: GPL ATTACK_RESPONSE id check returned root
Severity Group: 2
Category Class: Potentially Bad Traffic
Traffic Direction Ledger: 3.168.2.70 ➡️ 10.10.x.x
```

Notice the Traffic Direction Ledger metric. The source IP (3.168.2.70) is a public cloud infrastructure endpoint hosted outside our bounds on Amazon Web Services and was left raw to allow analysts to trace the threat origins. However, the destination host was automatically evaluated, intercepted, and obfuscated to 10.10.x.x.

This completely proves the security script logic is functioning perfectly: safeguarding core network topology maps from data leaks while providing total external threat intelligence visibility in real-time.

### 7. Deployment phase

**1. Execute the following commands to configure systemd-resolved and establish a global DNS scope:** 

```sudo mkdir -p /etc/systemd/resolved.conf.d```

```sudo nano /etc/systemd/resolved.conf.d/dns_servers.conf```

**Enter the following blocks into the dns_servers.conf file:**

```
[Resolve]
DNS=<DNS_1> <DNS_2>
FallbackDNS=<Fallback_DNS>
```
Note: Replace everything in <> with your actual DNS addresses. 

For example: `1.1.1.1` (cloudflare)

If you don't have experience with networking, please research before establishing custom DNS configurations on your network.

**2. Restart the resolution daemon and flush the local cache to apply the updates:**

```sudo systemctl restart systemd-resolved```

```sudo resolvectl flush-caches```

**3. Force an immediate network time synchronization with the NTP atomic clock**

```sudo timedatectl set-ntp false```

```sudo timedatectl set-ntp true```

**4. Verify the date/time settings are accurate**

```date```

NOTE: Once the data is correct, the cron and scripts should be operational. The best method for sniffing and webhook functionality involves using 2 ethernet connections. One on the port mirroring your network (with no assigned IP information) and another on the dedicated VLAN for the Pi using a USB to RJ45 adapter. 

The initial setup and testing was done via WiFi, which is not practical for real-time monitoring once connected to the stack via ethernet. In order to make this work, I had to configure the Suricata rules to use wlan0, which was later changed back to eth0 and additional rules were applied to allow promisc.

Furthermore, while this prevents the majority of writing to the disk, it is still advised to keep an eye on your disk writes and storage space. This only solves 99% of the disk writes. I may work on fixing this at a later date.

### 8. Static IP Configuration

If you're using an Ubuntu server on the Raspberry Pi 4B and doing a dual-NIC setup, you will need to configure the network interface a particular way to avoid the interfaces clashing.

First, locate your Netplan configuration file by listing the contents of the directory:

```ls -la /etc/netplan/```

Once you find the YAML file for your network configuration, edit it using `sudo nano /etc/netplan/<file_name>` 

Use the following configuration layout to deploy your dual-NIC setup:

```
network:
  version: 2
  ethernets:
    <VLAN_INTERFACE>: # i.e., enx[MAC_address]
      match:
        macaddress: "<MAC_ADDRESS_ONE>"
      addresses:
      - "<STATIC_IP>/24"
      nameservers:
        addresses:
        - "<VLAN_GATEWAY>" # If you are using DNS-over-TLS through your router (preferred), otherwise specificy your DNS here.
      dhcp4: false
      dhcp6: false
      optional: true
      set-name: "<INTERFACE_NAME>" # i.e., enx[MAC_address]
      routes:
      - to: "default"
        via: "<VLAN_GATEWAY>"
    <SNIFFING_INTERFACE>:
      match:
        macaddress: "<MAC_ADDRESS_TWO>"
      dhcp4: false
      dhcp6: false
      accept-ra: false
      link-local: [ ]
      optional: true
```
