# IPsec VPN on OPNsense with iOS, macOS

Getting IPsec VPN on OPNsense to work with iOS and macOS was too difficult.

## Versions

OPNsense 22.1.5  
iOS 15.3.1  
macOS Monterey (Version 12.3)

## Add group "VPN" [optional]

This step is optional, but it helps to ensure that only users in the VPN group can obtain VPN access.

System → Access → Groups → + (Add)  
Group name: VPN  
Description: VPN users

## Add one VPN user

System → Access → Users → + (Add)  
Username: vpnuser *(or any other name that you prefer)*  
Password: [password]

Optional (if you created the VPN group before)  
Group membership: make sure that "Member Of" contains "VPN"

## Create IPsec VPN, configure Mobile Clients

VPN → IPsec → Mobile Clients  
[unless otherwise noted, all checkboxes are off]  
✅ Enable IPsec Mobile Client Support  
Backend for authentication: Local Database  
(Optional, if you created a "VPN" group above): Enforce local group: VPN  
Virtual IPv4 Address Pool: ✅ Provide a virtual IPv4 address to clients  
Virtual IPv4 Address Pool: 192.168.2.1/24  
DNS Servers: ✅ Provide a DNS server list to clients  
DNS Servers: Server #1: 192.168.1.1 *(this needs to be the local IP of your OPNsense router)*  
DNS Servers: Server #2: 8.8.8.8 *(optional)*  
DNS Servers: Server #3: 8.8.4.4 *(optional)*  
Phase 2 PFS Group: off  
[all other checkboxes are off]

Save, Apply changes

## Create IPsec VPN, Create Phase 1

VPN → IPsec → Mobile Clients, press on "Create Phase1" at the top of the page, in the blue message bar.  
Connection method: default  
Key Exchange version: auto  
Internet Protocol: IPv4  
Interface: WAN  
Description: Mobile VPN *(or anything else descriptive)*

Phase 1 proposal (Authentication)  
Authentication method: Mutual PSK + Xauth  
My identifier: My IP address  
Pre-Shared Key: *(insert some randomly generate garbage, make sure you store it in a password manager)*

Phase 1 proposal (Algorithms)  
Encryption algorithm: AES  
Encryption algorithm: 256  
Hash algorithm: SHA256, SHA384, SHA512  
DH key group: 2, 5, 14, 15, 16, 17, 18  
Lifetime: 28800

Advanced options  
[all uncheck unless otherwise noted]  
Install policy: ✅  
NAT Traversal: Enable

Save, Apply changes

## Create IPsec VPN, Create Phase 2

VPN → IPsec → Tunnel Settings, in the newly created Phase 1 row, press on the + in the "Commands" column. (Do **NOT** press on the + below the row, because that creates a new Phase 1 row.)

Mode: Tunnel IPv4  
Description: *(something descriptive or leave empty)*  
Type: Network  
Address: 0.0.0.0 / 0 *(that is, set "0.0.0.0" in the Address field and choose 0 from the dropdown)*  
Protocol: ESP  
Encryption algorithms: AES128, AES192, AES256  
Hash algorithms: SHA256, SHA384, SHA512  
PFS key group: off  
Lifetime: 3600  
Automatically ping host: [leave empty]

Save, Apply changes

## Enable IPsec!

VPN → IPsec → Tunnel Clients

✅ Enable IPsec (near the bottom of the page)

Apply changes

## Firewall settings: NAT / WAN

Firewall → NAT → WAN

Add the following three rules:

Interface: WAN  
**TCP/IP Version: IPv4**  
**Protocol: ESP**  
Source: any  
Destination: WAN address  
Description: IPsec ESP

Interface: WAN  
**TCP/IP Version: IPv4**  
**Protocol: TCP/UDP**  
Source: any  
Destination: WAN address  
**Destination port range: from: ISAKMP, to: ISAKMP**  
Description: IPsec ISAKMP

Interface: WAN  
**TCP/IP Version: IPv4**  
**Protocol: TCP/UDP**  
Source: any  
Destination: WAN address  
**Destination port range: from: IPsec NAT-T, to: IPsec NAT-T**  
Description: IPsec NAT-T

The result should be:

 	IPv4 ESP      *   *   WAN address   *                      *     *     IPsec ESP
 	IPv4 TCP/UDP  *   *   WAN address   500  (ISAKMP)          *     *     IPsec ISAKMP
 	IPv4 TCP/UDP  *   *   WAN address   4500 (IPsec NAT-T)     *     *     IPsec NAT-T

Apply changes

## Firewall settings: NAT / IPsec

Firewall → NAT → IPsec

Add the following rule:

Interface: IPsec  
TCP/IP Version: IPv4  
Protocol: any  
Source: any  
Destination: any

The result should be:

```
IPv4 *     *     *     *     *     *     *
```

Apply changes

## Make DNS respond to queries from the VPN

Services → Unbound DNS → Access Lists

Add the following (using the + / Add button):

Access List name: 192.168.2.0/24 *(this is an open field, it does not matter)*  
Action: Allow  
Network: 192.168.2.0 / 24 *(this needs to be the range you chose under Mobile Clients)*  
Description: VPN DNS access

Save, Apply changes

## Optional: allow single user to create multiple VPN connections

Normally, a single user cannot create more than one simultaneous VPN connection (for example, using multiple devices). There is no UI option to fix this. It requires ssh access to the OPNsense server (via System → Settings → Administration → Enable Secure Shell)

ssh into OPNsense, create `/usr/local/etc/ipsec.opnsense.d/uniqueids-override.conf` and with the following contents:

```
config setup
  uniqueids = no
```

Then trigger a VPN restart through the UI.

## Ridiculous: A reboot may be necessary

Yes, this is ridiculous, but it did solve DNS problems for me and a number of other problems.

## Configuring IPsec VPN on iPhone

Settings → VPN → Add VPN Configuration...

Type: IPsec  
Description: [whatever you wish]  
Server: [public IP of OPNsense router]  
Account: vpnuser *(or the user you added above)*  
Password: [password of vpnuser]  
Use Certificate: [leave off]  
Group Name: [leave empty]  
Secret: [use the Pre-Shared Key configured above]

## Configuring IPsec VPN on macOS

System Preferences → Network → + (Add) button near the bottom left

Interface: VPN, VPN Type: Cisco IPSec  
Service Name: VPN (OPNsense) *(or whatever name you like)*  
Server Address: [public IP of OPNsense router]  
Account Name: vpnuser *(or the user you added above)*  
Password: [password of vpnuser]

Under Authentication Settings:  
Shared Secret: [use the Pre-Shared Key configured above]

## Sources (thank you!)

* OPNsense setup, albeit incomplete: https://docs.opnsense.org/manual/how-tos/ipsec-road.html
* Netgate documentation: https://docs.netgate.com/pfsense/en/latest/recipes/ipsec-mobile-ikev1-xauth.html
* A few forum discussions that helped
  * VPN Configuration: https://forum.opnsense.org/index.php?topic=10955.0
  * Debugging DNS: https://www.reddit.com/r/PFSENSE/comments/5y3qr1/pushing_dns_to_ipsec_clients/
  * No internet access for VPN clients: https://forum.opnsense.org/index.php?topic=11340.0
  * No internet access for VPN clients: https://forum.opnsense.org/index.php?topic=7477.0
  * And more: https://forum.opnsense.org/index.php?topic=14141.0
  * And even more: https://forum.opnsense.org/index.php?topic=14860.0;prev_next=prev#new
* Allowing same user to connect multpile times: https://forum.opnsense.org/index.php?topic=19462.0
