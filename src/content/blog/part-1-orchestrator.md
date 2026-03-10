---
title: 'Part 1: Building a Lightning-Fast MicroVM Orchestrator'
description: 'From Sadservers to Firecracker—how we built isolated, browser-based sandboxes using Rust, Firecracker, and ephemeral storage'
pubDate: 'Mar 09 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

## Part 1: Building a Lightning-Fast MicroVM Orchestrator — From Sadservers to Firecracker

**TL;DR:** We built an orchestrator that spins up isolated microvMs in milliseconds and exposes them through a browser-based terminal. No Docker, no containers—just Firecracker, Rust, and aggressive ephemeral storage. This post covers the "why," the architecture, and how we bridge the gap between your browser and the guest kernel via WebSockets and vsock.

---

## Chapter 1: The Discovery

I stumbled into this project while solving problems on Sadservers—you know, those Linux challenge scenarios where you SSH into a preconfigured broken system and have to fix it. Fun stuff. After a few rounds, I realized: "Hey, what if I could create instant, disposable Linux environments for *any* scenario?" That's when I discovered Firecracker and KVM, and I got curious.

Firecracker is AWS's lightweight hypervisor. It's not a full virtual machine—it's a minimal hypervisor optimized for MicroVMs. Think of it as the intersection of containers (fast, isolated) and VMs (truly isolated). After playing around with it for a week, I decided to build an orchestrator around it. The goal: create sandboxes that users can access through their browser, instantly, with full isolation.

---

## Chapter 2: Why Firecracker Over Docker?

Docker is brilliant for many things, but it's fundamentally a *container*—it's just a namespace and cgroup sandwich. If you need true security boundaries, you need actual hardware isolation. Firecracker gives you:

- **True Hardware Isolation**: Each MicroVM runs in its own KVM instance. A compromised guest cannot break out to the host or to other guests. Docker containers share the kernel with the host, so a kernel exploit is game-over.
- **Minimal Overhead**: Firecracker boots in milliseconds and uses minimal RAM (we configure ours to 128 MB). Docker has more moving parts and slower startup times.
- **Reproducibility**: We control the exact kernel, the exact rootfs, the exact memory. No "works on my machine" surprises.
- **Simplicity**: No orchestration framework. No Docker daemon. Just Firecracker, a config file, and some syscalls.

For our use case—ephemeral, browser-based sandboxes—this was the right tool.

---

## Chapter 3: The Ephemeral Storage Pattern

Here's the critical insight: **every session gets its own clone of the rootfs**.

When a browser connects to our WebSocket endpoint, the orchestrator does this:

```rust
// 1. Generate a unique session ID
let session_id = Uuid::new_v4().to_string();

// 2. Clone the golden image for this session
tokio::fs::copy("../rootfs.ext4", &rootfs_path)
    .await
    .expect("Failed to copy rootfs");
```

Why? Because this approach guarantees isolation. If user A runs a command that breaks their guest, it doesn't affect user B. Moreover, it's *ephemeral*—when the browser closes, we delete their rootfs clone:

```rust
let _ = tokio::fs::remove_file(&rootfs_path).await;
```

No persistent state. No cruft. Clean. This is the pattern that makes browser-based sandboxes feasible at scale.

---

## Chapter 4: The Axum WebSocket Bridge

