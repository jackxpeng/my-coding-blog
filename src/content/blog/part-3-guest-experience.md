---
title: 'Part 3: The Guest Experience'
description: 'Building a standalone Rust guest-agent, implementing VSOCK communication, and achieving dynamic terminal resizing.'
pubDate: 'Mar 11 2026'
heroImage: '../../assets/web_clients.png'
---

## Part 3: The Guest Experience — Bridging the Gap with VSOCK

This is the final installment of our series on building a lightning-fast MicroVM orchestrator. If you haven't yet, check out [Part 1](./part-1-orchestrator) where we tackle the architecture, and [Part 2](./part-2-network-handoff) where we solve the nightmare of dynamic TAP interfaces and subnet isolation.

In this post, we're fixing the user experience.

If you connect to our orchestrator WebSocket right now using a browser-based terminal emulator like `xterm.js`, it works! You can type, command output is returned, and you have root access to a fast, isolated Alpine Linux environment.

But try maximizing your browser window. Or opening DevTools. Or resizing the pane.
The terminal goes haywire. Text wraps arbitrarily, output gets clobbered, and the interface becomes unusable.

Why? Because the browser's terminal emulator knows it was resized to 140 columns by 45 rows, but the shell running inside the Firecracker MicroVM still thinks it's a fixed 80x24 terminal.

We need to tell the guest kernel about the size changes.

---

## Enter VSOCK: The Inter-VM Socket

Firecracker supports passing messages between the host process and the guest OS using a technology called `VSOCK`. It operates remarkably like standard Unix domain sockets, but instead of files on a filesystem, VSOCK endpoints are identified by a **CID** (Context ID) and a **Port**.

The Host is always `CID 2`.
The Guest can be assigned any CID (e.g., `CID 3`).

Our goal is simple: We want a lightweight daemon running *inside* the MicroVM (the "Guest Agent") listening on a specific VSOCK port. When the user resizes their browser, the Orchestrator on the host will serialize the new dimensions into a JSON message and blast it through the VSOCK tunnel to the Guest Agent.

### Configuring Firecracker for VSOCK

Before we boot the VM via the HTTP API, we add a VSOCK device:

```rust
let vsock_path = format!("/tmp/v.sock_{}", session_id);

// Tell Firecracker to bind a host-side unix socket to the VM's VSOCK device
client.put("http://localhost/vsock")
    .json(&serde_json::json!({
        "guest_cid": 3,
        "uds_path": &vsock_path
    }))
    .send()
    .await;
```

This tells Firecracker: "Take anything written to this Unix socket file on the host (`/tmp/v.sock_id`) and route it to the VM as VSOCK traffic on CID 3."

---

## The Guest Agent

Inside the Firecracker guest, we run a tiny, independent Rust binary. We want this binary to be as small and dependency-free as possible because it lives on the `rootfs.ext4` image that gets cloned thousands of times.

Instead of Tokio, we just use standard blocking sockets (using the `vsock` crate) because this daemon has only one job.

```rust
use std::io::Read;
use vsock::{VsockAddr, VsockListener, VMADDR_CID_ANY};

fn main() {
    // Listen on any VSOCK interface, port 5000
    let listener = VsockListener::bind_with_cid_port(VMADDR_CID_ANY, 5000)
        .expect("Failed to bind vsock listener");

    println!("Guest agent listening on VSOCK port 5000...");

    for stream in listener.incoming() {
        match stream {
            Ok(mut s) => {
                let mut buf = [0; 1024];
                if let Ok(bytes_read) = s.read(&mut buf) {
                    if bytes_read > 0 {
                        // We received a JSON string like: {"cols": 120, "rows": 40}
                        let msg_str = String::from_utf8_lossy(&buf[..bytes_read]);
                        handle_resize_message(&msg_str);
                    }
                }
            }
            Err(e) => eprintln!("Stream failed: {}", e),
        }
    }
}
```

### The Magic: ioctl and TIOCSWINSZ

How do you actually resize a PTY (pseudo-terminal) in Linux?
You use an `ioctl` system call. Specifically, the `TIOCSWINSZ` (Set Window Size) command.

When our Guest Agent receives the JSON message, it extracts the rows and columns, constructs a `winsize` C struct, and fires the `ioctl` against `stdin` (File Descriptor 0).

Since the Guest Agent is spawned from the same `inittab` session as the bash shell, it shares the terminal device.

```rust
use libc::{ioctl, winsize, TIOCSWINSZ, STDIN_FILENO};

fn handle_resize_message(json: &str) {
    // Parse JSON
    if let Ok(parsed) = serde_json::from_str::<serde_json::Value>(json) {
        if let (Some(cols), Some(rows)) = (parsed["cols"].as_i32(), parsed["rows"].as_i32()) {
            
            // Construct the C struct
            let ws = winsize {
                ws_row: rows as u16,
                ws_col: cols as u16,
                ws_xpixel: 0,
                ws_ypixel: 0,
            };

            // Call into the kernel
            unsafe {
                let res = ioctl(STDIN_FILENO, TIOCSWINSZ, &ws);
                if res < 0 {
                    eprintln!("Failed to resize terminal via ioctl");
                }
            }
        }
    }
}
```

When this runs, the Linux kernel immediately updates the PTY dimensions. If you are running `htop` or `vim` inside the guest, they immediately receive a `SIGWINCH` (Window Change) signal and redraw themselves to fit the new boundaries seamlessly.

---

## Bringing It Together on the Frontend

Back on the host side, our WebSocket Orchestrator receives resizing events from the browser's `xterm.js` instance.

```javascript
// In the browser
import { Terminal } from 'xterm';
import { FitAddon } from 'xterm-addon-fit';

const term = new Terminal();
const fitAddon = new FitAddon();
term.loadAddon(fitAddon);

// When the window resizes
window.addEventListener('resize', () => {
    fitAddon.fit();
    
    // Tell the server we resized!
    websocket.send(JSON.stringify({
        type: 'resize',
        cols: term.cols,
        rows: term.rows
    }));
});
```

The Orchestrator parses that WebSocket frame, sees it's a `resize` event rather than literal keystrokes, and writes it directly to the `/tmp/v.sock_id` file. The Firecracker hypervisor picks it up, tunnels it over VSOCK to the Guest Agent, and the `ioctl` is fired.

The entire roundtrip takes less than a millisecond.

---

## Conclusion

Over the last three parts, we've walked through building a complete MicroVM orchestration system. By prioritizing lightweight components—Firecracker, Rust, Ephemeral EXT4 cloning, TAP routing, and VSOCK bridging—we've created a system capable of launching fully isolated Linux environments on-demand in mere milliseconds.

The orchestration landscape is dominated by Kubernetes and Docker, and for stateless microservices, that's absolutely correct. But when you need to hand a user a browser-based root shell where they can break the kernel or run malware, containerization simply isn't secure enough.

Firecracker enables this new paradigm. It's the engine behind AWS Lambda, and as we've seen, it's remarkably accessible for building your own specialized compute platforms.

Thanks for reading! Happy hacking.
