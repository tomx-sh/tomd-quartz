---
publish: true
created: 2026-07-31
modified: 2026-08-07T09:59:08.191Z
---

# Run a WireGuard VPN server on Ubuntu 26.04

WireGuard provides an encrypted route into a home network without exposing individual services directly to the internet. This guide configures an Ubuntu home-lab machine as the VPN server. Laptops and phones will connect as WireGuard clients and can then reach services on the trusted LAN.

The example server is an HP EliteDesk running Ubuntu 26.04. Its network details are:

- Ethernet interface: `eno1`
- Reserved LAN address: `192.168.1.250`
- WireGuard interface: `wg0`
- VPN subnet: `10.8.0.0/24`
- Server VPN address: `10.8.0.1`
- UDP listen port: `51820`

Replace interface names and addresses where they differ on another network.

## Prerequisites

This setup assumes:

- Ubuntu has a working wired network connection
- SSH access to the server
- administrator access to the home router
- the router can forward an inbound UDP port

Do not expose SSH, RDP, dashboards, or other home-lab services individually. WireGuard should be the single remote-access entry point.

## 1. Install WireGuard

Update the package index and install WireGuard:

```sh
sudo apt update
sudo apt install wireguard
```

Verify the installed tools:

```sh
wg --version
```

This machine reports `wireguard-tools v1.0.20250521`.

## 2. Identify the LAN interface

List the available interfaces and addresses:

```sh
ip -brief address
```

The EliteDesk uses `eno1`. Its initial DHCP address was `192.168.1.3/24`:

```text
eno1    UP    192.168.1.3/24
```

The interface name is required by the NAT rule in the WireGuard configuration.

## 3. Generate the server keys

Generate a private key and derive its public key:

```sh
sudo sh -c 'umask 077; wg genkey | tee /etc/wireguard/server-private.key | wg pubkey > /etc/wireguard/server-public.key'
```

The restrictive `umask` prevents other users from reading either key file. Never publish or share `server-private.key`. The public key is safe to copy into client configurations.

## 4. Enable IPv4 forwarding

The server must forward packets between the WireGuard interface and the LAN interface. Make the kernel setting persistent:

```sh
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system
```

Confirm that the output includes:

```text
net.ipv4.ip_forward = 1
```

## 5. Configure the WireGuard interface

Create the configuration:

```sh
sudo nano /etc/wireguard/wg0.conf
```

Add:

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -D POSTROUTING -o eno1 -j MASQUERADE
```

Replace the placeholder without printing the private key:

```sh
sudo sh -c 'key=$(cat /etc/wireguard/server-private.key); sed -i "s|SERVER_PRIVATE_KEY|$key|" /etc/wireguard/wg0.conf; chmod 600 /etc/wireguard/wg0.conf'
```

The `PostUp` rules allow VPN traffic to be forwarded and translated through `eno1`. `PostDown` removes the same rules cleanly when the interface stops.

## 6. Start WireGuard

Start the interface:

```sh
sudo wg-quick up wg0
```

Enable it automatically at boot:

```sh
sudo systemctl enable wg-quick@wg0
```

Verify the running interface:

```sh
sudo wg show
```

The output should show `wg0`, its public key, and UDP port `51820`. The private key must appear as `(hidden)`.

## 7. Reserve the server's LAN address

A port-forwarding rule must always point to the same server address. The simplest way to achieve this is to create a DHCP reservation on the router: the server continues to use DHCP, but the router always assigns the same address to its Ethernet MAC address.

Open the router's administration interface and find its DHCP, LAN, address-reservation, or static-lease settings. Create an IPv4 reservation that maps the server's wired MAC address to an unused address on the LAN. Use an address permitted by the router's DHCP configuration and avoid one already assigned to another device.

Confirm the server's Ethernet MAC address with:

```sh
cat /sys/class/net/eno1/address
```

This EliteDesk uses `f8:b4:6a:b0:56:79`. Its reserved address for this guide is `192.168.1.250`.

> [!example] SFR GR140CG
>
> 1. Open `http://192.168.1.1` and sign in.
> 2. Go to **LAN → Baux statiques**.
> 3. Under IPv4, select **Ajouter bail statique**.
> 4. Select bridge `brlan0`.
> 5. Select the server's Ethernet MAC address.
> 6. Assign an unused address and create the reservation.
>
> This router would not reserve the EliteDesk's active dynamic address, `192.168.1.3`, because it considered that address already in use. After checking that it was available, `192.168.1.250` was reserved instead.

Renew the DHCP lease or reboot the server so it obtains the reserved address. Rebooting is the most reliable option when working remotely:

```sh
sudo reboot
```

Reconnect after startup:

```sh
ssh tom@192.168.1.250
```

Confirm that DHCP assigned the reservation:

```sh
ip -brief address show eno1
```

The expected address is `192.168.1.250/24`. Also run `sudo wg show` after the reboot to confirm that `wg-quick@wg0` started automatically and is still listening on UDP port `51820`.

## 8. Forward the WireGuard port

Open the router's NAT, firewall, or port-forwarding settings and create an inbound rule with:

