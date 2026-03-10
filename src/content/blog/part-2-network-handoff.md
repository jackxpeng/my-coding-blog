---
title: 'Part 2: The Network Handoff'
description: 'Deep dive into TAP devices, /30 subnets, and solving the Resource Busy error when networking Firecracker microVMs.'
pubDate: 'Mar 10 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

## Part 2: The Network Handoff — Navigating TAP Devices and Subnets

Welcome to Part 2 of our series on building a browser-based Firecracker MicroVM orchestrator! In [Part 1](./part-1-orchestrator), we discussed the overarching architecture and how we use ephemeral storage and WebSockets to spin up and interact with isolated environments.

Now, it's time to tackle the hardest part of the orchestrator: **Networking**.

If you've played with virtualization and Linux networking, you know where this is going. We need to seamlessly connect our lightweight, isolated Firecracker guests to the host's internet connection while maintaining strict isolation between guests.

---

## The Challenge: Connecting the MicroVM to the World

Firecracker gives us a microVM, but by default, it's a completely isolated island. There is no internet access. If our users want to download a package, curl an API, or do anything network-related in their browser sandbox, they will fail.

The standard solution is to use a **TAP interface**. A TAP interface is a virtual network kernel device that simulates a link-layer device. We create a TAP interface on the host, bridge it (or route it) to the internet, and configure Firecracker to use it.

Sounds simple in theory, but in practice, doing this dynamically for hundreds of concurrent microVMs introduces pain points.

---

## Dynamic TAP Creation

When a user requests a new session via WebSocket, we need to programmatically create a distinct TAP device.

```rust
use std::process::Command;

pub async fn setup_dynamic_tap(id: u8) -> Result<String, String> {
    let tap_name = format!("vmtap{}", id);
    let ip_pool_index = id as u32;

    // The host IP for this specific TAP device
    let host_ip = format!("10.200.{}.1/24", ip_pool_index);

    // 1. Create the TAP interface
    let _ = Command::new("sudo")
        .args(&["ip", "tuntap", "add", "dev", &tap_name, "mode", "tap"])
        .output()
        .await;

    // 2. Bring the interface up
    let _ = Command::new("sudo")
        .args(&["ip", "link", "set", "dev", &tap_name, "up"])
        .output()
        .await;

    // 3. Assign the host IP to the TAP interface
    let _ = Command::new("sudo")
        .args(&["ip", "addr", "add", &host_ip, "dev", &tap_name])
        .output()
        .await;

    Ok(tap_name)
}
```

Wait, `sudo ip tuntap add`?
Yes. Creating network interfaces requires root privileges (specifically, `CAP_NET_ADMIN`). In a production setting, you'd configure your orchestrator (or a helper binary) with the necessary capabilities so you aren't invoking shell `sudo` commands on every connection.

---

## The Infamous "Resource Busy" Error

If you follow the Firecracker networking tutorials, you attach the TAP device using the Firecracker HTTP API:

```rust
// The API payload
let network_config = serde_json::json!({
    "iface_id": "eth0",
    "host_dev_name": tap_name
});

client.put("http://localhost/network-interfaces/eth0")
    .json(&network_config)
    .send()
    .await;
```

This works great—until it doesn't.

If you try to spin up multiple VMs rapidly, or if a previous VM crashed and the orchestrator didn't clean up the TAP device properly, you'll hit a wall. Firecracker will reject the API request with an error indicating the network interface is invalid, or the host kernel will throw a **Resource Busy** error (`EBT_BUSY` or equivalent) when trying to bind to an already-bound or improperly cleaned TAP device.

### The Fix

The fix, as frustrating as it is, is meticulous cleanup and deterministic assignment.
Every session needs a unique ID that directly maps to a specific TAP device and subnet pool.

```rust
// E.g., Session 5 maps to vmtap5 and subnet 10.200.5.x
```

When a session ends—regardless of whether it ended cleanly or crashed—you must aggressively teardown the TAP interface.

```rust
// Cleanup routine
let _ = Command::new("sudo")
    .args(&["ip", "link", "delete", &tap_name])
    .output()
    .await;
```

If the TAP device is deleted, the kernel releases the resources, and the next session can recreate `vmtap5` without hitting the Resource Busy snag.

---

## Subnets and IP Sprawl

Notice out `/24` subnet?
For true isolation and to cleanly route traffic (e.g., via `iptables` NAT rules), every single microVM gets its own subnet.

We chose `/24` for simplicity in the prototype: `10.200.{id}.0/24`. We set `10.200.id.1` on the host side (the TAP device) and configure the guest kernel to boot with `10.200.id.2`.

```rust
// Constructing the kernel boot args
let boot_args = format!(
    "console=ttyS0 reboot=k panic=1 pci=off root=/dev/vda fc_ip=10.200.{}.2 fc_gw=10.200.{}.1",
    tap_id, tap_id
);
```

If we wanted to be more economical with IP addresses (e.g., maximizing the number of concurrent VMs on a single host), we could use `/30` subnets. A `/30` subnet gives you exactly two usable hosts: one for the TAP interface, one for the VM.

## Giving the MicroVM the Internet

By default, even with a TAP interface configured, the VM can only talk to the host. To let it reach the outside world, we need standard Linux NAT routing using `iptables` (or `nftables`).

Run this once on the host during orchestrator startup (assuming `eth0` is the host's actual internet-facing interface):

```bash
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1

# Add standard MASQUERADE NAT rule
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Allow forwarding from our TAP subnet pool
sudo iptables -A FORWARD -i vmtap+ -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o vmtap+ -m state --state RELATED,ESTABLISHED -j ACCEPT
```

These rules essentially say: "Take any traffic coming from a `vmtap` interface, rewrite the source address to look like it came from the host (`MASQUERADE`), and send it out to the internet. When the reply comes back, route it back to the correct `vmtap` interface."

With these pieces in place, the moment our user boots their WebSocket-connected sandbox, they can `ping 8.8.8.8` and get a reply.

---

## Next Steps

We have a booting MicroVM, ephemeral filesystem cloning, browser WebSocket multiplexing, and full internet access.

But the user experience isn't quite there yet. When the user resizes their browser window, the terminal emulator in the browser (`xterm.js`) resizes, but the bash shell inside the MicroVM has no idea! You'll get ugly line wrapping and display glitches.

In **Part 3**, we're crossing the boundary. We're going to build a tiny Rust daemon that runs *inside* the MicroVM guest, communicating with the host over a VSOCK tunnel to dynamically sync terminal dimensions.

[Read Part 3: The Guest Experience →](./part-3-guest-experience)
