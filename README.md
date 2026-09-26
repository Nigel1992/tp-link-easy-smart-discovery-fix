<div align="center">

# 🔧 TP-Link Easy Smart IPv4 Discovery Fix

### Fix TP-Link Easy Smart Switch discovery problems on Linux/Wine and troubleshoot Windows

<br>

[![Linux](https://img.shields.io/badge/Linux-Wine-FCC624?style=for-the-badge&logo=linux&logoColor=black)](https://www.winehq.org/)
[![Windows](https://img.shields.io/badge/Windows-Supported-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://www.microsoft.com/windows)
[![Protocol](https://img.shields.io/badge/Discovery-UDP%20Broadcast-6f42c1?style=for-the-badge)](#-how-discovery-works)
[![License](https://img.shields.io/badge/License-MIT-2ea44f?style=for-the-badge)](#license)

<br>

**Your TP-Link Easy Smart Switch can be fully reachable — yet completely invisible to the configuration utility.**

This repository documents a reproducible **IPv4 discovery workaround** for the TP-Link Easy Smart Configuration Utility, with a particular focus on **Linux/Wine** environments.

</div>

---

## 🚀 Quick Fix

> **Linux/Wine users:** If your switch is not detected, try forcing the bundled Java runtime to use IPv4.

Add:

```text
-Djava.net.preferIPv4Stack=true
````

### One-command launch

```bash
wine "$HOME/.wine/drive_c/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/jre/bin/javaw.exe" -Djava.net.preferIPv4Stack=true -Xmx300m -jar "C:/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/Easy Smart Configuration Utility.exe"
```

<details>
<summary>💡 Why does this work?</summary>

The TP-Link utility uses **UDP broadcast** to discover Easy Smart switches.

Under Wine, the bundled Java runtime can create an IPv6 socket using an IPv4-mapped address.

For example:

```text
::ffff:192.168.x.x
```

In the affected configuration:

```text
Switch
   │
   │ UDP response
   ▼
Linux network stack
   │
   │ response arrives
   ▼
Java / Wine socket
   │
   │ ❌ response not delivered correctly
   ▼
TP-Link Utility
```

Forcing Java to use the IPv4 stack changes the socket behavior:

```text
Switch
   │
   │ IPv4 UDP response
   ▼
Linux network stack
   │
   ▼
IPv4 Java socket
   │
   ▼
TP-Link Utility
   │
   └── ✅ Switch discovered
```

The workaround is:

```text
-Djava.net.preferIPv4Stack=true
```

</details>

---

# 📡 How Discovery Works

The Easy Smart Configuration Utility does **not** simply scan TCP port `80` to find switches.

It uses a separate UDP discovery mechanism.

```text
┌──────────────────────┐
│     PC / Utility     │
│                      │
│ UDP source: 29809    │
└──────────┬───────────┘
           │
           │ UDP Broadcast
           │ 29809 → 29808
           ▼
┌──────────────────────┐
│      LAN / Ethernet  │
│    255.255.255.255   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ TP-Link Easy Smart   │
│       Switch         │
│                      │
│ UDP 29808 → 29809    │
└──────────────────────┘
```

### This is why you can see:

| Test                 | Result  |
| -------------------- | ------- |
| Ping switch          | ✅ Works |
| Open web interface   | ✅ Works |
| Access TCP/80        | ✅ Works |
| Easy Smart discovery | ❌ Fails |

The switch does **not** necessarily have a connectivity problem.

---

# 🐧 Linux / Wine

## ✅ Recommended Fix

Use:

```text
-Djava.net.preferIPv4Stack=true
```

The option must be supplied **before `-jar`**.

### Permanent launcher

Run:

```bash
mkdir -p ~/.local/bin && printf '%s\n' '#!/bin/bash' 'exec wine "$HOME/.wine/drive_c/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/jre/bin/javaw.exe" -Djava.net.preferIPv4Stack=true -Xmx300m -jar "C:/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/Easy Smart Configuration Utility.exe"' > ~/.local/bin/tplink-easy-smart && chmod +x ~/.local/bin/tplink-easy-smart
```

Then simply run:

```bash
~/.local/bin/tplink-easy-smart
```

<details>
<summary>🧠 What exactly does this change?</summary>

Only the Java application's networking behavior.

It does **not**:

* disable IPv6 system-wide
* modify Linux networking
* modify your router
* modify your switch
* change the switch IP
* modify VLAN configuration
* modify the TP-Link executable
* remove virtual interfaces
* require disabling your firewall

It simply tells Java:

```text
Prefer IPv4 sockets instead of the IPv6 networking stack.
```

</details>

---

# 🪟 Windows

Windows users should first determine whether the problem is basic connectivity or discovery.

## 1. Test the switch

Open Command Prompt:

```cmd
ping 192.168.x.x
```

Then open:

```text
http://192.168.x.x
```

If both work but the TP-Link utility cannot find the switch, continue below.

---

## 2. Check Network Adapters

Press:

```text
Win + R
```

and run:

```text
ncpa.cpl
```

Look for unused adapters such as:

* VPN
* Hyper-V
* VMware
* VirtualBox
* WSL
* Docker
* Tailscale
* ZeroTier
* Other virtual adapters

Temporarily disable unused adapters for testing.

Then restart the TP-Link utility.

<details>
<summary>🔍 Why can virtual adapters matter?</summary>

The utility has to choose a network interface for its UDP discovery broadcast.

A PC may have:

```text
Ethernet       → 192.168.x.x
Wi-Fi           → 192.168.x.x
Hyper-V         → 172.x.x.x
VMware          → 192.168.x.x
VPN             → 10.x.x.x
WSL             → virtual
```

The switch may be reachable through Ethernet while the utility attempts discovery through another interface.

</details>

---

## 3. Windows Firewall

The discovery protocol uses UDP broadcast.

Therefore:

```text
HTTP / TCP 80       → may work
Ping / ICMP         → may work
UDP discovery       → may be blocked
```

Check Windows Firewall and third-party security software.

**Do not disable your entire firewall as the first troubleshooting step.**

---

## 4. Java IPv4 Option

If your TP-Link utility installation allows Java startup parameters to be modified, test:

```text
-Djava.net.preferIPv4Stack=true
```

However:

> The IPv6-mapped IPv4 socket issue documented in this repository was specifically reproduced under **Wine**.

For Windows, also investigate:

* network adapter selection
* VPN software
* virtual adapters
* firewall rules
* security software
* multiple active interfaces

---

# 🔬 Technical Investigation

<details>
<summary>Click to expand the technical details</summary>

The TP-Link Easy Smart Configuration Utility is a Java-based application bundled with its own Java runtime.

Relevant Java networking classes include:

```text
java.net.DatagramSocket
java.net.DatagramPacket
java.net.InetAddress
java.net.NetworkInterface
java.net.InetSocketAddress
```

The discovery process uses UDP.

In the affected Wine configuration, Java created an IPv6 socket and bound it to an IPv4-mapped address similar to:

```text
::ffff:192.168.x.x
```

The switch still responded to the broadcast.

The network capture therefore showed:

```text
PC ───── UDP discovery ─────> Switch
PC <──── UDP response ─────── Switch
```

while the Java application did not receive the response correctly.

A normal IPv4 Java `DatagramSocket` could receive the traffic.

Launching the TP-Link utility with:

```text
-Djava.net.preferIPv4Stack=true
```

caused the utility to discover the switch immediately.

### Result

```text
IPv6-mapped socket
        │
        ▼
   Wine / Java
        │
        X
        │
        ▼
   No discovery


IPv4 socket
        │
        ▼
   Wine / Java
        │
        ▼
   Discovery
        │
        ▼
   Switch found
```

</details>

---

# 🧪 Linux Diagnostic

If the switch still cannot be discovered, monitor the UDP traffic.

First find your physical LAN interface:

```bash
ip route
```

Example:

```text
default via 192.168.x.1 dev eno1
```

Then:

```bash
sudo tcpdump -ni eno1 'udp port 29808 or udp port 29809'
```

Start a scan in the TP-Link utility.

You may see:

```text
192.168.x.x.29809 > 255.255.255.255.29808
192.168.x.x.29808 > 255.255.255.255.29809
```

<details>
<summary>📊 How to interpret the capture</summary>

| Result                                     | Possible cause                          |
| ------------------------------------------ | --------------------------------------- |
| No outgoing UDP packet                     | Utility/interface problem               |
| Request but no response                    | Firewall/network/switch discovery issue |
| Request + response                         | Network discovery works                 |
| Response visible but utility finds nothing | Application/socket handling issue       |

If the switch response is clearly visible in `tcpdump` but the application cannot see it, this is an important clue that the physical/network path is functioning.

</details>

---

# 🌐 Multiple Network Interfaces

This is particularly important on Linux.

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

A Java application may select an unexpected interface.

Example:

```text
                    ┌───────────────┐
                    │    Linux PC   │
                    └───────┬───────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
       eno1              lxcbr0            VPN/etc.
   192.168.x.x          10.x.x.x           virtual
          │
          │
          ▼
    TP-Link Switch
```

The important interface is the one connected to the same LAN as the switch.

<details>
<summary>⚠️ Don't permanently disable virtual networking</summary>

Avoid permanently removing or disabling virtual interfaces simply to make the TP-Link utility work.

Prefer an application-level workaround such as:

```text
-Djava.net.preferIPv4Stack=true
```

over changing the entire system's networking configuration.

</details>

---

# 🔥 Firewall Considerations

Discovery uses UDP broadcast.

A firewall may therefore allow:

```text
TCP/80
ICMP
```

while blocking:

```text
UDP broadcast
```

## Linux

For UFW:

```bash
sudo ufw status
```

For nftables/firewalld, inspect the appropriate rules for your system.

## Windows

Check:

* Windows Defender Firewall
* Third-party antivirus/security software
* VPN firewall components
* Network security software

Avoid disabling the entire firewall unless you are deliberately performing a controlled diagnostic test.

---

# 🛠 Troubleshooting Checklist

<details>
<summary>✅ Basic connectivity</summary>

* [ ] Switch is powered on
* [ ] PC and switch are on the same LAN/VLAN
* [ ] Switch has a valid IP address
* [ ] PC can ping the switch
* [ ] Switch web interface opens

</details>

<details>
<summary>📡 Discovery</summary>

* [ ] Correct physical network interface is active
* [ ] No VPN is interfering
* [ ] No unexpected virtual adapter is selected
* [ ] Firewall permits discovery traffic
* [ ] UDP broadcast works
* [ ] TP-Link utility is running normally

</details>

<details>
<summary>🐧 Linux / Wine</summary>

* [ ] Wine is working
* [ ] TP-Link utility starts
* [ ] Bundled Java runtime is being used
* [ ] `-Djava.net.preferIPv4Stack=true` is supplied
* [ ] Option appears before `-jar`
* [ ] `tcpdump` shows the switch response
* [ ] Java is using IPv4

</details>

<details>
<summary>🪟 Windows</summary>

* [ ] Switch responds to ping
* [ ] Web management is reachable
* [ ] Correct Ethernet/Wi-Fi adapter is active
* [ ] VPN disabled for testing
* [ ] Unused virtual adapters disabled for testing
* [ ] Windows Firewall checked
* [ ] Third-party security software checked
* [ ] Utility restarted after adapter changes

</details>

---

# 📋 Useful Commands

<details>
<summary>🐧 Linux commands</summary>

### IP configuration

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

### Test HTTP

```bash
curl -I http://192.168.x.x/
```

### Monitor discovery

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

### Test switch

```cmd
ping 192.168.x.x
```

### Open Network Connections

```text
ncpa.cpl
```

</details>

---

# 💡 Why Not Disable IPv6?

A common troubleshooting suggestion is to disable IPv6 completely.

That is **not necessary** for this workaround.

The Java option:

```text
-Djava.net.preferIPv4Stack=true
```

only affects the Java application's socket behavior.

It does not globally disable IPv6.

This makes it a much more targeted solution.

---

# 📝 Reproduction Evidence

<details>
<summary>🔎 Reproduction details</summary>

The Linux/Wine issue was reproduced by observing:

1. The switch was reachable through HTTP.
2. The switch responded to UDP discovery.
3. Network capture showed the response arriving at the Linux host.
4. Java used an IPv6 socket with an IPv4-mapped address.
5. A normal IPv4 Java `DatagramSocket` could receive the response.
6. The TP-Link utility failed without the IPv4 option.
7. The utility discovered the switch immediately after adding:

```text
-Djava.net.preferIPv4Stack=true
```

This makes the workaround reproducible for the affected Linux/Wine configuration.

</details>

---

# 🧩 Compatibility

This repository focuses on:

* TP-Link Easy Smart Configuration Utility
* Easy Smart Switch discovery
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

---

# ⚖️ Legal / Clean-Room Note

This repository does **not** distribute modified TP-Link software.

It contains:

* troubleshooting information
* networking diagnostics
* configuration guidance
* a Java runtime startup option
* documentation of observed behavior

TP-Link software and trademarks remain the property of their respective owners.

No TP-Link executable or proprietary binary is distributed by this repository.

---

# 🤝 Contributing

Found another TP-Link switch or utility version with the same problem?

Pull requests and additional test results are welcome.

When opening an issue, please include:

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
Web management works:
UDP response visible:
```

<details>
<summary>🔒 Please remove sensitive information</summary>

Do not post:

* passwords
* credentials
* public IP addresses
* private keys
* sensitive network information
* unredacted packet captures

</details>

---

# ⭐ If This Helped

If this repository solved your discovery problem:

**⭐ Star the repository**

and consider opening an issue with your test results.

Additional reports can help establish which TP-Link utility versions, Java versions, Wine versions and switch revisions are affected.

---

<div align="center">

## TP-Link Easy Smart + Linux/Wine

### Reachable doesn't always mean discoverable.

```text
IPv4 UDP Broadcast
        ↓
Java Socket
        ↓
TP-Link Easy Smart Utility
        ↓
      Switch
```

**Made for easier troubleshooting.**

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
```
