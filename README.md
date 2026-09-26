# TP-Link Easy Smart IPv4 Discovery Fix

Fix and troubleshooting guide for TP-Link Easy Smart Switches that are not discovered by the Easy Smart Configuration Utility.

## The problem

The TP-Link Easy Smart Configuration Utility uses UDP broadcast for switch discovery. A switch can be fully reachable through its web interface and still remain invisible to the utility.

On Linux/Wine, the bundled Java runtime can use an IPv6 socket with an IPv4-mapped address. In this situation the switch response can reach the computer while Java/Wine fails to deliver the IPv4 broadcast response to the application.

## Linux / Wine fix

Start the TP-Link utility with:

`-Djava.net.preferIPv4Stack=true`

Example:

```bash
wine "$HOME/.wine/drive_c/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/jre/bin/javaw.exe" -Djava.net.preferIPv4Stack=true -Xmx300m -jar "C:/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/Easy Smart Configuration Utility.exe"
```

The option must be specified before `-jar`.

### Permanent launcher

```bash
mkdir -p ~/.local/bin && printf '%s\n' '#!/bin/bash' 'exec wine "$HOME/.wine/drive_c/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/jre/bin/javaw.exe" -Djava.net.preferIPv4Stack=true -Xmx300m -jar "C:/Program Files (x86)/TPLINK/EasySmartConfigurationUtility/Easy Smart Configuration Utility.exe"' > ~/.local/bin/tplink-easy-smart && chmod +x ~/.local/bin/tplink-easy-smart
```

Then run:

`~/.local/bin/tplink-easy-smart`

## Windows

Windows users should first verify that the switch responds to ping and that its web interface is reachable. If discovery still fails, check for multiple network adapters such as VPN, Hyper-V, VMware, VirtualBox, WSL, Docker, Tailscale or other virtual interfaces.

If the Java runtime allows custom JVM parameters, `-Djava.net.preferIPv4Stack=true` can also be tested. The Linux/Wine IPv6-mapped IPv4 socket issue documented here was specifically reproduced under Wine, so Windows users should also investigate adapter selection and firewall/security software.

## Why HTTP can work while discovery fails

The web interface and discovery protocol are separate:

```text
PC  ---- HTTP/80 ----------------> Switch   OK
PC  <--- HTTP/80 ----------------- Switch   OK

PC  ---- UDP broadcast ----------> Switch
PC  <--- UDP discovery response -- Switch   MAY FAIL
```

The discovery process therefore cannot be diagnosed solely by opening the switch web interface.

## Linux diagnostic

Monitor discovery traffic with:

```bash
sudo tcpdump -ni eno1 'udp port 29808 or udp port 29809'
```

Replace `eno1` with the network interface connected to the switch.

A working exchange looks approximately like:

```text
192.168.x.x.29809 > 255.255.255.255.29808
192.168.x.x.29808 > 255.255.255.255.29809
```

If the switch response is visible in tcpdump but the application does not discover the switch, the network path itself is working and the problem may be in the application socket handling.

## Important

This workaround does not disable IPv6 system-wide, modify the switch, change router settings, or modify the TP-Link executable. It only tells the Java runtime used by the application to prefer the IPv4 networking stack.

## Scope

Tested with the TP-Link Easy Smart Configuration Utility running under Wine with the bundled Java runtime. The exact underlying behavior may differ between operating systems, Java versions and Wine versions.

## Contributing

Pull requests and additional test results are welcome. Please include your operating system, Wine/Java version, TP-Link utility version, switch model/hardware revision and network topology when reporting results.

## License

Documentation in this repository is released under the MIT License. TP-Link software, trademarks and proprietary protocols remain the property of their respective owners.