- Name or service: `WireGuard`
- Destination device or address: `192.168.1.250`
- Protocol: UDP only
- External port: `51820`
- Internal port: `51820`
- Source: any address
- Status: enabled

If the interface asks for port ranges, enter `51820` as both the beginning and end. Select the WAN or internet-facing interface if the router asks which interface should receive the traffic. Use port forwarding, not port triggering, and do not create a TCP rule.

> [!example] SFR GR140CG
> Open **Sécurité → Accès → Réseau v4 → Redirection de ports**, then select **Créer une règle**. Do not use the similarly named **Déclenchement de ports** section.
>
> Configure:
>
> - Interface: `erouter0`
> - Nom du service: `WireGuard`
> - Adresse IP du serveur: `192.168.1.250`
> - Protocole: UDP
> - Ports externes, début and fin: `51820`
> - Ports internes, début and fin: `51820`
> - Activer la règle: On
>
> Select **Créer**, then confirm that the enabled `WireGuard` entry appears in the port-forwarding table.

Port forwarding requires a publicly reachable IPv4 address. If the router does not offer port forwarding, its WAN address differs from the public address reported by an external service, or an external test never reaches WireGuard, check whether the internet connection uses carrier-grade NAT. With CGNAT, request a public IPv4 address from the ISP or use an alternative design such as a small public VPS acting as the WireGuard rendezvous point.

## 9. Configure dynamic DNS

Most residential internet connections use a public IP address that can change. A WireGuard client configured with that numeric address will stop connecting after the ISP assigns a new one. Unless the connection includes a static public address, configure dynamic DNS before relying on the VPN remotely.

Dynamic DNS provides a stable hostname such as `vpn.example.com`. An updater on the router or server detects the current public IP address and keeps the hostname's DNS record synchronized with it.

The general procedure is:

1. Obtain a hostname from a dynamic-DNS provider or create one under a domain you own.
2. Create restricted update credentials for that hostname. Never store the provider account's main password in the router.
3. Configure the router's DynDNS client, or an updater on the server, with the hostname and restricted credentials.
4. Verify that the hostname resolves to the current public address.
5. Use the hostname, not a numeric public IP, in every WireGuard client endpoint.

Check the current public IPv4 address with:

```sh
curl -4 https://api.ipify.org; echo
```

After configuring dynamic DNS, compare it with:

```sh
dig +short vpn.example.com
```

Both commands should return the same address. DNS updates can take a few minutes to propagate. A WireGuard client may need to be deactivated and reactivated after an address change so it resolves the hostname again.

> [!example] OVH DynHost with an SFR GR140CG
> In the OVH DNS zone, open **DynHost**. Create a dedicated DynHost access restricted to subdomain `vpn`, then add `vpn.example.com` as a DynHost record using the current public IPv4 address.
>
> In the SFR router's **DynDNS** settings, select `ovh.fr` and enter:
>
> - Hostname: `vpn.example.com`
> - Username: the complete OVH DynHost identifier
> - Password/key: the dedicated DynHost password
>
> Enable DynDNS and save. Do not enter the main OVH account credentials.

## 10. Add the first client

Each client should receive its own key pair and unique VPN address, starting with `10.8.0.2/32`. Never reuse a private key between devices.

In the macOS WireGuard app, select **Add Empty Tunnel** and give the tunnel a descriptive name. The app generates the client key pair locally. Copy only the displayed public key; the private key must remain in the Mac configuration.

Add the Mac to `/etc/wireguard/wg0.conf` on the server:

```ini
[Peer]
# Mac
PublicKey = MAC_PUBLIC_KEY
AllowedIPs = 10.8.0.2/32
```

Reload the interface:

```sh
sudo systemctl restart wg-quick@wg0
```

Complete the Mac tunnel while retaining the private key generated by the app:

```ini
[Interface]
PrivateKey = MAC_PRIVATE_KEY
Address = 10.8.0.2/32

[Peer]
PublicKey = SERVER_PUBLIC_KEY
AllowedIPs = 10.8.0.0/24, 192.168.1.0/24
Endpoint = vpn.example.com:51820
PersistentKeepalive = 25
```

This is a split-tunnel configuration: only the WireGuard subnet and home LAN use the VPN. Normal internet traffic continues through the Mac's current connection.

## 11. Verify remote access

Connect the Mac to a phone hotspot over mobile data before activating the tunnel. This ensures the test actually enters through the router's public interface rather than remaining on the home LAN.

Activate the tunnel and test the server's VPN address:

```sh
ping -c 3 10.8.0.1
```

Test routing to the server's LAN address:

```sh
ping -c 3 192.168.1.250
```

Finally, connect through the VPN address:

```sh
ssh tom@10.8.0.1
```

All three tests succeeded from the Mac over a phone's mobile-data connection, confirming the WireGuard handshake, router port forwarding, VPN routing, and SSH access.

The short hostname `elitedesk` is normally supplied by the SFR router's local DNS and does not resolve automatically over WireGuard. Using `10.8.0.1` keeps the VPN configuration simple. Local DNS can be added later if hostname resolution becomes useful.

Useful server-side diagnostics include:

```sh
sudo wg show
ip -brief address show wg0
sudo systemctl status wg-quick@wg0
```
