# Phase 4: Compute Node Provisioning & Vulnerable Workload Staging

**Note:** Due to resource constraints and Juice Shop v20+ requiring an open-weight LLM, I have moved Juice Shop to a cheap laptop with slightly better hardware. The device still has some constraints, which is why I opted to run Juice Shop bare metal instead of in a docker or VM. It is still possible to run Juice Shop bare metal on Pis with more resources, such as the Raspberry Pi 5. For that reason, I will keep the setup process for Juice Shop running bare metal on ARM architecture at the end of this README.

**⚠️ CRITICAL SECURITY WARNING:**
OWASP Juice Shop is an intentionally vulnerable application designed with severe security flaws (including Remote Code Execution, SQL Injection, and Cross-Site Scripting).

Running this application bare-metal exposes your host system directly. It must be deployed on an isolated lab environment or a dedicated testing VLAN. The author assumes absolutely no liability for compromised hosts, data loss, network pivoting, or damages resulting from running this application on a production network or main VLAN. Deploy at your own risk.

## 1. Requirements

`nodejs`
`npm`
`ollama`

## 2. Setup Process

* **Install nodejs and npm:**

  ```sudo apt update && sudo apt install nodejs npm -y```

* **Install ollama:**

  ```curl -fsSL https://ollama.com/install.sh | sh```

* **Pull Gemma4 (e4b):**

  ```ollama pull gemma4:e4b```

* **Clone Juice Shop:**

  ```git clone https://github.com/juice-shop/juice-shop.git --depth 1```

* **Set up Juice Shop:**

  ```cd juice-shop```
  ```npm install```

* **Start Juice Shop:**

  ```npm start```

Once you have installed the prerequisites and started Juice Shop, you will be able to access the vulnerable web application from your browser at `http://localhost:3000`

In order to begin testing the vulnerable web application, I recommend installing `BurpSuite`, `ffuf`, `wordlists`/`seclists`, and `hashcat` or `John the Ripper` (alternatively, you can use `CyberChef` instead of installing `hashcat` or `john`). There are also other alternatives to some of these programs such as `OWASP ZAP`, `gobuster`, etc.

# Juice Shop Bare Metal - ARM Architecture

**Note:** There may be inconsistencies in the full setup and deployment of Juice Shop in this section of the writeup as I was unable to fully test Juice Shop on the Raspberry Pi 3B+ due to resource constraints. While it is possible to run older versions of the vulnerable web app on a resource-constrained device, such as a Pi 3B+, I decided to pivot to a laptop with more resources. In order to run older versions, it is highly likely you will need to troubleshoot this full setup process.

## 1. Operating System Initialization & Host Hardening

A baseline installation of Raspberry Pi OS Lite (64-bit) was deployed out-of-band. Because the underlying system is highly resource-constrained, the configuration emphasizes minimal operational overhead and explicit security controls.

### A. Localization Layout & Password Rotations

To mitigate keyboard mapping conflicts when entering complex cryptographic passphrases during initialization, a progressive password rotation strategy was enforced:

* **An initial non-special character administrative passphrase was defined to guarantee successful localization tracking.**

* **The default United Kingdom keyboard mapping was modified to standard United States ANSI layouts via the console configuration:**

  ```sudo nano /etc/default/keyboard```

  change `XKBLAYOUT="gb"` to `XKBLAYOUT="us"`

  Press `CTRL + O` to save, press `enter`, press `CTRL + X` to exit, press `enter`.

  ```sudo setupcon``` Note: This applies the keyboard layout

* **With the correct layout mapped, the passwd utility was executed to rotate the administrative account to an enterprise-grade passphrase containing alphanumeric characters and special symbols.**

  ```passwd```

### B. Public-Key Authentication & FIDO2 Hardware Attestation

Password-based SSH authentication was discarded in favor of explicit public-key cryptography. For enhanced physical operational security, the deployment leverages an out-of-band FIDO2 hardware key paired with an explicit user passphrase.

