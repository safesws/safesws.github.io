---
layout: post
title:  Linux Network Packets Path
categories: [Linux, Network, Kernel]
---

## INTRO

This post covers the __TCP/IP__ and __UDP/IP__ paths on kernel __5.10+__, with interactive diagrams.

Let's go:

- [Interactive Diagrams](#interactive-diagrams)
<br>
- [Egress Path (TX)](#egress-path-(tx))
<br>
- [Ingress Path (RX)](#ingress-path-(rx))
<br>
- [References](#references)

## Interactive Diagrams

The diagrams below trace the major kernel functions on both the egress (_TX_) and ingress (_RX_) paths, plus the `sk_buff` buffer layout and __netfilter/eBPF__ hook points. 

<p style="text-align:right; margin-bottom:6px;">
  <a href="../diagrams/linux-packet-path-diagram.html" target="_blank" rel="noopener" style="font-size:13px;">
    ↗ Open full diagram in new tab
  </a>
</p>
<iframe
  src="../diagrams/linux-packet-path-diagram.html"
  style="width:100%; height:85vh; border:1px solid #2a3545; border-radius:8px; background:#0a0e14;"
  loading="lazy"
  allowfullscreen>
</iframe>
<noscript>
  <p><a href="../diagrams/linux-packet-path-diagram.html">Open the interactive diagram</a> (requires JavaScript).</p>
</noscript>

---

## Egress Path (TX)

### 1. Userspace → Socket Layer

A process calls `write()`, `send()`, or `sendto()` on a socket __fd__. The __VFS dispatches__ through `sock_sendmsg()`, which pulls the `struct sock` from the __fd__, attaches credentials from `task_struct` (_PID/UID/GID_), runs __LSM hooks__ (_SELinux/AppArmor_), then dispatches to the _transport protocol_ via the `sk_prot->sendmsg` function pointer. 
<br>
The `INDIRECT_CALL_INET` macro is an optimization that avoids indirect branch prediction penalties by hardcoding checks for the two most common protocols (_TCP/UDP_) before falling through to the generic indirect call.

### 2. Transport Layer — TCP

`tcp_sendmsg()` first checks that the connection is __ESTABLISHED__, then iterates over the user buffer, allocating `sk_buff` structures sized to __MSS__. 
<br>
Data is copied from userspace via `skb_add_data_nocache()` (_into existing tail room_) or by allocating new pages. 
<br>
Segments are enqueued to the _socket write queue_ (`sk->sk_write_queue`). 
<br>
Then `tcp_push()` → `tcp_write_xmit()` walks the queue, applying congestion window (`cwnd`) and receiver window (`rwnd`) constraints, setting retransmission timers, building the TCP header (_seq/ack/flags/window/options_), computing the checksum (or deferring it to hardware via `CHECKSUM_PARTIAL`), and finally calling `ip_queue_xmit()` through the `icsk_af_ops->queue_xmit` function pointer.

### Transport Layer — UDP

`udp_sendmsg()` is simpler: it resolves the route via `ip_route_output_flow()`, handles __corking__ (`UDP_CORK` — batching multiple `sendmsg()` calls into one IP datagram) vs __non-corking__ (_immediate send_), builds the UDP header, computes the checksum, and calls `ip_make_skb()` + `udp_send_skb()` which hands off to IP.

### 3. IP Layer

`__ip_queue_xmit()` (_TCP path_) or `ip_push_pending_frames()` (_UDP path_) handles route lookup via the __FIB__ (_Forwarding Information Base_ — the compiled routing table). 
<br>
If the route is cached in `skb->_skb_refdst`, the lookup is skipped. 
<br>
The IP header is constructed (version, IHL, TOS, TTL, protocol, src/dst addresses). 
<br>
Then:

- `NF_INET_LOCAL_OUT` netfilter hook fires — this is where _iptables/nftables_ __OUTPUT__ chain rules execute, and where conntrack begins tracking the flow.
- `ip_output()` fires `NF_INET_POST_ROUTING` — the __POSTROUTING__ chain (_SNAT/masquerade happens here_).
- `ip_finish_output()` checks __MTU__ and fragments if necessary via `ip_fragment()`. Fragmentation is avoided if the __DF__ bit is set (_PMTUD_).
- Neighbor subsystem resolves __L2__ address: `neigh_resolve_output()` does __ARP__ lookup (or uses the neighbor cache). If no __ARP__ reply exists, the __skb__ is queued in the neighbor's `arp_queue` _pending_ resolution.
- The Ethernet header is pushed onto the __skb__.

### 4. Qdisc / Device Layer

`dev_queue_xmit()` sets `skb->mac_header`, then enters the queueing discipline (__qdisc__). 
<br>
The default is `pfifo_fast` (or `fq_codel` on modern distros). `__qdisc_run()` dequeues __skbs__, runs `validate_xmit_skb()` (_VLAN tag insertion, GSO/TSO segmentation if hardware supports it, checksum finalization_), then calls the driver's `ndo_start_xmit()`. 
<br>
The __skb__ is placed in the TX ring buffer (typically a DMA-mapped ring descriptor). 
<br>
The driver writes the descriptor and pokes the __NIC's__ doorbell register (_MMIO write_) to trigger transmission.

**Bypass paths:** 
<br>
`dev_direct_xmit()` is used by __XDP__ and `AF_XDP` to _skip_ the __qdisc__ entirely. 
<br>
`XDP_TX` reflects a packet at the driver level without ever going up the stack. 
<br>
TC egress (`tc_egress()` / `tcf_classify()`) runs __tc-BPF__ or __u32/flower__ classifiers between __qdisc__ enqueue and the driver.

---

## Ingress Path (RX)

### 1. NIC → Driver

The __NIC__ _DMAs_ the packet into a pre-allocated ring buffer (_RX ring_), writes the descriptor with metadata (_length, checksum status, RSS hash_), and raises a hardware interrupt (_or_ __MSI-X__ _vector_). 
<br>
The driver's __ISR__ calls `napi_schedule()` to schedule __NAPI__ polling, then _masks_ the interrupt. This interrupt _coalescing_ is critical — without __NAPI__, per-packet interrupts would kill throughput.

### 2. NAPI Poll / Driver → netdev

In __softirq__ context (`NET_RX_SOFTIRQ`), `napi_poll()` calls the driver's poll function, which walks the _RX ring_, allocates `sk_buff` structures, fills in metadata (_protocol via `eth_type_trans()`, device, rx hash_), and calls `napi_gro_receive()`.

**GRO (Generic Receive Offload)** _coalesces_ multiple TCP segments into a single large __skb__ before passing it up, reducing per-packet overhead.

### 3. `netif_receive_skb()`

- `skb->mac_header` is set, Ethernet header is pulled.
- `af_packet` sockets (_tcpdump/libpcap_) get a clone here via `deliver_skb()` to all registered `ptype_all` handlers.
- `tc_ingress()` runs if a clsact/ingress __qdisc__ is attached — this is a major __eBPF__ hook point (`BPF_PROG_TYPE_SCHED_CLS`).
- VLAN tagged frames are dispatched to the correct VLAN sub-interface.
- rx_handler()` can steal the packet if the interface is enslaved to a bridge or has a registered __rx_handler__.
- protocol demux dispatches to `ip_rcv()` based on `skb->protocol` (ETH_P_IP).

### 4. IP Layer

`ip_rcv()` validates the IP header (_version == 4, IHL >= 5, total length consistent, header checksum_), sets `skb->transport_header`, fires `NF_INET_PRE_ROUTING` (_PREROUTING chain, DNAT, conntrack_). 
<br>
`ip_rcv_finish()` does the route lookup via `ip_route_input_noref()` → __FIB__ lookup. 
<br>
The routing decision sets `skb->dst->input` to one of:

- `ip_local_deliver()` — packet is for us.
- `ip_forward()` — packet needs forwarding (decrements TTL, fires `NF_INET_FORWARD` hook).
- `ip_mr_input()` — multicast routing.

For local delivery: `ip_defrag()` reassembles fragments, `NF_INET_LOCAL_IN` fires (_INPUT chain_), then `ip_local_deliver_finish()` strips the IP header and dispatches to the transport protocol handler.

### 5. Transport Layer — TCP

`tcp_v4_rcv()` validates the TCP header, verifies the checksum, looks up the socket via `__inet_lookup_skb()` (_established hash table first, then listener table_). 
<br>
For established connections: `tcp_v4_do_rcv()` → `tcp_rcv_established()`.

- **Fast path**: header prediction (_predicted next seq/ack, no special flags, window unchanged_) → data is copied directly to userspace or queued to `sk->sk_receive_queue`. 
- **Slow path**: out-of-order segments, SACK processing, ECN, urgent data — full state machine treatment.

For SYN packets to listeners: `tcp_v4_cookie_check()` (_SYN cookies if SYN flood_), `tcp_check_req()` for _3WHS_ completion.

### Transport Layer — UDP

`udp_rcv()` → `__udp4_lib_rcv()`: _socket lookup by destination port_, checksum verification, `udp_queue_rcv_skb()` enqueues to `sk->sk_receive_queue`, `sk->sk_data_ready()` wakes the reader.

### 6. Socket Layer / Userspace Read

`read()` → `sock_recvmsg()` → `inet_recvmsg()` → `tcp_recvmsg()` / `udp_recvmsg()`. 
<br>
Data is copied from __skb(s)__ in the receive queue to the userspace buffer. The __skbs__ are freed after consumption.

## References

### Primary

- Stephan & Wüstrich, [*The Path of a Packet Through the Linux Kernel*](https://www.net.in.tum.de/fileadmin/TUM/NET/NET-2024-04-1/NET-2024-04-1_16.pdf) (TUM, 2024)
- PackageCloud, [*Monitoring and Tuning the Linux Networking Stack: Receiving Data*](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-receiving-data/)
- PackageCloud, [*Monitoring and Tuning the Linux Networking Stack: Sending Data*](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-sending-data/)
- PackageCloud, [*Illustrated Guide to Monitoring and Tuning the Linux Networking Stack: Receiving Data*](https://blog.packagecloud.io/illustrated-guide-monitoring-tuning-linux-networking-stack-receiving-data/)

### Official Kernel Documentation

- [Networking subsystem index](https://docs.kernel.org/networking/index.html)
- [NAPI documentation](https://docs.kernel.org/networking/napi.html)
- [sk_buff documentation](https://docs.kernel.org/networking/skbuff.html)
- [Kernel networking API](https://www.kernel.org/doc/html/v4.14/networking/kapi.html)
- [`/proc/sys/net/` tuning parameters](https://docs.kernel.org/admin-guide/sysctl/net.html)
- [Kernel source browser (v5.10.8)](https://elixir.bootlin.com/linux/v5.10.8/source)

### Linux Foundation Wiki

- [Kernel Flow — full packet path diagram](https://wiki.linuxfoundation.org/networking/kernel_flow)
- [NAPI driver implementation guide](https://wiki.linuxfoundation.org/networking/napi)

### Academic Papers

- Høiland-Jørgensen et al., [*The eXpress Data Path: Fast Programmable Packet Processing in the Operating System Kernel*](https://dl.acm.org/doi/10.1145/3281411.3281443) (ACM CoNEXT '18)
- Cai et al., [*Understanding Host Network Stack Overheads*](https://dl.acm.org/doi/10.1145/3452296.3472888) (ACM SIGCOMM '21) — [PDF](https://www.cs.cornell.edu/~ragarwal/pubs/network-stack.pdf)
- Chimata, [*Path of a Packet in the Linux Kernel Stack*](https://www.cs.dartmouth.edu/~sergey/netreads/path-of-packet/Network_stack.pdf) (2005, kernel 2.6.11)

### Community

- [*Linux Network Performance Ultimate Guide*](https://ntk148v.github.io/posts/linux-network-performance-ultimate-guide/)
- [*Path of a Received Packet in the Kernel — Overview*](https://www.sheharyaar.in/blog/packet-path-overview/) (Sheharyaar, 2024)
- [DaveM's Linux Networking Blog](http://vger.kernel.org/~davem/cgi-bin/X_blog.cgi) — GRO internals

### eBPF / XDP

- [Cilium BPF and XDP Reference Guide](https://docs.cilium.io/en/latest/bpf/)
- [XDP paper source + benchmarks](https://github.com/xdp-project/xdp-paper)
- [SIGCOMM '21 network stack profiling tools](https://github.com/Terabit-Ethernet/Understanding-network-stack-overheads-SIGCOMM-2021)
