---
title: "Network and VPN Config for Kubernetes Home cluster"
date: 2026-10-25
lastmod: 2026-10-25
description: "Create an isolated network for a Kubernetes home cluster and set up VPN"
tags: ["kubernetes", "Home Cluster"]
categories: ["Networking"]
series: ["Kubernetes Home Cluster"]
series_order: 1
draft: false

# --- Hero image ---
# Drop any image named feature.jpg / feature.png / feature.webp into this
# directory. It becomes the card thumbnail, the hero banner, AND the
# Open Graph image shared on Twitter/LinkedIn/Slack automatically.
showHero: true
heroStyle: "big"          # basic | big | background | thumbAndBackground

# --- Reading experience ---
showTableOfContents: true
showReadingTime: true
showDateUpdated: true
showTaxonomies: true
showPagination: true

# --- dev.to cross-posting ---
# Publish here first, get your canonical URL, then paste it into the
# dev.to editor under "Advanced" > "Canonical URL". That's all you need.
# Google will index YOUR site as the original source.
---

This is the first in a series of posts on building a K3s Kubernetes home cluster from three old Lenovo ThinkCentre M910q mini PCs and a [MikroTik hEX S](https://mikrotik.com/product/hex_s) router. This post configures the router and WireGuard to achieve two main things:
1) Create a network that is isolated from my home LAN for the Kubernetes hosts
2) Manage the router and hosts from my laptop securely without a cable

### Network design
The hEX sits on my home LAN like any other device, with its WAN port (ether1) at `192.168.1.7`. Behind it, the router runs a separate cluster LAN on `10.20.0.0/24`. The only way in from the home LAN is the WireGuard port, and only my laptop holds a key.

{{< mermaid >}}
flowchart LR
internet((Internet)) --- isp

subgraph home["Home LAN 192.168.1.0/24"]
isp["ISP router"]
laptop["Laptop<br/>Wi-Fi 192.168.1.x<br/>tunnel 10.20.100.2"]
other["Other home devices"]
end

subgraph hex["MikroTik hEX S"]
ether1["ether1 (WAN)<br/>192.168.1.7"]
wg0["wg0<br/>10.20.100.1"]
bridge["bridge (LAN)<br/>10.20.0.1"]
end

subgraph cluster["K8s Cluster LAN 10.20.0.0/24"]
controlplane["controlplane<br/>10.20.0.11"]
node1["node1<br/>10.20.0.12"]
node2["node2<br/>10.20.0.13"]
end

isp --- ether1
laptop == "WireGuard UDP 51820" ==> ether1
other -- "dropped" --x ether1
ether1 -. "decrypted" .-> wg0
wg0 -- "routed" --> bridge
bridge --- controlplane
bridge --- node1
bridge --- node2
{{< /mermaid >}}

The tunnel gets its own subnet, `10.20.100.0/24`. The router's end (`wg0`) is `.1` and the laptop is `.2`. Because VPN clients are not on `10.20.0.0/24`, firewall and service rules can tell tunnel traffic apart from hosts on the cluster LAN. Once the router decrypts a packet on `wg0`, it routes it to the cluster LAN as it would between any two directly connected networks.

### Why 10.20.0.0/24

Every subnet in the diagram below must stay distinct. If two overlap, the laptop or the nodes cannot tell which network an address belongs to.

{{< mermaid >}}
flowchart TB
subgraph r192["192.168.0.0/16"]
homelan["192.168.1.0/24<br/>Home LAN (WAN side)"]
mtdefault["192.168.88.0/24<br/>MikroTik default"]
end

subgraph r10["10.0.0.0/8"]
clan["10.20.0.0/24<br/>Cluster LAN"]
tun["10.20.100.0/24<br/>WireGuard tunnel"]
pods["10.42.0.0/16<br/>K3s pods"]
svcs["10.43.0.0/16<br/>K3s services"]
end

classDef mine fill:#d4f4dd,stroke:#2e7d32,color:#000
classDef avoid fill:#fde2e1,stroke:#c62828,color:#000
class clan,tun mine
class homelan,mtdefault,pods,svcs avoid
{{< /mermaid >}}

