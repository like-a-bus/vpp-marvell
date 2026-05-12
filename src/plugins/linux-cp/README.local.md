# Local fix for linux-xfrm-nl plugin (route-based IPIP mode)

This branch contains a single patch on top of Marvell's `stable/2402`
fork of VPP, fixing the `interface ipip` parser in the `linux-xfrm-nl`
plugin.

## Upstream

Forked from:
https://github.com/MarvellEmbeddedProcessors/vpp (branch `stable/2402`)

## What this fixes

The XFRM netlink plugin in Marvell's VPP fork supports two route-based
IPsec modes selected via startup.conf:
linux-xfrm-nl {
enable-route-mode-ipsec
interface ipsec     # native VPP ipsec-itf
or
interface ipip      # IPIP tunnel + ipsec_tun_protect
}

The parser in `lcp_xfrm_itf_pair_config()` only recognised
`interface ipsec`. With `interface ipip` configured, the parser
silently fell back to the ipsec-itf code path because
`nm->interface_type` remained at zero (`NL_INTERFACE_TYPE_NONE`).

Additionally, the original parser used `unformat ... "%s"` which
returns a `vec` without a null terminator, making subsequent
`clib_strcmp` comparisons unreliable.

## The patch

`src/plugins/linux-cp/lcp_xfrm_nl.c` — replaces the original
`unformat "interface %s" + clib_strcmp` block with direct unformat
literals:

```c
else if (unformat (input, "interface ipsec"))
  nm->interface_type = NL_INTERFACE_TYPE_IPSEC;
else if (unformat (input, "interface ipip"))
  nm->interface_type = NL_INTERFACE_TYPE_IPIP;
```

The unused `tunnel_name` variable was removed.

See commit `fix/linux-xfrm-nl-ipip-parser` for the diff.

## Verification

Tested in a virtualised environment (Ubuntu 24.04, Xeon Gold 6448H,
virtio-net + DPDK/uio_pci_generic) with:

- strongSwan 5.9.13 from Ubuntu packages (stock `kernel-netlink` plugin)
- IKEv2 / PSK, AES-128-CBC + HMAC-SHA1
- traffic selectors `10.10.0.0/24 <-> 10.20.0.0/24`
- WAN endpoints `10.40.0.1 <-> 10.40.0.2`

After the patch, `vppctl show ipip tunnel` correctly shows the IPIP
tunnel created by the plugin on NEWSA notification, and traffic is
encrypted via `esp4-encrypt-tun`. End-to-end iperf3 single-stream TCP
throughput: ~2.81 Gbit/s (vs ~2.58 Gbit/s with the vpp-sswan plugin in
the same setup, both bottlenecked by the hypervisor — virtio + single
vhost-net thread).

Packet trace captured on the WAN-side interface confirmed the
encrypted ESP datagram structure (outer IP 10.40.0.1->10.40.0.2,
proto=ESP, SPI matching `vppctl show ipsec sa`).

## Building

Build the same way as upstream — see Marvell's README. Only
`linux_nl_plugin.so` is affected, so a partial rebuild via ninja is
enough after pulling this patch:
cd build-root/build-vpp-native/vpp
ninja linux_nl_plugin

## Notes

* `interface ipsec` in route-mode still has separate issues with
  inner-protocol mapping (`no tunnel protocol` drops in
  `esp4-decrypt-tun`) — not addressed by this patch.
* Policy-based mode (without `enable-route-mode-ipsec`) is not tested
  here.
