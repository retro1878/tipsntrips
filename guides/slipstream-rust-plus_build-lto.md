# Building slipstream-rust-plus from Source with Rust LTO

A step-by-step guide for compiling [slipstream-rust-plus](https://github.com/Fox-Fig/slipstream-rust-plus) from source using Rust's Link-Time Optimization (LTO) for maximum performance. Based on the [Fox-Fig/slipstream-rust-plus-deploy](https://github.com/Fox-Fig/slipstream-rust-plus-deploy) deployment framework.

> **Note:** Clients and servers built from this repo are only compatible with each other. Do **not** mix with the original `Mygod/slipstream-rust` binaries.

---

## Why Build with LTO?

The deploy script (`slipstream-rust-deploy.sh`) downloads prebuilt binaries by default. Building from source with LTO gives you:

- Better optimization on the QUIC/crypto hot paths
- `codegen-units = 1` enables full cross-crate inlining
- Smaller, stripped binary
- Slightly lower latency under sustained load

The tradeoff is a longer compile time (3-20 min depending on VPS specs).

---

## Prerequisites

- Ubuntu/Debian Linux server (x86_64 or ARM64)
- Root or sudo access
- At least 1 GB free RAM (2 GB+ recommended — add swap if needed)

---

## Step 1: Add Swap (if low on RAM)

LTO linking is memory-hungry. If your VPS has 1-2 GB RAM, create a swap file first:

```bash
fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
```

---

## Step 2: Install System Dependencies

```bash
apt install -y build-essential cmake pkg-config libssl-dev git
```

> Do **not** install `cargo` via `apt` — the packaged version (1.75) is outdated. Use rustup instead (next step).

---

## Step 3: Install Rust via rustup

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

When prompted, press `1` for the default installation. Then activate it in your current session:

```bash
source $HOME/.cargo/env
```

Verify:

```bash
cargo --version
```

You should see `cargo 1.8x.x` or newer.

---

## Step 4: Clone the Source

```bash
git clone https://github.com/Fox-Fig/slipstream-rust-plus.git
cd slipstream-rust-plus
git submodule update --init --recursive
```

---

## Step 5: Enable LTO in Cargo.toml

Open the workspace `Cargo.toml`:

```bash
nano /root/slipstream-rust-plus/Cargo.toml
```

Scroll to the very bottom and add:

```toml
[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
strip = "symbols"
panic = "abort"
```

Save with `Ctrl+X`, `Y`, `Enter`.

Verify it saved:

```bash
tail -10 /root/slipstream-rust-plus/Cargo.toml
```

### Alternative: Use environment variables instead (no file editing)

```bash
CARGO_PROFILE_RELEASE_LTO=fat \
CARGO_PROFILE_RELEASE_CODEGEN_UNITS=1 \
cargo build -p slipstream-client -p slipstream-server --release
```

---

## Step 6: Build

```bash
cargo build -p slipstream-client -p slipstream-server --release
```

This will take **3-20 minutes** depending on your VPS. Expected output at the end:

```
Finished `release` profile [optimized] target(s) in Xm XXs
```

---

## Step 7: Verify the Binaries

```bash
ls -lh target/release/slipstream-server target/release/slipstream-client
```

---

## Step 8: Deploy

### Option A — Used the deploy script already

If you previously ran the deploy script to set up the systemd service and config, just replace the binary and restart:

```bash
sudo cp target/release/slipstream-server /usr/local/bin/slipstream-server
sudo systemctl restart slipstream-rust-server
sudo systemctl status slipstream-rust-server
```

### Option B — Fresh install

Run the deploy script first (it sets up the systemd service, config directory, and interactive setup):

```bash
bash <(curl -Ls https://raw.githubusercontent.com/Fox-Fig/slipstream-rust-plus-deploy/master/slipstream-rust-deploy.sh)
```

Then overwrite the downloaded prebuilt binary with your LTO build:

```bash
sudo cp target/release/slipstream-server /usr/local/bin/slipstream-server
sudo systemctl restart slipstream-rust-server
```

---

## File Locations (after deploy script)

| Path | Description |
|------|-------------|
| `/usr/local/bin/slipstream-server` | Main server binary |
| `/usr/local/bin/slipstream-rust-deploy` | Management script |
| `/etc/slipstream-rust/` | Configuration directory |

---

## Troubleshooting

**`linker 'cc' not found`**
```bash
apt install -y build-essential
```

**`cargo` not found after rustup install**
```bash
source $HOME/.cargo/env
```

**`nano Cargo.toml` opens empty file**
You're in the wrong directory. Run:
```bash
cd ~/slipstream-rust-plus
nano Cargo.toml
```

**Build OOM-killed / hangs**
Add swap (Step 1) and retry. LTO with `codegen-units=1` uses significantly more memory during linking than a default release build.

**Service won't start after replacing binary**
```bash
sudo journalctl -u slipstream-rust-server -f
```

---

## References

- [slipstream-rust-plus core](https://github.com/Fox-Fig/slipstream-rust-plus)
- [slipstream-rust-plus-deploy script](https://github.com/Fox-Fig/slipstream-rust-plus-deploy)
- [Cargo profile documentation](https://doc.rust-lang.org/cargo/reference/profiles.html)
