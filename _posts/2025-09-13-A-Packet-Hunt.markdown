---
layout: post
title:  "A Packet Hunt in a Virtualized Environment"
date:   2025-09-13 01:01:00 +0000
categories: virtualization networking
---

I recently encountered an interesting networking issue while trying to get a Windows guest to connect to the network. The guest was started with OpenVMM, and while Linux guests worked perfectly, Windows guests couldn't obtain DHCP leases.

## Network Configuration

To configure guest networking with minimal steps, I use the following commands:

```bash
sudo systemctl start virtnetworkd
sudo systemctl start virtqemud
sudo virsh net-start default

$ ip link show virbr0
6: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc htb state DOWN mode DEFAULT group default qlen 1000
  link/ether 52:54:00:77:6f:b8 brd ff:ff:ff:ff:ff:ff
```

These commands start a bridge (`virbr0`) on the host and a `dnsmasq` process to serve DHCP and DNS requests from guests. This setup creates a NAT network on the host, where network traffic to and from the guest is routed through forwarding rules correctly configured by the libvirt services.

Guests need to receive DHCP responses to connect to the configured network. This blog will focus primarily on DHCP traffic analysis.

## DHCP Packet Flow Analysis

To properly understand the issue, let's trace the path of DHCP request and response packets from a guest:

1. **Guest OS** sends a DHCP request packet
2. **TAP device** receives the packet on the host
3. **ebtables** filters packets from the TAP device before forwarding them to the connected bridge interface
4. If the packet passes ebtables filtering, it proceeds to the **bridge** (`virbr0`)
5. If the `br_netfilter` module is loaded and `net.bridge.bridge-nf-call-iptables=1`, the host upcalls L2 frames to the L3 IP filtering stack (**iptables**)
6. If the packet passes iptables filtering, it reaches the bridge
7. If `net.bridge.bridge-nf-call-iptables=0`, the packet is forwarded to the bridge without iptables filtering
8. **dnsmasq** process sees the DHCP request packets and sends a DHCP response
9. The response packet travels back: **bridge** → **TAP** → **Guest OS**

## Reproducing the Issue

I ran the following command to initiate a DHCP request from the Windows guest:

```cmd
C:\Windows\system32>ipconfig /renew

Windows IP Configuration

An error occurred while renewing interface Ethernet 6 : unable to contact your DHCP server. Request has timed out.
```

As shown above, the request times out and the guest never receives a DHCP response.

## Initial Investigation

Using the following commands, I confirmed that DHCP request packets were arriving at both the TAP interface and the `virbr0` bridge:

```
## TAP device
sudo tshark -i ovmtap0 -f "udp port 67 or udp port 68"
Running as user "root" and group "root". This could be dangerous.
Capturing on 'ovmtap0'
    1 0.000000000      0.0.0.0 → 255.255.255.255 DHCP 342 DHCP Discover - Transaction ID 0x1ade762b
    2 3.675131047      0.0.0.0 → 255.255.255.255 DHCP 342 DHCP Discover - Transaction ID 0x1ade762b


## Bridge device
$ tshark_dhcp virbr0
Running as user "root" and group "root". This could be dangerous.
Capturing on 'virbr0'
    1 0.000000000      0.0.0.0 → 255.255.255.255 DHCP 344 DHCP Discover - Transaction ID 0x7072f6cc
    2 3.804583920      0.0.0.0 → 255.255.255.255 DHCP 344 DHCP Discover - Transaction ID 0x7072f6cc
```

By running strace on `dnsmasq` process I confirmed that the process does not
read the packets received on the bridge. I added logging steps to double check
the packets are not being dropped by above mentioned filters (iptables, ebtables, etc).
The packets seem to be dropped while being processed within the bridge.