- **Home LAN** (`192.168.1.0/24`): the hEX's WAN side. Reusing it on the LAN side would break routing.
- **MikroTik default** (`192.168.88.0/24`): every hEX ships with it, so another MikroTik on a network I visit could clash.
- **K3s pods and services** (`10.42.0.0/16`, `10.43.0.0/16`): K3s uses these inside the cluster. Node addresses must not overlap them.
- **Remote networks**: with a split tunnel, the laptop routes only `10.20.0.0/24` and `10.20.100.0/24` through WireGuard, so a remote network clashes only if it also uses those ranges.

### Address plan

The cluster LAN splits into fixed blocks. Static addresses sit below the DHCP pool and a block above it is reserved for MetalLB, so DHCP never hands out an address a node or service already uses.

{{< mermaid >}}
flowchart LR
a[".1<br/>Router"] --> b[".10<br/>VIP<br/>(Future HA)"] --> c[".11 - .13<br/>K3s nodes"] --> d[".100 - .199<br/>DHCP pool"] --> e[".200 - .220<br/>Possible MetalLB pool"]

classDef static fill:#dbe9ff,stroke:#1565c0,color:#000
classDef dynamic fill:#fff4d6,stroke:#ef6c00,color:#000
classDef lb fill:#ead9f7,stroke:#6a1b9a,color:#000
class a,b,c static
class d dynamic
class e lb
{{< /mermaid >}}

| Address | Use |
|---|---|
| 10.20.0.1 | Router |
| 10.20.0.10 | VIP (future HA) |
| 10.20.0.11-13 | K3s nodes |
| 10.20.0.100-199 | DHCP pool |
| 10.20.0.200-220 | Possible MetalLB pool |
| 10.20.100.1 | Router tunnel end (wg0) |
| 10.20.100.2 | Laptop tunnel end |

## Starting point

I had already upgraded RouterOS to the latest v7 (WireGuard needs v7), set an admin password and plugged my laptop into ether2.

The WebFig Terminal doesn't accept pasted text, so I used SSH from my laptop instead. Log in with the password first, then add your public key. The two options stop SSH offering your local keys, so it goes straight to the password prompt.

```
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no admin@192.168.88.1
```

### SSH key access

RouterOS has no `ssh-copy-id`. Add the key from the router instead:

```
/user ssh-keys add user=admin key="ssh-ed25519 AAAA... you@example"
```

Paste the full contents of your `~/.ssh/id_ed25519.pub` between the quotes.

Once a user has a key, RouterOS stops accepting password logins over SSH for that user. Keep the password session open and test key login from a second terminal:

```
ssh -i ~/.ssh/id_ed25519 admin@192.168.88.1
```

## Move the LAN to 10.20.0.0/24

The order matters: add the new address, move DHCP, reconnect, then remove the old address. The router keeps answering on at least one address the whole time, so you can't lock yourself out.

### Add the new address alongside the old one

```
/ip address add address=10.20.0.1/24 interface=bridge comment=lan
/ip address print
```

The router now answers on both `192.168.88.1` and `10.20.0.1`, so the current session stays up. If a later step fails, the old address is still there.

### Point DHCP and DNS at the new network

```
/ip pool set [find name=default-dhcp] ranges=10.20.0.100-10.20.0.199
/ip dhcp-server network set [find comment=defconf] address=10.20.0.0/24 gateway=10.20.0.1 dns-server=10.20.0.1
/ip dns static set [find name=router.lan] address=10.20.0.1
```

- `ip pool set`: moves the DHCP range to `.100-.199`, clear of the router, VIP, nodes and MetalLB block.
- `dhcp-server network set`: tells DHCP clients the new subnet, and that `10.20.0.1` is their gateway and DNS server.
- `dns static set`: points the `router.lan` name at the new address.

A `set` on a `find` that matches nothing does nothing and reports no error, so print each one to confirm it worked:

```
/ip pool print
/ip dhcp-server network print
/ip dns static print
```

### Get a new lease on the laptop

The laptop still holds its old `192.168.88.x` lease. Unplug and replug the cable to get a `10.20.0.x` address. The SSH or WebFig session drops at this point.

```
ip addr
ping -c 3 10.20.0.1
ssh admin@10.20.0.1
```

`ip addr` should show a `10.20.0.1xx` address on the wired interface. The ping and SSH confirm the router answers on the new address.

### Remove the old address

Only do this once the laptop reaches `10.20.0.1`.

