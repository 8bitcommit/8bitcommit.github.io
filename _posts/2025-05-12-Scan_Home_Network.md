---
layout: post
title: "Who Built the Machines That Live in Our Homes?"
date: 2025-05-12
author: Adam Kearsey
description: "Map and identify every device on your home network using a stealth Python script. Learn who built them and what they’re doing."
tags: [networking, python, security]
---



![smartdevices](/assets/img/smart_devices.jpg)
Photo by <a href="https://unsplash.com/@jakubzerdzicki?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Jakub Żerdzicki</a> on <a href="https://unsplash.com/photos/a-computer-keyboard-light-bulbs-and-other-electronics-on-a-purple-and-yellow-background-3Ik3JMp7Woo?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>
      
> *Who developed our IoT devices?*

Most of us live with dozens of connected devices. Smart speakers, TVs, routers, security cameras, printers, all quietly talking across our networks. But **do you know who built them**? Or where the firmware came from?

This post walks through a Python-based script that can scan your local network and identify each device by:
- IP address
- MAC address: The first 3 bytes of a MAC address (the OUI) map to a vendor.
- Vendor name (like Apple, TP-Link, or Espressif)
- Open ports (to understand what services they expose)

---
##  Why This Matters

Your home network is a quiet showcase of who you’ve trusted, **knowingly or not.**

- That baby monitor might phone home to a server on the other side of the world.
- That smart plug might be running firmware written by a company you've never heard of.
- That "router" might also be a VPN endpoint, file server, or DNS proxy.

Knowing **who built what**, and **what it’s doing**, is a first step toward taking control of your digital home.


---

## Requirements

Install the necessary packages on Linux or MacOS:

```bash
pip3 install scapy mac-vendor-lookup
```
---

## Finding Your Local IP and Subnet

Before scanning, you need to know your network range:  
usually something like `192.168.1.0/24`.

### On Linux/macOS:
Open a terminal and run:

 ```bash 
 ip a
 ```


Look for a line like:

```bash
inet 192.168.1.42/24
```
That means your subnet is 192.168.1.0/24.   
Replace the 'Local_Subnet' variable to match this.


## The Code
This script uses ARP to silently discover devices, probes a set of common ports, and identifies the manufacturer via MAC prefix. View [8BitCommit GitHub Repository](https://github.com/8bitcommit/Home_Network) for more info.

 ```python
    from scapy.all import ARP, Ether, srp
    from mac_vendor_lookup import MacLookup
    import socket
    import time
    import csv

    Local_Subnet = '192.168.1.0/24'
    COMMON_PORTS = [
        21, 22, 23, 53, 80, 123, 137, 138, 139, 445,
        443, 3306, 3389, 5357, 8000, 8080, 8443
    ]
    
    def quietly_probe_ports(ip, ports):
        open_ports = []
        for port in ports:
            try:
                sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                sock.settimeout(0.5)
                result = sock.connect_ex((ip, port))
                if result == 0:
                    open_ports.append(port)
                sock.close()
                time.sleep(0.3)
            except:
                pass
        return open_ports

    def quiet_arp_scan(subnet = Local_Subnet, output_file='network_inventory.csv'):
        arp = ARP(pdst=subnet)
        ether = Ether(dst="ff:ff:ff:ff:ff:ff")
        packet = ether / arp
        result = srp(packet, timeout=2, verbose=0)[0]

        mac_lookup = MacLookup()
        try:
            mac_lookup.update_vendors()
        except:
            pass

        inventory = []

        for _, received in result:
            ip = received.psrc
            mac = received.hwsrc
            vendor = 'Unknown'
            try:
                vendor = mac_lookup.lookup(mac)
            except:
                pass

            open_ports = quietly_probe_ports(ip, COMMON_PORTS)
            ports_str = ', '.join(map(str, open_ports)) if open_ports else 'None'

            print(f"\nDevice Found:")
            print(f"  IP: {ip}")
            print(f"  MAC: {mac}")
            print(f"  Vendor: {vendor}")
            print(f"  Open Ports: {ports_str}")

            inventory.append({
                'IP': ip,
                'MAC': mac,
                'Vendor': vendor,
                'Open Ports': ports_str
            })

        with open(output_file, mode='w', newline='') as f:
            writer = csv.DictWriter(f, fieldnames=['IP', 'MAC', 'Vendor', 'Open Ports'])
            writer.writeheader()
            writer.writerows(inventory)

        print(f"\nInventory saved to {output_file}")

    if __name__ == '__main__':
        quiet_arp_scan()

```
---

## Running It

```bash
sudo python3 scan.py
```

---

## Sample Output

```csv
IP,MAC,Vendor,Open Ports
192.168.1.1,44:65:0d:ef:23:aa,TP-Link Technologies,80, 443
192.168.1.42,fc:99:47:99:a7:1c,Apple Inc.,None
```
Heres a sample redacted result:
![Network Inventory Screenshot](/assets/img/Network_Inventory.png)

## Investigating Open Ports and Developers


You can check if your devices are exposed to the internet using a service like [CanYouSeeMe.org](https://canyouseeme.org). It will tell you if specific ports on your public IP are reachable from the outside world.

Once you’ve identified a device vendor (from its MAC address or hostname), search for more context:
    
    CompanyName + firmware

    CompanyName + exploit

    CompanyName + backdoor
    

You’ll often uncover:

- **OEM/ODM relationships** -- companies like Tuya manufacture smart plugs and white-label them for dozens of "brands."
- **3rd-party firmware vendors** -- names like Espressif, Realtek, or Amlogic often provide the chipsets and embedded software that run the show behind the scenes.

This gives you a clearer picture of not just *what* is on your network, but *who’s really behind it*.

---
## Conclusion

Most of us invited these machines in without much scrutiny. They showed up in Amazon boxes, in router default configs, in light bulbs that suddenly needed apps.

But they're here now, running firmware from companies we've never heard of, making connections we didn't authorize, and sitting silently on our networks.

This script doesn’t solve all that. But it gives you a map.

It shows you what’s here, who made it, and what it might be doing. That’s a good place to begin.

*– Adam Kearsey*

>*Curious what to do with this info next? I’m working on a follow-up:  **“Hardening the Home Network”** — keep an eye out.*