After some research I came across
`dropwatch` [https://linux.die.net/man/1/dropwatch] userspace tool. This tools
helps identify the location at which kernel dropped packets. To limit noise from
other interfaces, I disabled all the other network interfaces on the host and
initiated DHCP request from the guest again. Everytime I see a DHCP packet in
virbr0, I see a line like below from dropwatch:

```bash
sudo dropwatch -l kas
Initializing kallsyms db
dropwatch> start
Enabling monitoring...
Kernel monitoring activated.
Issue Ctrl-C to stop monitoring
...
1 drops at ip_rcv_core+239 (0xffffffff87ce3589) [software]
...
2 drops at ip_rcv_core+239 (0xffffffff87ce3589) [software]
...
```

This was my first hint about what might be dropping the packets. Before investigating further, I confirmed that these messages from dropwatch didn't appear when using a Linux guest.

## Deep Dive into the Kernel Code

To investigate the dropwatch report further, I used the kernel's address-to-line tool:

```bash
$ ./scripts/faddr2line vmlinux ip_rcv_core+239
ip_rcv_core+239/0x360:
ip_rcv_core at net/ipv4/ip_input.c:506
```

**Corresponding kernel code:**
```c
  453 /*
  454  *  Main IP Receive routine.
  455  */
  456 static struct sk_buff *ip_rcv_core(struct sk_buff *skb, struct net *net)
  457 {
...
  505
  506   if (!pskb_may_pull(skb, iph->ihl*4))
  507     goto inhdr_error;
  508
  509   iph = ip_hdr(skb);
  510
  511   if (unlikely(ip_fast_csum((u8 *)iph, iph->ihl)))
  512     goto csum_error;
```

After adding logging to check the specific condition causing packet drops, I discovered that the issue wasn't at line 506, but at line 511. **The packets were being dropped because IPv4 checksum verification failed.**

## Understanding the Problem

At this point, I consulted Copilot's assistance:

```
**Prompt:** Why are packets being dropped by checksum failures when using a Windows guest but not a Linux guest?

**Answer:** This usually happens because of checksum offloading differences between Windows and Linux guests in virtualized environments.

### Why Windows guests fail:

- Windows networking stack uses checksum offload, but:
  - If the hypervisor or virtual switch doesn't fully support offload, or
  - If the virtual NIC driver in Windows is misconfigured or buggy,
  
  Then packets may leave the guest with incorrect or uninitialized checksums and reach the Linux host without being corrected.

- When Linux receives such packets, it verifies the checksum in software (e.g., `ip_fast_csum`) and drops them because they appear corrupted.

### Recommended fixes:

**Disable checksum offloading in the Windows guest:**

In Device Manager → Network Adapter → Advanced → Disable:
- IPv4 Checksum Offload
- TCP Checksum Offload  
- UDP Checksum Offload

```

## The Solution

Following the recommendation above, I modified the network adapter settings: 

**Device Manager** → **Network Adapters** → **Microsoft Hyper-V Network Adapter** → **Advanced** → **{IPv4 Checksum Offload, TCP Checksum Offload, UDP Checksum Offload}** → **Disabled**

After modifying above configurations and restarting the Windows guest, the guest successfully received a DHCP response and
able to connect to internet.

```cmd
C:\Windows\system32>ipconfig

Windows IP Configuration

Ethernet adapter Ethernet 3:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::6068:fd4:7af:75d8
   IPv4 Address. . . . . . . . . . . : 192.168.122.99
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.122.1
```

## Conclusion

This issue highlights the importance of understanding how different operating systems handle network features like checksum offloading in virtualized environments. Disabling checksum offloading in the Windows guest restored network connectivity, but that change could be a workaround rather than a definitive fix. Determining the root cause and implementing a proper fix requires further investigation.

Key takeaways:
1. **Use packet analysis tools** like `tshark` and `dropwatch` to trace network issues.
2. **Understand the complete packet flow** in virtualized networking.
3. **Consider OS-specific network behaviors** when troubleshooting cross-platform issues.
4. **Checksum offloading** can cause compatibility issues between hypervisors and guest operating systems.