The orchestrator is a single-threaded async Rust server built with [Axum](https://github.com/tokio-rs/axum). Here's the flow:

1. **Browser connects** → `/ws` endpoint
2. **Generate session** → UUID, unique paths for sockets and rootfs
3. **Spawn Firecracker** → Via subprocess with `--api-sock` pointing to our Unix socket
4. **Send boot commands** → Via HTTP to the Firecracker API
5. **Enter the multiplexer loop** → Shovel bytes between browser ↔ VM

The multiplexer is the heartbeat of the system:

```rust
loop {
    tokio::select! {
        // Browser -> Firecracker (user typing)
        msg = socket.recv() => {
            if let Some(Ok(Message::Text(text))) = msg {
                // Route to either stdin or guest-agent (via vsock)
                child_stdin.write_all(text.as_bytes()).await?;
            }
        }
        
        // Firecracker -> Browser (console output)
        bytes_read = child_stdout.read(&mut buf) => {
            if let Ok(n) = bytes_read {
                socket.send(Message::Text(output_text)).await?;
            }
        }
    }
}
```

This uses `tokio::select!` to multiplex reads and writes. When the browser sends a keystroke, we write it to the VM's stdin. When the VM outputs text, we read it and send it back to the browser. It's elegant and efficient; under load, this scales because `tokio` handles thousands of concurrent sessions without threading overhead.

---

## Chapter 5: The Vsock Bridge to the Guest Agent

Here's where it gets interesting. Terminal resizing is hard in this architecture. The browser's terminal emulator (xterm.js or similar) can detect window resize events, but how do we tell the guest kernel?

We use **vsock**—a virtual socket device that Firecracker provides. It's like a Unix socket, but instead of connecting to a process on the host, it connects to a process inside the guest. Firecracker sets this up:

```rust
let _ = client
    .put("http://localhost/vsock")
    .json(&serde_json::json!({
        "guest_cid": 3,
        "uds_path": &vsock_path
    }))
    .send()
    .await;
```

Inside the guest, a lightweight Rust binary (the **guest-agent**) listens on vsock port 5000:

```rust
let listener = VsockListener::bind_with_cid_port(VMADDR_CID_ANY, 5000)
    .expect("Failed to bind vsock listener");
```

When the browser resizes, we send a JSON-encoded resize message through the vsock tunnel:

```rust
// Browser sends: {"type":"resize","cols":120,"rows":40}
// Host receives, routes to vsock

// Guest-agent receives and calls ioctl:
let ws = winsize {
    ws_row: msg.rows,
    ws_col: msg.cols,
    ws_xpixel: 0,
    ws_ypixel: 0,
};

unsafe {
    ioctl(fd, TIOCSWINSZ, &ws);
}
```

This isn't a new pattern—the Linux kernel's `ioctl(TIOCSWINSZ)` has been setting terminal size for decades. We're just routing the resize event through the vsock tunnel instead of a local Unix socket. It's a bridge between the browser world and the kernel world.

---

## Chapter 6: Practical Isolation — Networking

Each VM gets its own TAP (network interface) with a dynamically allocated subnet. When the browser connects:

```rust
let tap_id = NEXT_TAP_ID.fetch_add(1, Ordering::SeqCst);
let tap_name = setup_dynamic_tap(tap_id).await.unwrap();
let guest_mac = format!("AA:FC:00:00:00:{:02x}", (tap_id % 254) + 1);

let boot_args = format!(
    "console=ttyS0 reboot=k panic=1 root=/dev/vda fc_ip=10.200.{}.2 fc_gw=10.200.{}.1",
    tap_id, tap_id
);
```

Each guest gets its own `/24` subnet (`10.200.{id}.0/24`). With 256 possible TAP interfaces, we can support 256 concurrent VMs with full IP isolation. Want to add a firewall rule for one guest? No problem—its MAC and IP are unique.

---

## Chapter 7: Cleanup

When the session ends (browser closes or timeout), the orchestrator cleans up:

```rust
let _ = child.kill().await;                           // Kill Firecracker
let _ = tokio::fs::remove_file(&rootfs_path).await;  // Delete ephemeral rootfs
let _ = tokio::fs::remove_file(&sock_path).await;    // Delete API socket
let _ = Command::new("sudo")
    .args(&["ip", "link", "delete", &tap_name])
    .output()
    .await;                                            // Tear down TAP interface
```

No residual processes. No dangling files. No ghost TAP devices. Clean shutdown.

---

## Looking Ahead

This is part 1 of a 3-part series. In the next post, we'll dive into the Firecracker API—how we configure the VM with just HTTP requests, why the API is so clean, and how we push it to its limits. And in part 3, we'll talk about scaling this thing and the lessons we learned.

For now, the key takeaway: combining Firecracker's minimal design, Rust's async ecosystem, and some clever ephemeral storage patterns gives you a fast, isolated, browser-accessible sandbox orchestrator. No heavy frameworks required.

---

## What's Next?

- **Part 2**: The Firecracker HTTP API—configuring boot sources, drives, network interfaces, and vsock with just HTTP PUT requests
- **Part 3**: Scaling challenges, performance tuning, and lessons from production