```
/ip address remove [find address="192.168.88.1/24"]
/ip address print
/ip dhcp-server network print
/ip pool print
```

Only `10.20.0.1/24` should remain on the bridge. WebFig is now at `http://10.20.0.1`.

## WireGuard on the router

A WireGuard interface is a virtual network interface. Packets routed into it leave the router encrypted, as UDP to the peer. Each end has a key pair: the private key stays on the device, and the other end holds the public key. Each side accepts packets only from keys it knows.

First, give the hEX a DHCP reservation on the ISP router so ether1 always gets `192.168.1.7`. The laptop config points at that address. Check it:

```
/ip address print where interface=ether1
```

Create the interface and give it the router's tunnel address:

```
/interface wireguard add name=wg0 listen-port=51820 comment=wg
/ip address add address=10.20.100.1/24 interface=wg0 comment=wg
```

- `interface wireguard add`: creates `wg0`, listening on UDP 51820. RouterOS v7 generates the key pair.
- `ip address add`: puts the router at `10.20.100.1` on the tunnel subnet. The `/24` tells the router that all of `10.20.100.0/24` is reached through `wg0`.

Show the router's public key and copy it for the laptop config:

```
/interface wireguard print
```

## WireGuard on the laptop

### Install tools and generate a key pair

```
sudo apt install -y wireguard
sudo sh -c 'umask 077; wg genkey > /etc/wireguard/laptop.key'
sudo sh -c 'wg pubkey < /etc/wireguard/laptop.key > /etc/wireguard/laptop.pub'
sudo cat /etc/wireguard/laptop.pub
```

- `apt install wireguard`: installs `wg` and `wg-quick`.
- `umask 077; wg genkey`: generates a private key in a file only root can read. The `umask` applies before the file is created, so other users can never read the key.
- `wg pubkey`: derives the public key from the private key. The public key is safe to share; the router needs it in the next section.

### Write the config

Set `ROUTER_PUB` to the key from `/interface wireguard print`.

```
ROUTER_PUB="ROUTER_PUBLIC_KEY"
LAPTOP_KEY=$(sudo cat /etc/wireguard/laptop.key)
sudo tee /etc/wireguard/wg-home.conf > /dev/null <<CONF
[Interface]
PrivateKey = $LAPTOP_KEY
Address = 10.20.100.2/32

[Peer]
PublicKey = $ROUTER_PUB
Endpoint = 192.168.1.7:51820
AllowedIPs = 10.20.0.0/24, 10.20.100.0/24
PersistentKeepalive = 25
CONF
sudo chmod 600 /etc/wireguard/wg-home.conf
```

`[Interface]` describes the laptop's end of the tunnel:
- `PrivateKey`: the laptop's identity. It never leaves the machine.
- `Address`: the laptop's tunnel IP, `10.20.100.2`. The `/32` means this one address only; the routes come from `AllowedIPs`.

WireGuard has no DHCP, so I picked the tunnel IP myself. It must match the `allowed-address` I gave the laptop peer on the router.

`[Peer]` describes the router:
- `PublicKey`: proves the other end is the hEX.
- `Endpoint`: the hEX's WAN address on the home LAN, where the encrypted packets go.
- `AllowedIPs`: which destinations go through the tunnel. Only the cluster LAN and the tunnel subnet do, so web browsing and the home LAN use the normal connection. This is the split tunnel.
- `PersistentKeepalive`: sends a small packet every 25 seconds so NAT and firewall state stay open between bursts of traffic.

`chmod 600` keeps the file root-only, since it contains the private key.

## Peer and firewall

### Add the laptop as a peer

Replace the key with the laptop's public key from `laptop.pub`.

```
/interface wireguard peers add interface=wg0 public-key="LAPTOP_PUBLIC_KEY" allowed-address=10.20.100.2/32 comment=laptop
```

This tells `wg0` to accept packets from the laptop's key. `allowed-address=10.20.100.2/32` limits that peer to `10.20.100.2` as its source address, so even a valid key cannot pose as another host.

### Open the firewall

```
/interface list member add list=LAN interface=wg0
/ip firewall filter add chain=input action=accept protocol=udp dst-port=51820 in-interface-list=WAN comment="wg: allow wireguard" place-before=[find where comment="defconf: drop all not coming from LAN"]
```

