# Program type `BPF_PROG_TYPE_XDP`

<!-- [FEATURE_TAG](BPF_PROG_TYPE_XDP) -->
[:octicons-tag-24: v4.8](https://github.com/torvalds/linux/commit/6a773a15a1e8874e5eccd2f29190c31085912c95)
<!-- [/FEATURE_TAG] -->

XDP (Express Data Path) programs can attach to network devices and are called for every incoming (ingress) packet received by that network device. XDP programs can take quite a large number of actions, most prominent of which are manipulation of the packet, dropping the packet, redirecting it and letting it pass to the network stack.

Notable use cases for XDP programs are for DDoS protection, Load Balancing, and high-throughput packet filtering. If loaded with native driver support, XDP programs will be called just after receiving the packet but before allocating memory for a socket buffer. This callsite makes XDP programs extremely performant, especially in use cases where traffic is forwarded or dropped a lot in comparison to other eBPF program types or techniques which run after the relatively expensive socket buffer allocation process has taken place, only to discard it.

## Usage

XDP programs are typically put into an [ELF](../../elf.md) section prefixed with `xdp`. The XDP program is called by the kernel with a [xdp_md](../program-context/xdp_md.md) context. The return value indicates what action the kernel should take with the packet, the following values are permitted:

* `XDP_ABORTED` - Signals that a unrecoverable error has taken place. Returning this action will cause the kernel to trigger the `xdp_exception` tracepoint and print a line to the trace log. This allows for debugging of such occurrences. It is also expensive, so should not be used without consideration in production.
* `XDP_DROP` - Discards the packet. It should be noted that since we drop the packet very early, it will be invisible to tools like `tcpdump`. Consider recording drops using a custom feedback mechanism to maintain visibility.
* `XDP_PASS` - Pass the packet to the network stack. The packet can be manipulated before hand
* `XDP_TX` - Send the packet back out the same network port it arrived on. The packet can be manipulated before hand.
* `XDP_REDIRECT` - Redirect the packet to one of a number of locations. The packet can be manipulated before hand.

`XDP_REDIRECT` should not be returned by itself, always in combination with a helper function call. A number of helper functions can be used to redirect the current packet. These annotate hidden values in the context to inform the kernel what actual redirection action to take after the program exists.

Packets can be redirected in the following ways:

* The packet can be redirected to egress on a different interface than where it entered (like `XDP_TX` but for a different interface). This can be done using the [`bpf_redirect`](../helper-function/bpf_redirect.md) helper (not recommended) or the [`bpf_redirect_map`](../helper-function/bpf_redirect_map.md) helper in combination with a [`BPF_MAP_TYPE_DEVMAP`](../map-type/BPF_MAP_TYPE_DEVMAP.md) or [`BPF_MAP_TYPE_DEVMAP_HASH`](../map-type/BPF_MAP_TYPE_DEVMAP_HASH.md) map.
* The packet can be redirected to another CPU for further processing using the [`bpf_redirect_map`](../helper-function/bpf_redirect_map.md) helper in combination with a [`BPF_MAP_TYPE_CPUMAP`](../map-type/BPF_MAP_TYPE_CPUMAP.md) map.
* The packet can be redirected to userspace, bypassing the kernel network stack using the [`bpf_redirect_map`](../helper-function/bpf_redirect_map.md) helper in combination with a [`BPF_MAP_TYPE_XSKMAP`](../map-type/BPF_MAP_TYPE_XSKMAP.md) map

## Context

XDP programs are called with the `struct xdp_md` context. This is a very simple context representing a single packet.

### `data`

[:octicons-tag-24: v4.8](https://github.com/torvalds/linux/commit/6a773a15a1e8874e5eccd2f29190c31085912c95)

This field contains a pointer to the start of packet data. The XDP program can read from this region between `data` and `data_end`, as long as it always performs bounds checks.

### `data_end`

[:octicons-tag-24: v4.8](https://github.com/torvalds/linux/commit/6a773a15a1e8874e5eccd2f29190c31085912c95)

This field contains a pointer to the end of the packet data. The verifier will enforce that any XDP program checks that offsets from `data` are less then `data_end` before the program attempts to read from it.

### `data_meta`

[:octicons-tag-24: v4.15](https://github.com/torvalds/linux/commit/de8f3a83b0a0fddb2cf56e7a718127e9619ea3da)

This field contains a pointer to the start of a metadata region in the packet memory. By default, no metadata room is available, so the value of `data_meta` and `data` will be the same. The XDP program can request metadata with the [`bpf_xdp_adjust_meta`](../helper-function/bpf_xdp_adjust_meta.md) helper, on success `data_meta` is updated so it is not less then `data`. The room between `data_meta` and `data` is freely useable by the XDP program.

If the packet with metadata is passed to the kernel, that metadata will be available in the [`__sk_buff`](../program-context/__sk_buff.md) via its [`data_meta`](../program-context/__sk_buff.md#data_meta) and `data` fields.

This means that XDP programs can communicate information to for example `BPF_PROG_TYPE_SCHED_CLS` programs which can then manipulate the socket buffer to change `__sk_buff->mark` or `__sk_buff->priority` on behalf of an XDP program.

### `ingress_ifindex`

[:octicons-tag-24: v4.16](https://github.com/torvalds/linux/commit/02dd3291b2f095bbc88e1d2628fd5bf2e92de69b)

This field contains the network interface index the packet arrived on.

### `rx_queue_index`

[:octicons-tag-24: v4.16](https://github.com/torvalds/linux/commit/02dd3291b2f095bbc88e1d2628fd5bf2e92de69b)

This field contains the queue index within the NIC on which the packet was received.

!!! note
    While this field is normally read-only, offloaded XDP programs are allowed to write to it to perform custom RSS (Receive-Side Scaling) in the network device [:octicons-tag-24: v4.18](https://github.com/torvalds/linux/commit/0d8300325660f81787892a1c58dc1f9428a67143)

### `egress_ifindex`

[:octicons-tag-24: v5.8](https://github.com/torvalds/linux/commit/64b59025c15b244c0954cf52b24fbabfcf5ed8f6)

This field is read-only and contains the network interface index the packet has been redirected out of. This field is only ever set after an initial XDP program redirected a packet to another device with a [`BPF_MAP_TYPE_DEVMAP`](../map-type/BPF_MAP_TYPE_DEVMAP.md) and the value of the devmap contained a file descriptor of a secondary XDP program. This secondary program will be invoked with a context that has `egress_ifindex`, `rx_queue_index`, and `ingress_ifindex` set so it can modify fields in the packet to match the redirection.

### XDP fragments

[:octicons-tag-24: v5.18](https://github.com/torvalds/linux/commit/c2f2cdbeffda7b153c19e0f3d73149c41026c0db)

An increasingly common performance optimization technique is to use larger packets and to bulk process them (Jumbo packets, GRO, BIG-TCP). It might therefor happen that packets get larger than a single memory page or that we want to glue multiple already allocated packets together. This breaks the existing assumption XDP programs have of all the packet data living in a linear area between `data` and `data_end`. 

In order to offer support and not break existing programs, the concept of "XDP fragment aware" programs was introduced. XDP program authors writing such programs can compare the length between the `data` and `data_end` pointer and the output of [`bpf_xdp_get_buff_len`](../helper-function/bpf_xdp_get_buff_len.md). If the XDP program needs to work with data beyond the linear portion it should use the [`bpf_xdp_load_bytes`](../helper-function/bpf_xdp_load_bytes.md) and [`bpf_xdp_store_bytes`](../helper-function/bpf_xdp_store_bytes.md) helpers.

To indicate that a program is "XDP Fragment aware" the program should be loaded with the [`BPF_F_XDP_HAS_FRAGS`](../syscall/BPF_PROG_LOAD.md#bpf_f_xdp_has_frags) flag. Program authors can indicate that they wish libraries like libbpf to load programs with this flag by placing their program in a `xdp.frags/` ELF section instead of a `xdp/` section.

!!! note
    If a program is both "XDP Fragment aware" and should be attached to a CPUMAP or DEVMAP the two ELF naming conventions are combined: `xdp.frags/cpumap/` or `xdp.frags/devmap`.

!!! warning
    XDP fragments are not supported by all network drivers, check the [driver support](#driver-support) table.

## Attachment

There are two ways of attaching XDP programs to network devices, the legacy way of doing is is via a [netlink](https://man7.org/linux/man-pages/man7/netlink.7.html) socket the details of which are complex. Examples of libraries that implement netlink XDP attaching are [vishvananda/netlink](https://github.com/vishvananda/netlink/blob/afa2eb2a66aac1f8f370287f236ba93d4c078dd6/link_linux.go#L934) and [libbpf](https://github.com/libbpf/libbpf/blob/ea284299025bf85b85b4923191de6463cd43ccd6/src/netlink.c#L321).

The modern and recommended way is to use BPF links. Doing so is as easy as calling [`BPF_LINK_CREATE`](../syscall/BPF_LINK_CREATE.md) with the `target_ifindex` set to the network interface target, `attach_type` set to `BPF_LINK_TYPE_XDP` and the same `flags` as would be used for the netlink approach.

There are some subtile differences. The netlink method will give the network interface a reference to the program, which means that after attaching, the program will stay attached until it is detached by a program, even if the original loader exists. This is in contrast to kprobes for example which will stop as soon as the loader exists (assuming we are not pinning the program). With links however, this referencing doesn't occur, the creation of the link returns a file descriptor which is used to manage the lifecycle, if the link fd is closed or the loader exists without pinning it, the program will be detached from the network interface.

!!! warning
    Hardware offloaded GRO and LSO are incompatible with XDP and have to be disabled in order to use XDP. Not doing so will result in a `-EINVAL` error upon attaching.
    The following commands can be used to disable GRO and LSO: `ethtool -K {ifname} lro off gro off`

!!! warning
    For XDP programs without fragments support there exists a max MTU of between 1500 and 4096 bytes, the exact limit depends on the driver. If the configured MTU on the device is set higher then the limit, XDP programs cannot be attached.

### Flags

#### `XDP_FLAGS_UPDATE_IF_NOEXIST`

If set, the kernel will only attach the XDP program if the network interface doesn't have a XDP program attached already.

!!! note
    This flag is only used with the netlink attach method, the link attach method handles this behavior more generically.

#### `XDP_FLAGS_SKB_MODE`

If set, the kernel will attach the program in SKB (Socket buffer) mode. This mode is also known as "Generic mode". This always works regardless of driver support. It works by calling the XDP program after a socket buffer has already been allocated further up the stack that an XDP program would normally be called. This negates the speed advantage of XDP programs. This mode also lacks full feature support since some actions cannot be taken this high up the network stack anymore. 

It is recommended to use `BPF_PROG_TYPE_SCHED_CLS` prog types instead if driver support isn't available since it offers more capabilities with roughtly the same performance.

This flag is mutually exclusive with `XDP_FLAGS_DRV_MODE` and `XDP_FLAGS_HW_MODE`

#### `XDP_FLAGS_DRV_MODE`

If set, the kernel will attach the program in driver mode. This does require support from the network driver, but most predominant network card vendors have support in the latest kernel.

This flag is mutually exclusive with `XDP_FLAGS_SKB_MODE` and `XDP_FLAGS_HW_MODE`

#### `XDP_FLAGS_HW_MODE`

If set, the kernel will attach the program in hardware offload mode. This requires both driver and hardware support for XDP offloading. Currently only select Netronome devices [support offloading](https://www.netronome.com/media/documents/eBPF_HW_OFFLOAD_HNiMne8_2_.pdf). However, it should be noted that only a subset of normal features are supported. 

#### `XDP_FLAGS_REPLACE`

If set, the kernel will atomically replace the existing program for this new program. You will also have to pass the file descriptor of the old program via the netlink request.

!!! note
    This flag is only used with the netlink attach method, the link attach method handles this behavior more generically.

### Device map program

[:octicons-tag-24: v5.8](https://github.com/torvalds/linux/commit/281920b7e0b31e0a7706433ff58e7d52ac97c327)

XDP programs can be attached to map values of a [`BPF_MAP_TYPE_DEVMAP`](../map-type/BPF_MAP_TYPE_DEVMAP.md) map. Once attached this program will run after the first program concluded but before the packet is sent of to the new network device. These programs are called with additional context, see [`egress_ifindex`](#egress_ifindex).

Only XDP programs that have been loaded with the `BPF_XDP_DEVMAP` value in [`expected_attach_type`](../syscall/BPF_PROG_LOAD.md#expected_attach_type) are allowed to be attached in this way.

Program authors can indicate to loaders like libbpf that a given program should be loaded with this expected attach type by placing the program in a `xdp/devmap/` ELF section.

### CPU map program

[:octicons-tag-24: v5.9](https://github.com/torvalds/linux/commit/9216477449f33cdbc9c9a99d49f500b7fbb81702).

XDP programs can be attached to map values of a [`BPF_MAP_TYPE_CPUMAP`](../map-type/BPF_MAP_TYPE_CPUMAP.md) map. Once attached this program will run on the new logical CPU. The idea being that you would spend minimal time in the first XDP program and only schedule it and perform the more CPU intensive tasks in this second program.

Only XDP programs that have been loaded with the `BPF_XDP_CPUMAP` value in [`expected_attach_type`](../syscall/BPF_PROG_LOAD.md#expected_attach_type) are allowed to be attached in this way.

Program authors can indicate to loaders like libbpf that a given program should be loaded with this expected attach type by placing the program in a `xdp/cpumap/` ELF section.

## Driver support

| Driver name            | Native XDP                                                                                                   | XDP HW Offload   | XDP Fragments                                                                                                                                                                                                                 | AF_XDP                                                                                                       |
| ---------------------- | ------------------------------------------------------------------------------------------------------------ | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Mellanox mlx4          | [:octicons-tag-24: v4.8](https://github.com/torvalds/linux/commit/47a38e155037f417c5740e24ccae6482aedf4b68)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Mellanox mlx5          | [:octicons-tag-24: v4.9](https://github.com/torvalds/linux/commit/86994156c736978d113e7927455d4eeeb2128b9f)  | :material-close: | [:octicons-tag-24: v5.18](https://github.com/torvalds/linux/commit/ea5d49bdae8b4c9dcdac574eef96b1bd47000c2a)[^1], [:octicons-tag-24: v6.4](https://github.com/torvalds/linux/commit/f52ac7028bec22e925c8fece4f21641eb13b4d6f) | [:octicons-tag-24: v5.3](https://github.com/torvalds/linux/commit/db05815b36cbd486c86fd002dfa81c9af6245e25)  |
| Qlogic qede            | [:octicons-tag-24: v4.10](https://github.com/torvalds/linux/commit/496e051709588f832d7a6a420f44f8642b308a87) | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Netronome nfp          | [:octicons-tag-24: v4.10](https://github.com/torvalds/linux/commit/ecd63a0217d5f1e8a92f7516f5586d1177b95de2) | :material-check: | :material-close:                                                                                                                                                                                                              | [:octicons-tag-24: v5.18](https://github.com/torvalds/linux/commit/6402528b7a0bf9869aca1f7eed43b809d57f0ae5) |
| Virtio                 | [:octicons-tag-24: v4.10](https://github.com/torvalds/linux/commit/f600b690501550b94e83e07295d9c8b9c4c39f4e) | :material-close: | [:octicons-tag-24: v6.3](https://github.com/torvalds/linux/commit/22174f79a44baf5e46faafff1d7b21363431b93a)                                                                                                                   | :material-close:                                                                                             |
| Broadcom bnxt          | [:octicons-tag-24: v4.11](https://github.com/torvalds/linux/commit/c6d30e8391b85e00eb544e6cf047ee0160ee9938) | :material-close: | [:octicons-tag-24: v5.19](https://github.com/torvalds/linux/commit/9f4b28301ce6a594a692a0abc2002d0bb912f2b7)                                                                                                                  | :material-close:                                                                                             |
| Intel ixgbe            | [:octicons-tag-24: v4.12](https://github.com/torvalds/linux/commit/9247080816297de4e31abb684939c0e53e3a8a67) | :material-close: | :material-close:                                                                                                                                                                                                              | [:octicons-tag-24: v4.20](https://github.com/torvalds/linux/commit/d0bcacd0a130974f58a56318db7a5ca6a7ba1d5a) |
| Cavium thunder (nicvf) | [:octicons-tag-24: v4.12](https://github.com/torvalds/linux/commit/05c773f52b96ef3fbc7d9bfa21caadc6247ef7a8) | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Intel i40e             | [:octicons-tag-24: v4.13](https://github.com/torvalds/linux/commit/0c8493d90b6bb0f5c4fe9217db8f7203f24c0f28) | :material-close: | [:octicons-tag-24: v6.4](https://github.com/torvalds/linux/commit/e213ced19befc09d6d6913799053b67896596cd1)                                                                                                                   | [:octicons-tag-24: v4.20](https://github.com/torvalds/linux/commit/0a714186d3c0f7c563a03537f98716457c1f5ae0) |
| Tun                    | [:octicons-tag-24: v4.14](https://github.com/torvalds/linux/commit/761876c857cb2ef8489fbee01907151da902af91) | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Netdevsim              | [:octicons-tag-24: v4.16](https://github.com/torvalds/linux/commit/31d3ad832948c75139b0e5b653912f7898a1d5d5) | :material-check: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Intel ixgbevf          | [:octicons-tag-24: v4.17](https://github.com/torvalds/linux/commit/c7aec59657b60f3a29fc7d3274ebefd698879301) | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Veth                   | [:octicons-tag-24: v4.19](https://github.com/torvalds/linux/commit/948d4f214fde43743c57aae0c708bff44f6345f2) | :material-close: | [:octicons-tag-24: v5.5 ](https://github.com/torvalds/linux/commit/7cda76d858a4e71ac4a04066c093679a12e1312c)                                                                                                                  | :material-close:                                                                                             |
| Freescale dpaa2        | [:octicons-tag-24: v5.0](https://github.com/torvalds/linux/commit/7e273a8ebdd3b83f94eb8b49fc8ee61464f47cc2)  | :material-close: | :material-close:                                                                                                                                                                                                              | [:octicons-tag-24: v6.2](https://github.com/torvalds/linux/commit/48276c08cf5d2039aad5ef92ae7057ae0946d51e)  |
| Socionext netsec       | [:octicons-tag-24: v5.3](https://github.com/torvalds/linux/commit/ba2b232108d3c2951bab02930a00f23b0cffd5af)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| TI cpsw                | [:octicons-tag-24: v5.3](https://github.com/torvalds/linux/commit/9ed4050c0d75768066a07cf66eef4f8dc9d79b52)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Solarflare efx         | [:octicons-tag-24: v5.5](https://github.com/torvalds/linux/commit/e45a4fed9d006480a5cc2312d5d4f7988a3a655e)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Intel ice              | [:octicons-tag-24: v5.5](https://github.com/torvalds/linux/commit/efc2214b6047b6f5b4ca53151eba62521b9452d6)  | :material-close: | [:octicons-tag-24: v6.3](https://github.com/torvalds/linux/commit/2fba7dc5157b6f85dbf1b8e26e63a724db1f3d79)                                                                                                                   | [:octicons-tag-24: v5.5](https://github.com/torvalds/linux/commit/2d4238f5569722197612656163d824098208519c)  |
| Marvell mvneta         | [:octicons-tag-24: v5.5](https://github.com/torvalds/linux/commit/0db51da7a8e99f0803ec3a8e25c1a66234a219cb)  | :material-close: | [:octicons-tag-24: v5.18](https://github.com/torvalds/linux/commit/e121d27083e38bfe6ca9494fbed039e69889d5c7)                                                                                                                  | :material-close:                                                                                             |
| Amazon ena             | [:octicons-tag-24: v5.6](https://github.com/torvalds/linux/commit/838c93dc5449e5d6378bae117b0a65a122cf7361)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Hyper-V netvsc         | [:octicons-tag-24: v5.6](https://github.com/torvalds/linux/commit/351e1581395fcc7fb952bbd7dda01238f69968fd)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Marvell mvpp2          | [:octicons-tag-24: v5.9](https://github.com/torvalds/linux/commit/07dd0a7aae7f72af7cec18909581c2bb570edddc)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Xen xennet             | [:octicons-tag-24: v5.9](https://github.com/torvalds/linux/commit/6c5aa6fc4defc2a0977a2c59e4710d50fa1e834c)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Intel igb              | [:octicons-tag-24: v5.10](https://github.com/torvalds/linux/commit/9cbc948b5a20c9c054d9631099c0426c16da546b) | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Freescale dpaa         | [:octicons-tag-24: v5.11](https://github.com/torvalds/linux/commit/86c0c196cbe48f844721783d9162e46bc35c0c5a) | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Intel igc              | [:octicons-tag-24: v5.13](https://github.com/torvalds/linux/commit/26575105d6ed8e2a8e43bd008fc7d98b75b90d5c) | :material-close: | :material-close:                                                                                                                                                                                                              | [:octicons-tag-24: v5.14](https://github.com/torvalds/linux/commit/fc9df2a0b520d7d439ecf464794d53e91be74b93) |
| STmicro stmmac         | [:octicons-tag-24: v5.13](https://github.com/torvalds/linux/commit/5fabb01207a2d3439a6abe1d08640de9c942945f) | :material-close: | :material-close:                                                                                                                                                                                                              | [:octicons-tag-24: v5.13](https://github.com/torvalds/linux/commit/bba2556efad66e7eaa56fece13f7708caa1187f8) |
| Freescale enetc        | [:octicons-tag-24: v5.13](https://github.com/torvalds/linux/commit/d1b15102dd16adc17fd5e4db8a485e6459f98906) | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Bond                   | [:octicons-tag-24: v5.15](https://github.com/torvalds/linux/commit/9e2ee5c7e7c35d195e2aa0692a7241d47a433d1e) | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Marvell otx2           | [:octicons-tag-24: v5.16](https://github.com/torvalds/linux/commit/06059a1a9a4a58f139352c65b02989ea6077091a) | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Microsoft mana         | [:octicons-tag-24: v5.17](https://github.com/torvalds/linux/commit/ed5356b53f070dea5dff5a01b740561cb8222199) | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Fungible fun           | [:octicons-tag-24: v5.18](https://github.com/torvalds/linux/commit/ee6373ddf3a974c4239f56931f5944fd289146e7) | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Atlantic aq            | [:octicons-tag-24: v5.19](https://github.com/torvalds/linux/commit/0d14657f40830243266f972766f1e4d00436e648) | :material-close: | [:octicons-tag-24: v5.19](https://github.com/torvalds/linux/commit/0d14657f40830243266f972766f1e4d00436e648)                                                                                                                  | :material-close:                                                                                             |
| Mediatek mtk           | [:octicons-tag-24: v6.0](https://github.com/torvalds/linux/commit/7c26c20da5d420cde55618263be4aa2f6de53056)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Freescale fec_enet     | [:octicons-tag-24: v6.2](https://github.com/torvalds/linux/commit/6d6b39f180b83dfe1e938382b68dd1e6cb51363c)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Microchip lan966x      | [:octicons-tag-24: v6.2](https://github.com/torvalds/linux/commit/6a2159be7604f5cdd7f574f4e0922f61e63c3f16)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |
| Engleder tsnep         | [:octicons-tag-24: v6.3](https://github.com/torvalds/linux/commit/f0f6460f91305fc907b6a4ba9846e1586be0a0a2)  | :material-close: | :material-close:                                                                                                                                                                                                              | [:octicons-tag-24: v6.4](https://github.com/torvalds/linux/commit/3fc2333933fdf1148b694d15db824e10449ecbc1)  |
| Google gve             | [:octicons-tag-24: v6.4](https://github.com/torvalds/linux/commit/75eaae158b1b7d8d5bde2bafc0bcf778423071d3)  | :material-close: | :material-close:                                                                                                                                                                                                              | [:octicons-tag-24: v6.4](https://github.com/torvalds/linux/commit/fd8e40321a12391e6f554cc637d0c4b6109682a9)  |
| VMware vmxnet3         | [:octicons-tag-24: v6.6](https://github.com/torvalds/linux/commit/54f00cce11786742bd11e5e68c3bf85e6dc048c9)  | :material-close: | :material-close:                                                                                                                                                                                                              | :material-close:                                                                                             |

!!! note
    This table has last been updated for Linux :octicons-tag-24: v6.7 and is subject to change in the future.

[^1]: Only the legacy RQ mode supports XDP frags, which is not the default and will require setting via ethtool.

## Helper functions

Not all helper functions are available in all program types. These are the helper calls available for XDP programs:

<!-- DO NOT EDIT MANUALLY -->
<!-- [PROG_HELPER_FUNC_REF] -->
??? abstract "Supported helper functions"
    * [bpf_perf_event_output](../helper-function/bpf_perf_event_output.md)
    * [bpf_get_smp_processor_id](../helper-function/bpf_get_smp_processor_id.md)
    * [bpf_csum_diff](../helper-function/bpf_csum_diff.md)
    * [bpf_xdp_adjust_head](../helper-function/bpf_xdp_adjust_head.md)
    * [bpf_xdp_adjust_meta](../helper-function/bpf_xdp_adjust_meta.md)
    * [bpf_redirect](../helper-function/bpf_redirect.md)
    * [bpf_redirect_map](../helper-function/bpf_redirect_map.md)
    * [bpf_xdp_adjust_tail](../helper-function/bpf_xdp_adjust_tail.md)
    * [bpf_xdp_get_buff_len](../helper-function/bpf_xdp_get_buff_len.md)
    * [bpf_xdp_load_bytes](../helper-function/bpf_xdp_load_bytes.md)
    * [bpf_xdp_store_bytes](../helper-function/bpf_xdp_store_bytes.md)
    * [bpf_fib_lookup](../helper-function/bpf_fib_lookup.md)
    * [bpf_check_mtu](../helper-function/bpf_check_mtu.md)
    * [bpf_sk_lookup_udp](../helper-function/bpf_sk_lookup_udp.md)
    * [bpf_sk_lookup_tcp](../helper-function/bpf_sk_lookup_tcp.md)
    * [bpf_sk_release](../helper-function/bpf_sk_release.md)
    * [bpf_skc_lookup_tcp](../helper-function/bpf_skc_lookup_tcp.md)
    * [bpf_tcp_check_syncookie](../helper-function/bpf_tcp_check_syncookie.md)
    * [bpf_tcp_gen_syncookie](../helper-function/bpf_tcp_gen_syncookie.md)
    * [bpf_map_lookup_elem](../helper-function/bpf_map_lookup_elem.md)
    * [bpf_map_update_elem](../helper-function/bpf_map_update_elem.md)
    * [bpf_map_delete_elem](../helper-function/bpf_map_delete_elem.md)
    * [bpf_map_push_elem](../helper-function/bpf_map_push_elem.md)
    * [bpf_map_pop_elem](../helper-function/bpf_map_pop_elem.md)
    * [bpf_map_peek_elem](../helper-function/bpf_map_peek_elem.md)
    * [bpf_map_lookup_percpu_elem](../helper-function/bpf_map_lookup_percpu_elem.md)
    * [bpf_get_prandom_u32](../helper-function/bpf_get_prandom_u32.md)
    * [bpf_get_smp_processor_id](../helper-function/bpf_get_smp_processor_id.md)
    * [bpf_get_numa_node_id](../helper-function/bpf_get_numa_node_id.md)
    * [bpf_tail_call](../helper-function/bpf_tail_call.md)
    * [bpf_ktime_get_ns](../helper-function/bpf_ktime_get_ns.md)
    * [bpf_ktime_get_boot_ns](../helper-function/bpf_ktime_get_boot_ns.md)
    * [bpf_ringbuf_output](../helper-function/bpf_ringbuf_output.md)
    * [bpf_ringbuf_reserve](../helper-function/bpf_ringbuf_reserve.md)
    * [bpf_ringbuf_submit](../helper-function/bpf_ringbuf_submit.md)
    * [bpf_ringbuf_discard](../helper-function/bpf_ringbuf_discard.md)
    * [bpf_ringbuf_query](../helper-function/bpf_ringbuf_query.md)
    * [bpf_for_each_map_elem](../helper-function/bpf_for_each_map_elem.md)
    * [bpf_loop](../helper-function/bpf_loop.md)
    * [bpf_strncmp](../helper-function/bpf_strncmp.md)
    * [bpf_spin_lock](../helper-function/bpf_spin_lock.md)
    * [bpf_spin_unlock](../helper-function/bpf_spin_unlock.md)
    * [bpf_jiffies64](../helper-function/bpf_jiffies64.md)
    * [bpf_per_cpu_ptr](../helper-function/bpf_per_cpu_ptr.md)
    * [bpf_this_cpu_ptr](../helper-function/bpf_this_cpu_ptr.md)
    * [bpf_timer_init](../helper-function/bpf_timer_init.md)
    * [bpf_timer_set_callback](../helper-function/bpf_timer_set_callback.md)
    * [bpf_timer_start](../helper-function/bpf_timer_start.md)
    * [bpf_timer_cancel](../helper-function/bpf_timer_cancel.md)
    * [bpf_trace_printk](../helper-function/bpf_trace_printk.md)
    * [bpf_get_current_task](../helper-function/bpf_get_current_task.md)
    * [bpf_get_current_task_btf](../helper-function/bpf_get_current_task_btf.md)
    * [bpf_probe_read_user](../helper-function/bpf_probe_read_user.md)
    * [bpf_probe_read_kernel](../helper-function/bpf_probe_read_kernel.md)
    * [bpf_probe_read_user_str](../helper-function/bpf_probe_read_user_str.md)
    * [bpf_probe_read_kernel_str](../helper-function/bpf_probe_read_kernel_str.md)
    * [bpf_snprintf_btf](../helper-function/bpf_snprintf_btf.md)
    * [bpf_snprintf](../helper-function/bpf_snprintf.md)
    * [bpf_task_pt_regs](../helper-function/bpf_task_pt_regs.md)
    * [bpf_trace_vprintk](../helper-function/bpf_trace_vprintk.md)
<!-- [/PROG_HELPER_FUNC_REF] -->
