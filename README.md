<div align="center">

# TP-Link Easy Smart IPv4 Discovery Fix

### Fix switch discovery when the TP-Link Easy Smart Configuration Utility cannot find your switch

[![Platform](https://img.shields.io/badge/Linux-Wine-FCC624?style=for-the-badge&logo=linux&logoColor=black)](https://www.winehq.org/)
[![Windows](https://img.shields.io/badge/Windows-Supported-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://www.microsoft.com/windows)
[![Protocol](https://img.shields.io/badge/Discovery-UDP%20Broadcast-6f42c1?style=for-the-badge)](#how-discovery-works)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](#license)

**Your switch can be reachable by IP and still be invisible to the TP-Link configuration utility.**

This repository documents an IPv4 discovery workaround and troubleshooting procedure for the **TP-Link Easy Smart Configuration Utility**, with a particular focus on Linux/Wine environments.

</div>

---

## ⚡ TL;DR

If you are running the TP-Link Easy Smart Configuration Utility under **Linux/Wine** and your switch is not detected, try forcing the bundled Java runtime to use IPv4:

```text
-Djava.net.preferIPv4Stack=true
````

Example:

```bash
wine "$HOME/.wine/drive_c/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/jre/bin/javaw.exe" \
-Djava.net.preferIPv4Stack=true \
-Xmx300m \
-jar "C:/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/Easy Smart Configuration Utility.exe"
```

The important part is:

```text
-Djava.net.preferIPv4Stack=true
```

### Why?

The utility uses **UDP broadcast** for device discovery.

Under Wine, the bundled Java runtime can use an IPv6 socket with an IPv4-mapped address. In the affected configuration, the switch's response can reach the Linux host while Java/Wine does not deliver the IPv4 broadcast response to the application.

Forcing Java to use the IPv4 stack resolves this specific discovery problem.

---

# 📡 How Discovery Works

The Easy Smart Configuration Utility does **not** simply scan TCP port 80 to find switches.

It uses a separate UDP discovery mechanism.

A simplified representation is:

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
│     Ethernet / LAN   │
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

This means these two things are independent:

```text
HTTP management       → works
UDP discovery         → fails
```

So you can have:

```text
ping 192.168.x.x              ✅
http://192.168.x.x             ✅
TP-Link Utility discovery     ❌
```

and the switch itself can still be completely operational.

---

# 🐧 Linux / Wine

## Recommended Fix

Launch the TP-Link utility using:

```text
-Djava.net.preferIPv4Stack=true
```

### One-Command Launcher

```bash
mkdir -p ~/.local/bin && printf '%s\n' '#!/bin/bash' 'exec wine "$HOME/.wine/drive_c/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/jre/bin/javaw.exe" -Djava.net.preferIPv4Stack=true -Xmx300m -jar "C:/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/Easy Smart Configuration Utility.exe"' > ~/.local/bin/tplink-easy-smart && chmod +x ~/.local/bin/tplink-easy-smart
```

Then run:

```bash
~/.local/bin/tplink-easy-smart
```

You can also create a desktop launcher pointing to this script.

### What This Changes

The workaround only changes the Java application's socket preference.

It does **not**:

* disable IPv6 system-wide
* modify the switch
* modify the router
* change the switch IP
* change VLAN configuration
* modify the TP-Link executable
* require removing virtual network interfaces
* require disabling your firewall

---

# 🪟 Windows

Windows users should first determine whether the problem is actually network discovery rather than basic connectivity.

## 1. Test Connectivity

Open Command Prompt:

```cmd
ping 192.168.x.x
```

Replace `192.168.x.x` with your switch's IP address.

Then open:

```text
http://192.168.x.x
```

If the web interface works but the Easy Smart Configuration Utility does not detect the switch, continue below.

---

## 2. Check Network Adapters

Windows systems may have multiple network interfaces:

* Ethernet
* Wi-Fi
* VPN adapters
* Hyper-V
* VMware
* VirtualBox
* WSL
* Docker
* Tailscale
* ZeroTier
* Other virtual adapters

The TP-Link utility may select an unexpected interface for discovery.

Press:

```text
Win + R
```

and run:

```text
ncpa.cpl
```

Temporarily disable unused VPN or virtual adapters and leave the physical adapter connected to the switch enabled.

Restart the TP-Link utility and scan again.

---

## 3. Check Windows Firewall

The discovery process uses UDP broadcast.

Make sure Windows Firewall or third-party security software is not blocking the TP-Link utility.

Do **not** immediately disable the entire firewall.

Instead, check whether the TP-Link utility is permitted to communicate on your local/private network.

---

## 4. Java IPv4 Option

If your particular TP-Link utility installation allows Java startup parameters to be modified, you can test:

```text
-Djava.net.preferIPv4Stack=true
```

However, the IPv6-mapped IPv4 socket problem documented in this repository was specifically reproduced with the TP-Link application under **Wine**.

Therefore, Windows users should also investigate:

* network adapter selection
* firewall rules
* VPN software
* virtual adapters
* security software
* multiple active network interfaces

before assuming Java is the cause.

---

# 🔬 Technical Details

The TP-Link Easy Smart Configuration Utility is a Java-based application bundled with its own Java runtime.

The relevant networking behavior involves Java networking classes such as:

```text
java.net.DatagramSocket
java.net.DatagramPacket
java.net.InetAddress
java.net.NetworkInterface
java.net.InetSocketAddress
```

The discovery process uses UDP broadcast.

In the affected Wine configuration, Java can create an IPv6 socket and bind it to an IPv4-mapped address similar to:

```text
::ffff:192.168.x.x
```

The switch can still respond to the UDP broadcast.

The important distinction is:

```text
Network level:

PC ───── UDP discovery ─────> Switch
PC <──── UDP response ─────── Switch
                         ✅
```

while the application can experience:

```text
Application level:

Java/Wine receives response
                         ❌
```

Using:

```text
-Djava.net.preferIPv4Stack=true
```

causes Java to use an IPv4 socket instead.

Result:

```text
PC ───── IPv4 UDP broadcast ─────> Switch
PC <──── IPv4 UDP response ─────── Switch
                         ✅

Java receives response
                         ✅

Utility discovers switch
                         ✅
```

---

# 🧪 Verify Discovery Traffic on Linux

If the utility still cannot find your switch, monitor the network traffic.

First determine your LAN interface:

```bash
ip route
```

For example:

```text
default via 192.168.x.1 dev eno1
```

Then run:

```bash
sudo tcpdump -ni eno1 'udp port 29808 or udp port 29809'
```

Start a scan in the TP-Link utility.

A discovery exchange may look similar to:

```text
192.168.x.x.29809 > 255.255.255.255.29808
192.168.x.x.29808 > 255.255.255.255.29809
```

The exact packet contents are proprietary and are not required for basic troubleshooting.

### What the Results Mean

| tcpdump result                            | Likely situation                        |
| ----------------------------------------- | --------------------------------------- |
| No outgoing UDP packet                    | Utility/interface problem               |
| Outgoing request, no response             | Network/firewall/switch discovery issue |
| Request + switch response visible         | Network discovery is working            |
| Response visible but utility sees nothing | Application/socket handling issue       |

---

# 🌐 Multiple Network Interfaces

This is especially important on Linux.

Check:

```bash
ip addr
```

You may see interfaces such as:

```text
eno1
wlan0
docker0
lxcbr0
virbr0
tailscale0
```

A Java application may choose an interface you did not expect.

For example:

```text
                    ┌───────────────┐
                    │   Linux PC    │
                    └───────┬───────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
       eno1              lxcbr0            VPN/etc.
    192.168.x.x         10.x.x.x           virtual
          │
          │
          ▼
    TP-Link Switch
```

The physical LAN interface is the one that should be used for discovery.

### Important

Do **not** permanently delete or disable virtual interfaces just to make the TP-Link utility work.

Prefer an application-level IPv4 configuration over system-wide networking changes.

---

# 🔥 Firewall Considerations

Discovery uses UDP broadcast.

Therefore, a firewall can allow:

```text
TCP/80
ICMP/ping
```

while blocking:

```text
UDP broadcast
```

If HTTP works but discovery does not, check firewall rules.

On Linux with UFW:

```bash
sudo ufw status
```

For systems using nftables or firewalld, inspect the appropriate rules.

Avoid disabling the entire firewall as a first troubleshooting step.

---

# 🛠 Troubleshooting Checklist

## Basic Connectivity

* [ ] Switch is powered on
* [ ] PC and switch are on the same LAN/VLAN
* [ ] Switch has a valid IP address
* [ ] PC can ping the switch
* [ ] Switch web interface opens

## Discovery

* [ ] TP-Link Easy Smart Configuration Utility is installed
* [ ] Correct physical network interface is active
* [ ] No VPN is interfering
* [ ] No virtual adapter is being selected accidentally
* [ ] Firewall permits the discovery traffic
* [ ] UDP broadcast is working

## Linux / Wine

* [ ] Utility runs correctly under Wine
* [ ] Bundled Java runtime is being used
* [ ] `-Djava.net.preferIPv4Stack=true` is supplied before `-jar`
* [ ] tcpdump shows the discovery response
* [ ] Java is no longer using the problematic IPv6-mapped socket

## Windows

* [ ] Switch responds to ping
* [ ] Switch web interface is reachable
* [ ] Correct Ethernet/Wi-Fi adapter is active
* [ ] VPN is disabled for testing
* [ ] Unused virtual adapters are disabled for testing
* [ ] Windows Firewall permits the application
* [ ] Third-party security software is not blocking discovery

---

# 📋 Quick Diagnostic Commands

## Linux

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

## Windows

### Test switch

```cmd
ping 192.168.x.x
```

### Open Network Connections

```text
ncpa.cpl
```

---

# 💡 Why `preferIPv4Stack` Instead of Disabling IPv6?

A common troubleshooting suggestion is to disable IPv6 system-wide.

That is unnecessary for this problem.

The Java option:

```text
-Djava.net.preferIPv4Stack=true
```

only affects the Java application's networking behavior.

It does **not** globally disable IPv6.

This makes it a targeted workaround rather than a system-wide networking modification.

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

The exact behavior can vary depending on:

* TP-Link utility version
* switch model
* switch hardware revision
* switch firmware
* Java version
* Wine version
* Linux distribution
* Windows version
* firewall configuration
* network topology
* VLAN configuration
* VPN/virtual adapters

The workaround should therefore be considered a targeted fix for the affected configuration rather than a universal requirement for every TP-Link switch.

---

# 📝 Evidence / Reproduction

The Linux/Wine issue documented here was reproduced by observing all of the following:

1. The switch was reachable through its HTTP management interface.
2. The switch responded to UDP discovery packets.
3. Network capture showed the switch's UDP response arriving at the Linux host.
4. The Java application used an IPv6 socket with an IPv4-mapped address.
5. A normal IPv4 Java `DatagramSocket` could receive the response.
6. The TP-Link utility failed to discover the switch without the IPv4 option.
7. Launching the same utility with:

```text
-Djava.net.preferIPv4Stack=true
```

caused the switch to be discovered immediately.

This provides a reproducible explanation for the Linux/Wine case rather than simply assuming that IPv4 is required.

---

# ⚖️ Legal / Clean-Room Note

This repository does not distribute modified TP-Link software.

It provides:

* troubleshooting information
* configuration guidance
* networking diagnostics
* a Java runtime startup option
* documentation of observed network behavior

TP-Link software, trademarks and proprietary implementations remain the property of their respective owners.

No TP-Link executable or proprietary binary is included in this repository.

---

# 🤝 Contributing

Found another TP-Link switch or utility version with the same problem?

Contributions and additional test results are welcome.

When reporting an issue, please include:

```text
Operating system:
Distribution / Windows version:
Wine version:
Java version:
TP-Link Utility version:
Switch model:
Hardware revision:
Firmware:
Network interface:
VPN / virtual adapters:
Does ping work?:
Does web management work?:
Does tcpdump show the UDP response?:
```

Please do not post:

* passwords
* private credentials
* public IP addresses
* sensitive network information
* complete packet captures containing sensitive data

---

# ⭐ If This Helped

If this fixed your TP-Link switch discovery problem:

**⭐ Star the repository**

and consider opening an issue with your:

* switch model
* utility version
* operating system
* results

Additional reports can help determine how widespread the discovery/interface issue is.

---

<div align="center">

### TP-Link Easy Smart Switches + Linux/Wine

**Reachable doesn't always mean discoverable.**

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

```

**Note:** I deliberately kept the Windows section cautious: the **IPv4-mapped socket failure was actually reproduced under Wine**, while for Windows we can document the same JVM option as something to test rather than claiming the identical bug is proven there.
```
