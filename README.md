# CCNA Labs

This repository holds the hands-on networking labs I build while preparing for the Cisco Certified Network Associate (CCNA) certification. Most labs are built in Cisco Packet Tracer; some use Wireshark to look at real traffic on my own network.

Each lab folder contains the `.pkt` file, so you can open it in Packet Tracer and look at the configuration yourself. The repo grows as I work through new CCNA topics.

## Labs

| # | Lab | Topic | Tools | Folder |
|---|-----|-------|-------|--------|
| 1 | [ARP Protocol Test](#1-arp-protocol-test) | ARP requests, replies and probes | Packet Tracer, Wireshark | [`arp_protocol_test/`](arp_protocol_test/) |
| 2 | [Static Routing](#2-static-routing) | Static routes between three routers | Packet Tracer | [`staticrouting/`](staticrouting/) |
| 3 | [Standard ACL](#3-standard-acl) | Filtering traffic by source address | Packet Tracer | [`standartACL/`](standartACL/) |
| 4 | [Extended ACL](#4-extended-acl) | Filtering by source, destination and protocol (FTP / ICMP) | Packet Tracer | [`extendedACL/`](extendedACL/) |
| 5 | [Telnet ACL](#5-telnet-acl) | Restricting remote (VTY) access to the router | Packet Tracer | [`telnetACL/`](telnetACL/) |

---

## 1. ARP Protocol Test

ARP (Address Resolution Protocol) maps an IPv4 address to a MAC address inside a local network. In this lab I captured live ARP traffic with Wireshark, using the `arp` display filter.

![Wireshark ARP capture](arp_protocol_test/ARP_TRAFIK.png)

What the capture shows:

- **ARP requests are broadcast.** `Who has 192.168.1.1? Tell 192.168.1.38` goes to every device on the LAN, here a host looking up the MAC of its default gateway.
- **ARP replies are unicast.** `192.168.1.47 is at 26:aa:b4:40:53:37` goes straight back to the device that asked (`192.168.1.48`).
- **ARP probes.** `Who has 192.168.1.38? (ARP Probe)` is a device checking that nobody else is already using the address it wants before it claims it (duplicate address detection).
- **Retries.** When no reply arrives, the same request is sent again (`192.168.1.47` was asked twice before it answered).

Files: [`arp_protocol_test.pkt`](arp_protocol_test/arp_protocol_test.pkt), [`ARP_TRAFIK.png`](arp_protocol_test/ARP_TRAFIK.png)

---

## 2. Static Routing

Three 1941 routers, named after cities, are connected over serial links. Each router has two LANs behind it, and the goal is for every PC to reach every other PC using only static routes.

![Static routing topology](staticrouting/staticrouting.png)

| Router   | LAN (interface)              | Host             | WAN link                       |
|----------|------------------------------|------------------|--------------------------------|
| KOCAELİ  | 192.168.3.0/24 (Gig0/0)      | PC15 192.168.3.3 | Se0/1/0 → ISTANBUL (1.1.1.0/24) |
|          | 192.168.4.0/24 (Gig0/1)      | PC16 192.168.4.4 |                                |
| ISTANBUL | 192.168.1.0/24 (Gig0/0)      | PC17 192.168.1.2 | Se0/1/0 → KOCAELİ (1.1.1.0/24) |
|          | 192.168.2.0/24 (Gig0/1)      | PC18 192.168.2.2 | Se0/1/1 → BURSA (2.2.2.0/24)   |
| BURSA    | 192.168.5.0/24 (Gig0/0)      | PC13 192.168.5.5 | Se0/1/1 → ISTANBUL (2.2.2.0/24) |
|          | 192.168.6.0/24 (Gig0/1)      | PC14 192.168.6.6 |                                |

ISTANBUL is the hub. A router only knows its directly connected networks, so each one needs a static route for every remote LAN:

| Router   | Networks it must learn statically                    | Next hop    |
|----------|------------------------------------------------------|-------------|
| KOCAELİ  | 192.168.1.0, 192.168.2.0, 192.168.5.0, 192.168.6.0, 2.2.2.0 | ISTANBUL |
| BURSA    | 192.168.1.0, 192.168.2.0, 192.168.3.0, 192.168.4.0, 1.1.1.0 | ISTANBUL |
| ISTANBUL | 192.168.3.0, 192.168.4.0                              | KOCAELİ     |
|          | 192.168.5.0, 192.168.6.0                              | BURSA       |

The general form of the command:

```
ip route <destination-network> <subnet-mask> <next-hop-ip | exit-interface>
```

Since KOCAELİ and BURSA have only one way out, a default route (`ip route 0.0.0.0 0.0.0.0 <next-hop>`) toward ISTANBUL would also work on them.

Files: [`staticrouting.pkt`](staticrouting/staticrouting.pkt), [`staticrouting.png`](staticrouting/staticrouting.png)

---

## 3. Standard ACL

A standard IP access list controls which hosts can reach the server at `172.16.10.100`. PC2's network and PC0 are allowed; the rest of `192.168.1.0/24` (PC1) is blocked.

![Standard ACL topology](standartACL/standartACL.png)

```
access-list 10 permit 192.168.2.0 0.0.0.255
access-list 10 permit host 192.168.1.10
access-list 10 deny 192.168.1.0 0.0.0.255
```

A standard ACL only checks the source address, so it goes as close to the destination as possible and is applied outbound on the interface facing the server. Rule order matters: the `permit host` line has to come before the `deny` for its subnet.

After pinging the server from all three PCs, `show access-lists` shows every rule matching traffic:

![show access-lists output](standartACL/show-access-lists.png)

Full write-up: [`standartACL/README.md`](standartACL/README.md)

---

## 4. Extended ACL

The same idea as the standard ACL lab, but now the filter looks at the protocol too. Two routers are linked over a `1.1.1.0/30` serial link: Router0 (`.1`) has the `192.168.1.0/24` LAN (PC0, PC1) and the `192.168.2.0/24` LAN (PC2), and Router1 (`.2`) has the server at `172.16.10.100`.

![Extended ACL topology](extendedACL/topology.png)

The goal: only PC0 (`192.168.1.10`) may use FTP on the server, the rest of `192.168.1.0/24` may not, and ping has to keep working for everyone.

```
access-list 105 permit tcp host 192.168.1.10 host 172.16.10.100 eq ftp
access-list 105 deny tcp 192.168.1.0 0.0.0.255 host 172.16.10.100 eq ftp
access-list 105 permit icmp any any
```

An extended ACL (100–199) checks source, destination, protocol and port, so it can block one service without cutting off the whole host. That is why it is usually placed close to the source, before unwanted traffic crosses the WAN link. The last line matters: every ACL ends with an implicit `deny any`, so without `permit icmp any any` ping would be blocked as well. Any other traffic not listed here (HTTP, for example) is still dropped by that implicit deny.

Testing from PC1 (`192.168.1.20`): the FTP connection times out, but ping to the server gets all four replies.

![FTP blocked, ping allowed](extendedACL/ping-ftp-test.png)

`show access-lists` confirms that each rule is being hit:

![show access-lists output](extendedACL/show-acl.png)

Files: [`extendedACL.pkt`](extendedACL/extendedACL.pkt), [`topology.png`](extendedACL/topology.png), [`ping-ftp-test.png`](extendedACL/ping-ftp-test.png), [`show-acl.png`](extendedACL/show-acl.png)

---

## 5. Telnet ACL

ACLs can also protect the router itself. Here PC0 (NETWORK ADMIN) and PC1 (USER) sit on the same switch behind Router0, and only the admin should be able to Telnet into the router.

![Telnet ACL topology](telnetACL/topology.png)

A standard ACL lists the allowed host, and `access-class` applies it to the VTY lines instead of an interface:

```
access-list 10 permit host 192.168.1.3
access-list 10 deny any
!
line vty 0 4
 access-class 10 in
```

![Telnet ACL configuration](telnetACL/acltelnetconfig.png)

`access-group` filters traffic passing through an interface; `access-class` filters who can open a remote session (Telnet/SSH) to the device. Traffic from the user PC still goes through the router normally; only its Telnet attempts to the router are refused. The explicit `deny any` is not strictly needed because of the implicit deny, but it makes the intent clear.

Files: [`telnetACL.pkt`](telnetACL/telnetACL.pkt), [`topology.png`](telnetACL/topology.png), [`acltelnetconfig.png`](telnetACL/acltelnetconfig.png)

---

## Repository Structure

```
CCNA/
├── README.md
├── arp_protocol_test/
│   ├── arp_protocol_test.pkt
│   └── ARP_TRAFIK.png
├── staticrouting/
│   ├── staticrouting.pkt
│   └── staticrouting.png
├── standartACL/
│   ├── README.md
│   ├── standartACL.pkt
│   ├── standartACL.png
│   └── show-access-lists.png
├── extendedACL/
│   ├── extendedACL.pkt
│   ├── topology.png
│   ├── ping-ftp-test.png
│   └── show-acl.png
└── telnetACL/
    ├── telnetACL.pkt
    ├── topology.png
    └── acltelnetconfig.png
```

## How to Use

1. Install [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer).
2. Clone the repo: `git clone <repo-url>`
3. Open any `.pkt` file in Packet Tracer to see the topology and device configs.
