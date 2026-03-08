# dnstt Client Setup with SSH Mode (Multi-Tunnel Configuration)

> **Context:** This guide assumes the server is already running [dnstm](https://github.com/mosajjal/dnstm) (DNS Tunnel Manager) with one or more dnstt-server instances configured in SSH mode — meaning each tunnel's dnstt-server forwards traffic to `127.0.0.1:22` on the server. This guide covers everything needed to set up the client side from scratch for any number of tunnels.

---

## Architecture Overview

```
[Your Apps]
    │
    ▼
[SOCKS port] ← SSH SOCKS5 proxy (dnstt-socks-N.service)
    │
    ▼
[dnstt port] ← dnstt-client (dnstt-N1.service)
    │
    ▼ (DNS tunnel over UDP port 53)
[dnstt-server on remote] → 127.0.0.1:22 (SSH)
    │
    ▼
Internet
```

### Port Convention

This guide uses the following port naming convention. Adjust to whatever suits your setup, as long as the dnstt-client port and the SSH SOCKS port are different.

| Tunnel | dnstt-client port | SSH SOCKS port |
|--------|-------------------|----------------|
| 1      | 17011             | 7011           |
| 2      | 17021             | 7021           |
| N      | 170N1             | 70N1           |

The key rule: the dnstt-client **listens** on its port, and the SSH SOCKS service **connects** to that same port — so the two port numbers must not conflict with each other.

---

## Prerequisites

- Client VPS running Ubuntu/Debian
- SSH access to the remote dnstt server
- `dnstt-client` binary placed at `/opt/dnstt/dnstt-client-linux-amd64`
- The server's tunnel domains (e.g. `t1.yourdomain.com`, `t2.yourdomain.com`, ...)

Set a variable for your tunnel count to make the commands below easy to copy-paste:

```bash
N=3  # replace with your actual number of tunnels
```

---

## Step 1: Get the Correct Public Keys from the Server

> ⚠️ **Important:** Do NOT use `dnstt-server -gen-key` to retrieve keys — it overwrites the private key file on disk each time it runs. Always read the pubkey from the running service journal.

On the **server**, print all pubkeys from the running service logs:

```bash
for i in $(seq 1 $N); do
  echo "t$i: $(journalctl -u dnstm-t$i-yourdomain --no-pager | grep pubkey | tail -1 | awk '{print $NF}')"
done
```

> Replace `dnstm-t$i-yourdomain` with the actual systemd service name pattern used by your dnstm installation. Find the names with:
> ```bash
> systemctl list-units | grep dnstm
> ```

Note down each pubkey. They should also match the `server.pub` files in each tunnel's config directory:

```bash
for i in $(seq 1 $N); do
  echo "t$i: $(cat /etc/dnstm/tunnels/t$i-yourdomain/server.pub)"
done
```

If `server.pub` doesn't match the journal output, sync them:

```bash
for i in $(seq 1 $N); do
  journalctl -u dnstm-t$i-yourdomain --no-pager | grep pubkey | tail -1 | awk '{print $NF}' \
    > /etc/dnstm/tunnels/t$i-yourdomain/server.pub
done
```

---

## Step 2: Place Public Keys on the Client

On the **client**, write each pubkey collected in Step 1 to its corresponding file:

```bash
echo "PUBKEY_FOR_T1" > /opt/dnstt/pub-11.key
echo "PUBKEY_FOR_T2" > /opt/dnstt/pub-21.key
# repeat for each tunnel: pub-N1.key
```

The naming convention is `pub-N1.key` where N is the tunnel number.

---

## Step 3: Create dnstt-client Service Files

Create one service file per tunnel. The pattern is identical — only the tunnel number, domain, pubkey file, and port change.

**`/etc/systemd/system/dnstt-11.service`** (tunnel 1 example):

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

For each additional tunnel, create `dnstt-21.service`, `dnstt-31.service`, etc., replacing:
- `pub-11.key` → `pub-N1.key`
- `t1.yourdomain.com` → `tN.yourdomain.com`
- `127.0.0.1:17011` → `127.0.0.1:170N1`
- Service filename accordingly

Enable and start all:

```bash
systemctl daemon-reload
for i in $(seq 1 $N); do
  systemctl enable --now dnstt-${i}1.service
done
```

Verify sessions are established:

```bash
for i in $(seq 1 $N); do
  echo "=== dnstt-${i}1 ==="
  journalctl -u dnstt-${i}1 -n 3 --no-pager | grep -E "begin session|error"
done
```

You should see `begin session XXXXXXXX` for each tunnel. If you see repeated `opening stream: io: read/write on closed pipe` errors, the pubkey is wrong — go back to Step 1.

---

## Step 4: Create SSH Tunnel Users on the Server

On the **server**, create one SSH user per tunnel:

```bash
for i in $(seq 1 $N); do
  useradd -m -s /bin/bash tunnel$i
  mkdir -p /home/tunnel$i/.ssh
  chmod 700 /home/tunnel$i/.ssh
  chown tunnel$i:tunnel$i /home/tunnel$i/.ssh
done
```

Set a password for each (needed temporarily for the next step):

```bash
for i in $(seq 1 $N); do
  echo "Setting password for tunnel$i:"
  passwd tunnel$i
done
```

---

## Step 5: Generate SSH Key Pairs on the Client

On the **client**:

```bash
for i in $(seq 1 $N); do
  ssh-keygen -t ed25519 -f /opt/dnstt/tunnel$i.key -N ""
done

# Fix permissions — SSH refuses keys that are too open
chmod 600 /opt/dnstt/tunnel*.key
```

---

## Step 6: Copy SSH Public Keys to the Server

On the **client**, copy each key through its corresponding dnstt tunnel port. The tunnel must be active and streaming before this will work.

```bash
for i in $(seq 1 $N); do
  ssh-copy-id -i /opt/dnstt/tunnel$i.key -p 170${i}1 tunnel$i@127.0.0.1
done
```

Each command will prompt for the tunnel user's password set in Step 4. After entering it, the key is installed and passwords are no longer needed.

> **If ssh-copy-id freezes:** The tunnel session may still be warming up. Check for active streams:
> ```bash
> journalctl -u dnstt-11 -n 5 --no-pager
> ```
> Once you see `begin stream` lines the tunnel is ready. Retry the command.

> **If you get "Connection closed":** The dnstt session is broken or the pubkey is wrong. Restart the service and wait for a fresh `begin session`:
> ```bash
> systemctl restart dnstt-11.service
> journalctl -u dnstt-11 -f
> ```

Verify keys landed on the server:

```bash
for i in $(seq 1 $N); do
  echo "tunnel$i: $(cat /home/tunnel$i/.ssh/authorized_keys)"
done
```

---

## Step 7: Create SSH SOCKS Proxy Service Files

These services sit on top of the dnstt tunnels and expose a SOCKS5 proxy for your apps.

**`/etc/systemd/system/dnstt-socks-1.service`** (tunnel 1 example):

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

For each additional tunnel, create `dnstt-socks-2.service`, `dnstt-socks-3.service`, etc., replacing:
- `-D 127.0.0.1:7011` → `-D 127.0.0.1:70N1` (SOCKS output port)
- `tunnel1` → `tunnelN`
- `-p 17011` → `-p 170N1` (dnstt-client port)
- `dnstt-11.service` → `dnstt-N1.service`
- Service filename accordingly

Enable and start all:

```bash
systemctl daemon-reload
for i in $(seq 1 $N); do
  systemctl enable --now dnstt-socks-$i.service
done
```

---

## Step 8: Test Each Tunnel

```bash
for i in $(seq 1 $N); do
  echo -n "Tunnel $i (port 70${i}1): "
  curl -s --max-time 30 --socks5 127.0.0.1:70${i}1 https://api.ipify.org || echo "FAILED"
done
```

Each should return the server's public IP address.

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
Running `-gen-key` when the private key file already exists overwrites it with a newly generated key, invalidating all clients using the old pubkey. After this:
1. Get the new pubkey from the service journal (Step 1)
2. Sync `server.pub` on the server
3. Update the corresponding client pubkey file
4. Restart both the dnstt-client and dnstt-socks services on the client

---

## Quick Reference

| Task | Command |
|------|---------|
| Check all tunnel sessions | `for i in $(seq 1 $N); do journalctl -u dnstt-${i}1 -n 2 --no-pager; done` |
| Check all SOCKS services | `for i in $(seq 1 $N); do systemctl status dnstt-socks-$i.service; done` |
| Test all SOCKS proxies | `for i in $(seq 1 $N); do echo -n "t$i: "; curl -s --socks5 127.0.0.1:70${i}1 https://api.ipify.org; echo; done` |
| Get server pubkeys | `for i in $(seq 1 $N); do echo "t$i: $(journalctl -u dnstm-t$i-yourdomain --no-pager \| grep pubkey \| tail -1 \| awk '{print $NF}')"; done` |
| Restart all client services | `for i in $(seq 1 $N); do systemctl restart dnstt-${i}1.service; done` |
| Restart all SOCKS services | `for i in $(seq 1 $N); do systemctl restart dnstt-socks-$i.service; done` |

---

## Notes

- The dnstt tunnel does not use standard DNS A record lookups for tunnel traffic. It uses NS delegation, so `NXDOMAIN` on the tunnel domain's A record is expected and harmless.
- DNS tunnels are inherently slow due to the overhead of encoding data in DNS queries. This is normal.
- `ExitOnForwardFailure=yes` ensures systemd restarts the SOCKS service if the port can't bind, rather than silently running without a working proxy.
- Point your apps (e.g. x-ui outbound) at `127.0.0.1:70N1` for each tunnel N as SOCKS5 proxies.
- Why SSH mode over plain SOCKS? SSH adds encryption, proper key-based authentication, and built-in keepalives. The trade-off is one extra process per tunnel, which is negligible.