Each packet passes the firewall twice:

- **Encrypted, on ether1.** The default rules drop everything from WAN, so the second command lets in UDP 51820. WireGuard ignores packets without a valid key, so a port scan gets no reply.
- **Decrypted, on wg0.** The first command adds `wg0` to the `LAN` list, so the router trusts tunnel traffic like traffic from the bridge. It can reach SSH and WebFig, and the router forwards it to `10.20.0.x`.

The `LAN` list only controls trust. `wg0` keeps its own subnet, `10.20.100.0/24`.

RouterOS checks filter rules from top to bottom and stops at the first match. Without `place-before`, the accept rule would land below the default drop and never fire. Check the order:

```
/ip firewall filter print
```

The `wg: allow wireguard` rule must appear above `defconf: drop all not coming from LAN`.

## Testing

Unplug the cable from the hEX, join the home Wi-Fi and bring the tunnel up:

```
sudo wg-quick up wg-home
sudo wg show
```

- `wg-quick up wg-home`: reads `/etc/wireguard/wg-home.conf`, creates the `wg-home` interface, assigns the address and adds routes for `AllowedIPs`.
- `wg show`: shows the peer. A `latest handshake` line means both sides accepted each other's keys.

Test each hop in order, so a failure points to one layer:

```
ping -c 3 10.20.100.1
ping -c 3 10.20.0.1
ssh admin@10.20.100.1
```

- `10.20.100.1`: the router's tunnel end. Confirms the tunnel works.
- `10.20.0.1`: the router's LAN address. Confirms routing from the tunnel into the cluster LAN.
- `ssh`: confirms the firewall lets management traffic in over the tunnel.

Take the tunnel down, or start it on boot:

```
sudo wg-quick down wg-home
sudo systemctl enable wg-quick@wg-home
```

### Troubleshooting

- No handshake: check the ISP router still gives the hEX `192.168.1.7`, check the firewall rule sits above the default drop, and check the keys are not swapped.
- Handshake but no ping to `10.20.0.x`: check `wg0` is in the `LAN` interface list.

```
/interface wireguard peers print
/interface list member print
/log print where topics~"wireguard"
```

- `peers print`: shows the peer's key, allowed address and last handshake.
- `list member print`: confirms `wg0` is in `LAN` and `ether1` is in `WAN`.
- `log print`: shows WireGuard log lines, such as handshake failures.

## Hardening

The firewall already blocks router services from the WAN side. Restricting the services as well means a later firewall mistake doesn't expose them. Run these over the tunnel once testing passes.

First confirm the old `192.168.88.1` address is gone:

```
/ip address print
```

Disable telnet and FTP, which send passwords in plain text:

```
/ip service disable telnet,ftp
```

Restrict management to the cluster LAN and the VPN. Each service then refuses connections from any other source address:

```
/ip service set ssh address=10.20.0.0/24,10.20.100.0/24
/ip service set winbox address=10.20.0.0/24,10.20.100.0/24
/ip service set www address=10.20.0.0/24,10.20.100.0/24
```

Restrict the API the same way, but leave it enabled. Ansible's `community.routeros` collection uses it:

```
/ip service set api address=10.20.0.0/24,10.20.100.0/24
/ip service set api-ssl address=10.20.0.0/24,10.20.100.0/24
```

Disable services I don't use: the reverse proxy on 443 and the bandwidth test server on 2000:

```
/ip service disable reverse-proxy
/tool bandwidth-server set enabled=no
```

Optional: allow only modern SSH ciphers and key exchange:

```
/ip ssh set strong-crypto=yes
```

Verify. Every enabled service should list both subnets under ADDRESS, and `btest` should be gone:

```
/ip service print
```

## What I learned

- Change the LAN address in steps. Add the new address before removing the old one and you never lose access.
- `set [find ...]` fails silently when nothing matches. Print after every change.
- Firewall rule order matters. An accept rule below the default drop never runs.
- Adding `wg0` to the `LAN` list is what lets tunnel traffic reach the cluster. It changes trust, not addressing.
- Keep a cable to ether2 as a fallback until the tunnel works.

## Next steps

Next I'll cable the nodes to the hEX, give them static addresses on `10.20.0.0/24` and run the K3s Ansible playbooks over the tunnel.