* **Provision root SSH directory with absolute restrictive permissions:**
  
  ```mkdir -m 700 ~/.ssh```

* **Mount external hardware media containing the deployment public key:**

  ```lsblk```

  ```sudo mkdir /mnt/usb```

  ```sudo mount /dev/sda1 /mnt/usb```

  ```ls -l /mnt/usb``` Note: lists files on USB

* **Transfer public key and enforce strict filesystem permissions:**

  ```cp /mnt/usb/id_rsa.pub ~/.ssh/authorized_keys```

  ```chmod 600 ~/.ssh/authorized_keys```

  ```cd```

* **Cleanly unmount transient media:**

  ```sudo umount /mnt/usb```

* **Initialize the OpenSSH daemon:**

  ```sudo systemctl enable ssh --now```

### C. Persistent Storage I/O Optimization

To facilitate reliable storage operations when utilizing MicroSD-to-USB hardware translation adapters, the firmware boot properties were tuned to prevent interface race conditions:

```sudo nano /boot/firmware/config.txt```

Add this line to the bottom of the file: `program_usb_timeout=1`

`CTRL + O`, `enter`, `CTRL + X`, `enter`.

### D. Deterministic Network Layer Provisioning

To lock the host onto `VLAN 20` without relying on dynamic DHCP polling overhead, static IP assignments were implemented natively via the NetworkManager Text User Interface (`nmtui`) and command-line engine (`nmcli`):

```sudo nmtui```

* Select `edit a connection` and press enter.

* Select `Wired connection 1` and press enter.

* Navigate to `Automatic` next to IPv4 configuration and press enter. Change it to `Manual`.

* Navigate to `show` next to `IPv4 configuration` (to the right of where you changed it from "automatic" to "manual") and press enter.

* Navigate to `add` and add your static IP. Do the same for DNS.

NOTE: Your gateway needs to match the subnet of your IP. For instance:
If your IP is `10.10.100.x/24`, you will use the gateway `10.10.100.1` (unless you used a different address, such as 10.10.100.250. However, X.X.X.1 is a strong industry convention for gateways).

* Configure your DNS (common DNS servers include `1.1.1.1` or `8.8.8.8`, but you can use whatever DNS provider you prefer).

* Go down and select `ok` and press enter.

* Press `escape` (ESC) and then go down and select `ok` and press enter again.

* Force NetworkManager interface binding:

  ```sudo nmcli connection up "Wired connection 1"```

  ```sudo nmcli connection modify "Wired connection 1" connection.interface-name eth0```

  ```sudo nmcli connection modify "Wired connection 1" ipv6.method disabled```

  ```ip a show eth0``` Note: You should see your static IP if done correctly


## 2. Bare-Metal OWASP Juice Shop Compilation & Systemd Orchestration

Rather than relying on heavy containerization due to resource constraints, the vulnerable testing application was compiled cleanly as a native, bare-metal node application.

```sudo apt update && sudo apt install -y nodejs build-essential git cmake automake libtool pkg-config```

```sudo apt install npm```

```cd /opt```

```sudo git clone https://github.com/juice-shop/juice-shop.git --depth 1```

```cd juice-shop```

```sudo apt install typescript```

```sudo npx tsc``` 

```
cat << 'EOF' | sudo tee /etc/systemd/system/juiceshop.service > /dev/null
[Unit]
Description=OWASP Juice Shop Bare-Metal Service
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/juice-shop
ExecStart=/usr/bin/node /opt/juice-shop/build/app.js
Restart=always
Environment=NODE_ENV=production
Environment=NODE_OPTIONS="--max-old-space-size=512"

[Install]
WantedBy=multi-user.target
EOF
```

```sudo systemctl start juiceshop.service```

**Verify Juice Shop status:**

```sudo systemctl status juiceshop.service```

**NOTE:** For troubleshooting, it may require you to sync the systemd controller files and flush stale error metrics:

```sudo systemctl daemon-reload```

```sudo systemctl reset-failed juiceshop.service```

```sudo systemctl start juiceshop.service```
