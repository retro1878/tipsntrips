# dnstt Client Setup with SSH Mode (Multi-Tunnel Configuration)

> **Context:** This guide assumes the server is already running [dnstm](https://github.com/mosajjal/dnstm) (DNS Tunnel Manager) with one or more dnstt-server instances configured in SSH mode — meaning each tunnel's dnstt-server forwards traffic to `127.0.0.1:22` on the server. This guide covers everything needed to set up the client side from scratch.

> **Examples in this guide use 3 tunnels.** Extend or reduce all numbered lists, loops, and port ranges to match your actual tunnel count.

---

## Architecture Overview

```
[Your Apps]
    │
    ▼
[SOCKS port] ← SSH SOCKS5 proxy (dnstt-socks-N.service)  — apps connect here
    │
    ▼
[dnstt port] ← dnstt-client (dnstt-N1.service)           — SSH connects here
    │
    ▼ (DNS tunnel over UDP port 53)
[dnstt-server on remote] → 127.0.0.1:22 (SSH)
    │
    ▼
Internet
```

### Port Convention

The examples in this guide follow this port layout. You can use any free ports — just keep the dnstt-client port and the SOCKS port separate from each other.

| Tunnel | dnstt-client listens on | SOCKS5 apps connect to |
|--------|-------------------------|------------------------|
| 1      | 17011                   | 7011                   |
| 2      | 17021                   | 7021                   |
| 3      | 17031                   | 7031                   |
| N      | 170N1                   | 70N1                   |

---

## Prerequisites

- Client VPS running Ubuntu/Debian
- SSH access to the remote dnstt server
- `dnstt-client` binary placed at `/opt/dnstt/dnstt-client-linux-amd64`
- The server's tunnel domains (e.g. `t1.yourdomain.com`, `t2.yourdomain.com`, ...)

---

## Step 1: Get the Correct Public Keys from the Server

> ⚠️ **Important:** Do NOT use `dnstt-server -gen-key` to retrieve keys — it overwrites the private key file on disk each time it runs. Always read the pubkey from the running service journal.

On the **server**, print the pubkey from each running tunnel service:

```bash
# e.g. for 3 tunnels — add more lines for additional tunnels
journalctl -u dnstm-t1-yourdomain --no-pager | grep pubkey | tail -1
journalctl -u dnstm-t2-yourdomain --no-pager | grep pubkey | tail -1
journalctl -u dnstm-t3-yourdomain --no-pager | grep pubkey | tail -1
```

> Replace `dnstm-t1-yourdomain` with the actual systemd service names used by your dnstm installation. Find them with:
> ```bash
> systemctl list-units | grep dnstm
> ```

Note down each pubkey. They should also match the `server.pub` files on disk:

```bash
cat /etc/dnstm/tunnels/t1-yourdomain/server.pub
cat /etc/dnstm/tunnels/t2-yourdomain/server.pub
cat /etc/dnstm/tunnels/t3-yourdomain/server.pub
```

If `server.pub` doesn't match the journal output for any tunnel, sync it:

```bash
# e.g. for tunnel 2
journalctl -u dnstm-t2-yourdomain --no-pager | grep pubkey | tail -1 | awk '{print $NF}' \
  > /etc/dnstm/tunnels/t2-yourdomain/server.pub
# repeat for any other out-of-sync tunnels
```

---

## Step 2: Place Public Keys on the Client

On the **client**, write each pubkey collected in Step 1 to its corresponding file:

```bash
# e.g. for 3 tunnels — add more lines for additional tunnels
echo "PUBKEY_FOR_T1" > /opt/dnstt/pub-11.key
echo "PUBKEY_FOR_T2" > /opt/dnstt/pub-21.key
echo "PUBKEY_FOR_T3" > /opt/dnstt/pub-31.key
```

The naming convention is `pub-N1.key` where N is the tunnel number.

---

## Step 3: Create dnstt-client Service Files

Create one service file per tunnel. The pattern is identical across all tunnels — only the tunnel number, domain, pubkey file, and port change.

**`/etc/systemd/system/dnstt-11.service`** (tunnel 1):

```ini
[Unit]
Description=Dnstt DNS Tunnel Client - Tunnel 1
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/opt/dnstt
ExecStart=/opt/dnstt/dnstt-client-linux-amd64 -udp 8.8.8.8:53 -pubkey-file /opt/dnstt/pub-11.key t1.yourdomain.com 127.0.0.1:17011
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process

[Install]
WantedBy=multi-user.target
```

**`/etc/systemd/system/dnstt-21.service`** (tunnel 2 — same pattern, incremented):

```ini
[Unit]
Description=Dnstt DNS Tunnel Client - Tunnel 2
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/opt/dnstt
ExecStart=/opt/dnstt/dnstt-client-linux-amd64 -udp 8.8.8.8:53 -pubkey-file /opt/dnstt/pub-21.key t2.yourdomain.com 127.0.0.1:17021
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process

[Install]
WantedBy=multi-user.target
```

> Continue the same pattern for tunnel 3 (`dnstt-31.service`), tunnel 4 (`dnstt-41.service`), and so on — incrementing the tunnel number, domain, pubkey file, and port each time.

Enable and start all client services:

```bash
# e.g. for 3 tunnels
systemctl daemon-reload
systemctl enable --now dnstt-11.service
systemctl enable --now dnstt-21.service
systemctl enable --now dnstt-31.service
```

Verify sessions are established:

```bash
journalctl -u dnstt-11 -n 3 --no-pager | grep -E "begin session|error"
journalctl -u dnstt-21 -n 3 --no-pager | grep -E "begin session|error"
journalctl -u dnstt-31 -n 3 --no-pager | grep -E "begin session|error"
```

You should see `begin session XXXXXXXX` for each tunnel. If you see repeated `opening stream: io: read/write on closed pipe` errors, the pubkey is wrong — go back to Step 1.

---

## Step 4: Create SSH Tunnel Users on the Server

On the **server**, create one SSH user per tunnel:

```bash
# e.g. for 3 tunnels — repeat for additional tunnels
for i in 1 2 3; do
  useradd -m -s /bin/bash tunnel$i
  mkdir -p /home/tunnel$i/.ssh
  chmod 700 /home/tunnel$i/.ssh
  chown tunnel$i:tunnel$i /home/tunnel$i/.ssh
done
```

Set a password for each user (needed temporarily in Step 6):

```bash
passwd tunnel1
passwd tunnel2
passwd tunnel3
```

---

## Step 5: Generate SSH Key Pairs on the Client

On the **client**:

```bash
# e.g. for 3 tunnels
ssh-keygen -t ed25519 -f /opt/dnstt/tunnel1.key -N ""
ssh-keygen -t ed25519 -f /opt/dnstt/tunnel2.key -N ""
ssh-keygen -t ed25519 -f /opt/dnstt/tunnel3.key -N ""

# Fix permissions — SSH refuses keys that are too open
chmod 600 /opt/dnstt/tunnel*.key
```

---

## Step 6: Copy SSH Public Keys to the Server

On the **client**, copy each key through its corresponding dnstt tunnel port. The tunnel must be active and streaming before this will work.

```bash
# e.g. for 3 tunnels — the port (17011, 17021, 17031) is the dnstt-client port from Step 3
ssh-copy-id -i /opt/dnstt/tunnel1.key -p 17011 tunnel1@127.0.0.1
ssh-copy-id -i /opt/dnstt/tunnel2.key -p 17021 tunnel2@127.0.0.1
ssh-copy-id -i /opt/dnstt/tunnel3.key -p 17031 tunnel3@127.0.0.1
```

Each command will prompt for the tunnel user's password set in Step 4. After entering it, the key is installed and passwords are no longer needed.

> **If ssh-copy-id freezes:** The tunnel session may still be warming up. Check for active streams:
> ```bash
> journalctl -u dnstt-11 -n 5 --no-pager
> ```
> Once you see `begin stream` lines, the tunnel is ready. Retry the command.

> **If you get "Connection closed":** The dnstt session is broken or the pubkey is wrong. Restart the service and wait for a fresh `begin session`:
> ```bash
> systemctl restart dnstt-11.service
> journalctl -u dnstt-11 -f
> ```

Verify keys landed on the server:

```bash
cat /home/tunnel1/.ssh/authorized_keys
cat /home/tunnel2/.ssh/authorized_keys
cat /home/tunnel3/.ssh/authorized_keys
```

---

## Step 7: Create SSH SOCKS Proxy Service Files

These services sit on top of the dnstt tunnels and expose a SOCKS5 proxy for your apps.

**`/etc/systemd/system/dnstt-socks-1.service`** (tunnel 1):

```ini
[Unit]
Description=SSH SOCKS Proxy over Dnstt Tunnel 1
After=network-online.target dnstt-11.service
Requires=dnstt-11.service

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/bin/ssh -D 127.0.0.1:7011 -N -i /opt/dnstt/tunnel1.key \
  -o StrictHostKeyChecking=no \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -o ExitOnForwardFailure=yes \
  tunnel1@127.0.0.1 -p 17011
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**`/etc/systemd/system/dnstt-socks-2.service`** (tunnel 2 — same pattern, incremented):

```ini
[Unit]
Description=SSH SOCKS Proxy over Dnstt Tunnel 2
After=network-online.target dnstt-21.service
Requires=dnstt-21.service

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/bin/ssh -D 127.0.0.1:7021 -N -i /opt/dnstt/tunnel2.key \
  -o StrictHostKeyChecking=no \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -o ExitOnForwardFailure=yes \
  tunnel2@127.0.0.1 -p 17021
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> Continue the same pattern for tunnel 3 (`dnstt-socks-3.service`) and beyond — incrementing the SOCKS port (`7031`, `7041`, ...), the SSH key, the tunnel user, the dnstt-client port (`17031`, `17041`, ...), and the `Requires` service name each time.

Enable and start all SOCKS services:

```bash
# e.g. for 3 tunnels
systemctl daemon-reload
systemctl enable --now dnstt-socks-1.service
systemctl enable --now dnstt-socks-2.service
systemctl enable --now dnstt-socks-3.service
```

---

## Step 8: Test Each Tunnel

```bash
# e.g. for 3 tunnels — each should return the server's public IP
curl -s --max-time 30 --socks5 127.0.0.1:7011 https://api.ipify.org
curl -s --max-time 30 --socks5 127.0.0.1:7021 https://api.ipify.org
curl -s --max-time 30 --socks5 127.0.0.1:7031 https://api.ipify.org
```

---

## Troubleshooting

### Tunnel connects but streams fail immediately (`read/write on closed pipe`)
The pubkey on the client doesn't match the server's private key.
- Get the correct pubkey from the server journal (Step 1)
- Update the corresponding `pub-N1.key` file on the client
- Restart the dnstt client service

### `ssh-copy-id` returns "Connection closed"
The dnstt tunnel session is broken or not yet established.
- Check `journalctl -u dnstt-11 -n 10 --no-pager` for errors
- Restart the service and wait for `begin session`

### SSH SOCKS service crash-loops with `status=255/EXCEPTION`
Run the SSH command manually to see the real error:
```bash
ssh -v -D 127.0.0.1:7011 -N -i /opt/dnstt/tunnel1.key \
  -o StrictHostKeyChecking=no tunnel1@127.0.0.1 -p 17011
```

Common causes:
- **`bad permissions`** on the key file → `chmod 600 /opt/dnstt/tunnel1.key`
- **`Address already in use`** → another process owns the SOCKS port; choose a different port range
- **`kex_exchange_identification: Connection closed`** → pubkey mismatch, go back to Step 1

### After server reboot, tunnels break
The server's `server.pub` files may be out of sync with the actual private keys in use. Re-run the sync command in Step 1 and update all client pubkey files accordingly.

### `dnstt-server -gen-key` was run manually and broke the keys
Running `-gen-key` when the private key file already exists overwrites it, invalidating all clients using the old pubkey. After this:
1. Get the new pubkey from the service journal (Step 1)
2. Sync `server.pub` on the server
3. Update the corresponding client pubkey file
4. Restart both the dnstt-client and dnstt-socks services on the client

---

## Quick Reference

| Task | Command (example for tunnel 1 — repeat for others) |
|------|-----------------------------------------------------|
| Check tunnel session | `journalctl -u dnstt-11 -n 5 --no-pager` |
| Check SOCKS service | `systemctl status dnstt-socks-1.service` |
| Test SOCKS proxy | `curl -s --socks5 127.0.0.1:7011 https://api.ipify.org` |
| Get server pubkey | `journalctl -u dnstm-t1-yourdomain --no-pager \| grep pubkey \| tail -1` |
| Restart tunnel client | `systemctl restart dnstt-11.service` |
| Restart SOCKS service | `systemctl restart dnstt-socks-1.service` |

---

## Notes

- The dnstt tunnel does not use standard DNS A record lookups for tunnel traffic. It uses NS delegation, so `NXDOMAIN` on the tunnel domain's A record is expected and harmless.
- DNS tunnels are inherently slow due to the overhead of encoding data in DNS queries. This is normal.
- `ExitOnForwardFailure=yes` ensures systemd restarts the SOCKS service if the port can't bind, rather than silently running without a working proxy.
- Point your apps (e.g. x-ui outbound) at `127.0.0.1:7011`, `127.0.0.1:7021`, etc. as SOCKS5 proxies — one port per tunnel.
- Why SSH mode over plain SOCKS? SSH adds encryption, proper key-based authentication, and built-in keepalives. The trade-off is one extra process per tunnel, which is negligible.
