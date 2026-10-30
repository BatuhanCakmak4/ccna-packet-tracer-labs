# Standard ACL Lab

This lab uses a standard IP access list to control which hosts can reach the server at `172.16.10.100`. It was built in Cisco Packet Tracer.

## Topology

![Standard ACL topology](standartACL.png)

| Device  | Interface | IP Address        | Network          |
|---------|-----------|-------------------|------------------|
| PC0     | Fa0       | 192.168.1.10      | 192.168.1.0/24   |
| PC1     | Fa0       | 192.168.1.20      | 192.168.1.0/24   |
| PC2     | Fa0       | 192.168.2.10      | 192.168.2.0/24   |
| Router0 | Se0/1/0   | 1.1.1.1           | 1.1.1.0/30 (WAN) |
| Router1 | Se0/1/0   | 1.1.1.2           | 1.1.1.0/30 (WAN) |
| Server0 | Fa0       | 172.16.10.100     | 172.16.10.0/24   |

- PC0 and PC1 connect to Router0 Gig0/0 through a 2960 switch.
- PC2 connects directly to Router0 Gig0/1.
- Router0 and Router1 are linked over a serial connection (`1.1.1.0/30`).
- Server0 sits behind Router1 Gig0/0.

## Goal

| Source         | Access to Server0 |
|----------------|-------------------|
| 192.168.2.0/24 (PC2) | Allowed |
| 192.168.1.10 (PC0)   | Allowed |
| Rest of 192.168.1.0/24 (PC1) | Denied |

## ACL Configuration

```
access-list 10 permit 192.168.2.0 0.0.0.255
access-list 10 permit host 192.168.1.10
access-list 10 deny 192.168.1.0 0.0.0.255
```

A standard ACL only checks the source address, so it should be placed as close to the destination as possible. Otherwise it would also block traffic heading to other networks. It is applied outbound on the interface facing the server:

```
interface <interface-facing-server>
 ip access-group 10 out
```

Rule order matters: `permit host 192.168.1.10` has to come before `deny 192.168.1.0 0.0.0.255`, or PC0 would be caught by the deny line first. Every ACL also ends with an implicit `deny any`.

## Verification

After sending pings from the PCs to the server, `show access-lists` shows that every rule is matching traffic:

![show access-lists output](show-access-lists.png)

- PC2 → Server0: ping succeeds
- PC0 → Server0: ping succeeds
- PC1 → Server0: ping fails

## Files

- `standartACL.pkt`: Packet Tracer lab file
- `standartACL.png`: topology screenshot
- `show-access-lists.png`: ACL verification output
