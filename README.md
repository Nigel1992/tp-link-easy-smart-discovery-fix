<div align="center">

# 🔧 TP-Link Easy Smart IPv4 Discovery Fix

### Fix TP-Link Easy Smart Switch discovery problems

[![Linux](https://img.shields.io/badge/Linux-Wine-FCC624?style=for-the-badge&logo=linux&logoColor=black)](https://www.winehq.org/)
[![Windows](https://img.shields.io/badge/Windows-Supported-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://www.microsoft.com/windows)
[![Protocol](https://img.shields.io/badge/Discovery-UDP%20Broadcast-6f42c1?style=for-the-badge)](#)
[![License](https://img.shields.io/badge/License-MIT-2ea44f?style=for-the-badge)](#license)

**Can't find your TP-Link Easy Smart Switch?**

**Can access its web interface?**

**Running the Easy Smart Configuration Utility?**

You may be hitting an IPv4/IPv6 UDP discovery problem.

</div>

---

## 🚀 The Fix

### 🐧 Linux + Wine

Force the bundled Java runtime to use IPv4:

```text
-Djava.net.preferIPv4Stack=true
````

Run the utility with:

```bash
wine "$HOME/.wine/drive_c/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/jre/bin/javaw.exe" -Djava.net.preferIPv4Stack=true -Xmx300m -jar "C:/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/Easy Smart Configuration Utility.exe"
```

**That's the main fix.**

You do **not** need to:

* Disable IPv6
* Change your router
* Change your switch
* Change the switch IP
* Remove virtual network interfaces
* Disable your firewall
* Modify the TP-Link executable

---

## ⭐ Permanent Linux Launcher

Want it to work every time?

Run:

```bash
mkdir -p ~/.local/bin && printf '%s\n' '#!/bin/bash' 'exec wine "$HOME/.wine/drive_c/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/jre/bin/javaw.exe" -Djava.net.preferIPv4Stack=true -Xmx300m -jar "C:/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/Easy Smart Configuration Utility.exe"' > ~/.local/bin/tplink-easy-smart && chmod +x ~/.local/bin/tplink-easy-smart
```

Then launch it with:

```bash
~/.local/bin/tplink-easy-smart
```

---

# 🪟 Windows Users

If the switch is **not detected on Windows**, first check:

```text
1. Can you ping the switch?
2. Can you open its web interface?
3. Is your PC connected to the same LAN/VLAN?
4. Do you have VPN or virtual network adapters?
5. Is Windows Firewall/security software blocking discovery?
```

Open Windows network adapters with:

```text
Win + R → ncpa.cpl
```

Temporarily disable unused VPN/virtual adapters and restart the TP-Link utility.

<details>
<summary>🔧 Windows: Java IPv4 workaround</summary>

If your version of the TP-Link utility allows Java startup parameters to be changed, you can also test:

```text
-Djava.net.preferIPv4Stack=true
```

However, the specific IPv6-mapped IPv4 socket issue documented in this repository was reproduced under **Linux/Wine**.

On Windows, discovery problems can have other causes, particularly multiple network interfaces, VPNs, firewalls and virtual adapters.

</details>

---

# ❓ Switch Works but Utility Can't Find It?

This is possible.

The TP-Link utility uses a **separate UDP discovery mechanism**.

Your switch can therefore be:

```text
Ping              ✅
Web interface     ✅
TCP/80            ✅
UDP discovery     ❌
```

So **don't assume the switch is broken** just because the utility cannot find it.

<details>
<summary>📡 How TP-Link discovery works</summary>

The utility uses UDP broadcast rather than simply scanning the switch's HTTP interface.

The observed discovery traffic uses:

```text
UDP 29809 → UDP 29808
```

and the switch responds in the opposite direction:

```text
UDP 29808 → UDP 29809
```

Conceptually:

```text
┌──────────────┐
│ PC / Utility │
└──────┬───────┘
       │
       │ UDP Broadcast
       │ 29809 → 29808
       ▼
┌──────────────┐
│     LAN      │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ TP-Link      │
│ Easy Smart   │
│ Switch       │
└──────────────┘
       │
       │ UDP response
       │ 29808 → 29809
       ▼
      PC
```

</details>

---

# 🧠 Why Does the IPv4 Fix Work?

<details>
<summary>🔬 Technical explanation</summary>

The Easy Smart Configuration Utility is a Java application bundled with its own Java runtime.

Under the affected Wine configuration, Java creates an IPv6 socket and uses an IPv4-mapped address such as:

```text
::ffff:192.168.x.x
```

The switch's IPv4 UDP broadcast response can still reach the Linux machine.

However, the Java/Wine socket does not correctly deliver that response to the application.

Forcing:

```text
-Djava.net.preferIPv4Stack=true
```

causes Java to use an IPv4 socket instead.

The resulting behavior becomes:

```text
IPv6-mapped socket
        ↓
Java / Wine
        ↓
UDP response not delivered
        ↓
Utility sees nothing
```

versus:

```text
IPv4 socket
        ↓
Java / Wine
        ↓
UDP response received
        ↓
Utility finds switch
```

</details>

---

# 🧪 Still Not Working?

## Linux

Run:

```bash
ip route
```

Find the interface connected to your LAN.

For example:

```text
default via 192.168.178.1 dev eno1
```

Then monitor discovery:

```bash
sudo tcpdump -ni eno1 'udp port 29808 or udp port 29809'
```

Start a scan in the TP-Link utility.

<details>
<summary>📊 Understanding tcpdump results</summary>

You may see:

```text
192.168.x.x.29809 > 255.255.255.255.29808
192.168.x.x.29808 > 255.255.255.255.29809
```

Interpretation:

| Result                                     | Meaning                            |
| ------------------------------------------ | ---------------------------------- |
| No outgoing packet                         | Utility/interface problem          |
| Request but no response                    | Network/firewall/discovery problem |
| Request + response                         | Network discovery is working       |
| Response visible but utility finds nothing | Application/socket problem         |

If you can see the switch's response in `tcpdump` but the application cannot discover the switch, that is a very useful diagnostic clue.

</details>

---

# 🌐 Multiple Network Interfaces

<details>
<summary>🐧 Linux: virtual interfaces</summary>

Run:

```bash
ip addr
```

You may see:

```text
eno1
wlan0
docker0
lxcbr0
virbr0
tailscale0
```

A Java application may select an interface you did not expect.

For example:

```text
              Linux PC
                  │
       ┌──────────┼──────────┐
       │          │          │
       ▼          ▼          ▼
     eno1       lxcbr0      VPN
  192.168.x.x   10.x.x.x   virtual
       │
       ▼
 TP-Link Switch
```

The important interface is the one connected to the same LAN as the switch.

**Do not permanently remove virtual interfaces just to fix the TP-Link utility.**

</details>

<details>
<summary>🪟 Windows: virtual interfaces</summary>

Windows can have:

* Hyper-V
* VMware
* VirtualBox
* WSL
* Docker
* Tailscale
* ZeroTier
* VPN adapters
* Wi-Fi
* Ethernet

Open:

```text
Win + R
ncpa.cpl
```

For testing, temporarily disable unused interfaces.

Then restart the TP-Link utility.

</details>

---

# 🔥 Firewall

<details>
<summary>🐧 Linux firewall</summary>

Check UFW:

```bash
sudo ufw status
```

Other systems may use:

```text
nftables
firewalld
iptables
```

The important thing is that UDP broadcast discovery must be allowed.

</details>

<details>
<summary>🪟 Windows Firewall</summary>

Check:

* Windows Defender Firewall
* Third-party security software
* VPN firewall components

Make sure the TP-Link utility is allowed to communicate on your local/private network.

**Do not immediately disable the entire firewall.**

</details>

---

# 🛠 Quick Troubleshooting

<details>
<summary>✅ Basic connectivity</summary>

* [ ] Switch is powered on
* [ ] PC and switch are on the same LAN/VLAN
* [ ] Switch has an IP address
* [ ] Ping works
* [ ] Web interface works
* [ ] Correct network adapter is active

</details>

<details>
<summary>📡 Discovery</summary>

* [ ] UDP discovery is not blocked
* [ ] No VPN is interfering
* [ ] No unexpected virtual adapter is being selected
* [ ] Firewall allows the application
* [ ] Switch responds to discovery

</details>

<details>
<summary>🐧 Linux + Wine</summary>

* [ ] TP-Link utility starts
* [ ] Bundled Java runtime is used
* [ ] `-Djava.net.preferIPv4Stack=true` is present
* [ ] JVM option appears before `-jar`
* [ ] Correct physical LAN interface is used
* [ ] `tcpdump` shows the switch response

</details>

<details>
<summary>🪟 Windows</summary>

* [ ] Switch responds to ping
* [ ] Web interface is reachable
* [ ] Correct Ethernet/Wi-Fi adapter is active
* [ ] VPN disabled for testing
* [ ] Unused virtual adapters disabled for testing
* [ ] Firewall checked
* [ ] Security software checked
* [ ] TP-Link utility restarted

</details>

---

# 📋 Useful Commands

<details>
<summary>🐧 Linux commands</summary>

### Network interfaces

```bash
ip addr
```

### Routing

```bash
ip route
```

### Test switch

```bash
ping 192.168.x.x
```

### Test web interface

```bash
curl -I http://192.168.x.x/
```

### Monitor TP-Link discovery

```bash
sudo tcpdump -ni eno1 'udp port 29808 or udp port 29809'
```

### Check bundled Java

```bash
wine "$HOME/.wine/drive_c/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/jre/bin/javaw.exe" -version
```

</details>

<details>
<summary>🪟 Windows commands</summary>

Test connectivity:

```cmd
ping 192.168.x.x
```

Open network adapters:

```text
Win + R
ncpa.cpl
```

</details>

---

# 🔍 Technical Evidence

<details>
<summary>Expand technical investigation</summary>

The Linux/Wine issue was reproduced by observing:

1. The switch was reachable through HTTP.
2. The switch responded to UDP discovery.
3. Packet capture showed the response arriving at the Linux host.
4. Java used an IPv6 socket with an IPv4-mapped address.
5. A normal IPv4 Java `DatagramSocket` could receive the response.
6. The TP-Link utility failed to discover the switch without the IPv4 option.
7. Adding:

```text
-Djava.net.preferIPv4Stack=true
```

caused the utility to discover the switch immediately.

This provides a reproducible explanation for the affected Linux/Wine configuration.

</details>

---

# 🧩 Compatibility

<details>
<summary>Expand compatibility information</summary>

The repository focuses on:

* TP-Link Easy Smart Configuration Utility
* Easy Smart Switches
* UDP broadcast discovery
* IPv4/IPv6 socket handling
* Linux + Wine
* Java networking
* Windows multi-interface troubleshooting

Behavior can vary depending on:

* TP-Link utility version
* Switch model
* Hardware revision
* Firmware
* Java version
* Wine version
* Linux distribution
* Windows version
* Firewall configuration
* Network topology
* VLAN configuration
* VPN/virtual adapters

The IPv4 JVM option should therefore be considered a **targeted workaround**, not a universal requirement for every TP-Link switch.

</details>

---

# 📝 Reporting an Issue

If this does not solve your problem, open an issue and provide:

```text
Operating system:
Linux distribution / Windows version:
Wine version:
Java version:
TP-Link Utility version:
Switch model:
Hardware revision:
Firmware:
Network interface:
VPN / virtual adapters:
Ping works:
Web interface works:
UDP response visible:
```

<details>
<summary>🔒 Please remove sensitive information</summary>

Never post:

* passwords
* credentials
* private keys
* public IP addresses
* sensitive network information
* unredacted packet captures

</details>

---

# 🤝 Contributing

<details>
<summary>How to contribute</summary>

Found another TP-Link switch, firmware version or utility version with the same issue?

Contributions and additional test results are welcome.

Useful reports include:

* switch model
* hardware revision
* firmware version
* TP-Link utility version
* operating system
* Wine version
* Java version
* network configuration
* whether UDP discovery responses are visible

</details>

---

# ⚖️ Legal / Clean-Room Note

<details>
<summary>Legal information</summary>

This repository does **not** distribute modified TP-Link software.

It contains:

* troubleshooting information
* networking diagnostics
* configuration guidance
* Java startup options
* documentation of observed behavior

TP-Link software and trademarks remain the property of their respective owners.

No TP-Link executable or proprietary binary is distributed by this repository.

</details>

---

# ⭐ If This Helped

If this fixed your switch discovery problem:

### ⭐ Star the repository

If it did not, open an issue with your diagnostic results so the problem can be investigated further.

---

<div align="center">

### Reachable doesn't always mean discoverable.

`IPv4 UDP Broadcast` → `Java/Wine` → `Easy Smart Utility`

</div>

---

## License

MIT License

Copyright (c) 2026 Nigel1992

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files, to deal in the Software
without restriction, including without limitation the rights to use, copy,
modify, merge, publish, distribute, sublicense, and/or sell copies of the
Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
