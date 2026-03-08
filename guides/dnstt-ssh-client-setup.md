# dnstt Client Setup with SSH Mode (6-Tunnel Configuration)

> **Context:** This guide assumes the server is already running [dnstm](https://github.com/mosajjal/dnstm) (DNS Tunnel Manager) with multiple dnstt-server instances configured in SSH mode — meaning each tunnel's dnstt-server forwards traffic to `127.0.0.1:22` on the server. This guide covers everything needed to set up the client side from scratch.

---

## Architecture Overview

```
[Your Apps]
    │
    ▼
[port 70X1] ← SSH SOCKS5 proxy (dnstt-socks-N.service)
    │
    ▼
[port 170X1] ← dnstt-client (dnstt-N1.service)
    │
    ▼ (DNS tunnel over UDP port 53)
[dnstt-server on remote] → 127.0.0.1:22 (SSH)
    │
    ▼
Internet
```

**Port mapping for 6 tunnels:**

| Tunnel | dnstt-client port | SSH SOCKS port |
|--------|-------------------|----------------|
| 1      | 17011             | 7011           |
| 2      | 17021             | 7021           |
| 3      | 17031             | 7031           |
| 4      | 17041             | 7041           |
| 5      | 17051             | 7051           |
| 6      | 17061             | 7061           |

---

## Prerequisites

- Client VPS running Ubuntu/Debian
- SSH access to the remote dnstt server
- `dnstt-client` binary placed at `/opt/dnstt/dnstt-client-linux-amd64`
- The server's tunnel domains (e.g. `t1.u4o.fit` through `t6.u4o.fit`)
- `sshpass` (optional, only if using password auth instead of keys)

---

## Step 1: Get the Correct Public Keys from the Server

> ⚠️ **Important:** Do NOT use `dnstt-server -gen-key` to retrieve keys — it overwrites the private key file each time if permissions allow. Always read the pubkey from the running service journal.

On the **server**, run:

```bash
for i in 1 2 3 4 5 6; do
  echo "t$i: $(journalctl -u dnstm-t$i-u4o-fit --no-pager | grep pubkey | tail -1 | awk '{print $NF}')"
done
```

Note down all six pubkeys. They should also match `server.pub` in each tunnel's config directory:

```bash
for i in 1 2 3 4 5 6; do
  echo "t$i: $(cat /etc/dnstm/tunnels/t$i-u4o-fit/server.pub)"
done
```

If these don't match the journal output, sync them:

```bash
for i in 1 2 3 4 5 6; do
  journalctl -u dnstm-t$i-u4o-fit --no-pager | grep pubkey | tail -1 | awk '{print $NF}' \
    > /etc/dnstm/tunnels/t$i-u4o-fit/server.pub
done
```

---

## Step 2: Place Public Keys on the Client

On the **client**, write each pubkey to its corresponding file:

```bash
echo "PUBKEY_FOR_T1" > /opt/dnstt/pub-11.key
echo "PUBKEY_FOR_T2" > /opt/dnstt/pub-21.key
echo "PUBKEY_FOR_T3" > /opt/dnstt/pub-31.key
echo "PUBKEY_FOR_T4" > /opt/dnstt/pub-41.key
echo "PUBKEY_FOR_T5" > /opt/dnstt/pub-51.key
echo "PUBKEY_FOR_T6" > /opt/dnstt/pub-61.key
```

---

## Step 3: Create dnstt-client Service Files

Create one service file per tunnel. The pattern is consistent — only the tunnel number, domain, pubkey file, and port change.

**`/etc/systemd/system/dnstt-11.service`:**

```ini
[Unit]
Description=Dnstt DNS Tunnel Client
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/opt/dnstt
ExecStart=/opt/dnstt/dnstt-client-linux-amd64 -udp 8.8.8.8:53 -pubkey-file /opt/dnstt/pub-11.key t1.u4o.fit 127.0.0.1:17011
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process

[Install]
WantedBy=multi-user.target
```

Repeat for tunnels 2–6, replacing:
- `pub-11.key` → `pub-21.key`, `pub-31.key`, etc.
- `t1.u4o.fit` → `t2.u4o.fit`, `t3.u4o.fit`, etc.
- `17011` → `17021`, `17031`, etc.
- Service filename: `dnstt-21.service`, `dnstt-31.service`, etc.

Enable and start all:

```bash
systemctl daemon-reload
for i in 11 21 31 41 51 61; do
  systemctl enable --now dnstt-$i.service
done
```

Verify they're running and sessions are established:

```bash
for i in 11 21 31 41 51 61; do
  echo "=== dnstt-$i ===" 
  journalctl -u dnstt-$i -n 3 --no-pager | grep -E "begin session|error"
done
```

You should see `begin session XXXXXXXX` for each tunnel. If you see repeated `opening stream: io: read/write on closed pipe` errors, the pubkey is wrong — go back to Step 1.

---

## Step 4: Create SSH Tunnel Users on the Server

On the **server**, create one SSH user per tunnel:

```bash
for i in 1 2 3 4 5 6; do
  useradd -m -s /bin/bash tunnel$i
  mkdir -p /home/tunnel$i/.ssh
  chmod 700 /home/tunnel$i/.ssh
  chown tunnel$i:tunnel$i /home/tunnel$i/.ssh
done
```

Set passwords for each user (you'll need these in Step 6):

```bash
for i in 1 2 3 4 5 6; do
  echo "Setting password for tunnel$i:"
  passwd tunnel$i
done
```

---

## Step 5: Generate SSH Key Pairs on the Client

On the **client**:

```bash
for i in 1 2 3 4 5 6; do
  ssh-keygen -t ed25519 -f /opt/dnstt/tunnel$i.key -N ""
done
```

Fix permissions (SSH will refuse keys that are too open):

```bash
chmod 600 /opt/dnstt/tunnel*.key
```

---

## Step 6: Copy SSH Public Keys to the Server

On the **client**, copy each key through its corresponding dnstt tunnel port:

```bash
ssh-copy-id -i /opt/dnstt/tunnel1.key -p 17011 tunnel1@127.0.0.1
ssh-copy-id -i /opt/dnstt/tunnel2.key -p 17021 tunnel2@127.0.0.1
ssh-copy-id -i /opt/dnstt/tunnel3.key -p 17031 tunnel3@127.0.0.1
ssh-copy-id -i /opt/dnstt/tunnel4.key -p 17041 tunnel4@127.0.0.1
ssh-copy-id -i /opt/dnstt/tunnel5.key -p 17051 tunnel5@127.0.0.1
ssh-copy-id -i /opt/dnstt/tunnel6.key -p 17061 tunnel6@127.0.0.1
```

> **If ssh-copy-id freezes:** The tunnel session may still be warming up. Wait 15–20 seconds and check for active streams:
> ```bash
> journalctl -u dnstt-11 -n 5 --no-pager
> ```
> Once you see `begin stream` lines, the tunnel is ready. Retry the command.

> **If you get "Connection closed":** The dnstt session is broken. Restart the service and wait for a fresh `begin session`:
> ```bash
> systemctl restart dnstt-11.service
> journalctl -u dnstt-11 -f
> ```

Each `ssh-copy-id` will prompt for the tunnel user's password (set in Step 4). After entering it, the key is installed and you won't need the password again.

Verify keys landed correctly on the server:

```bash
for i in 1 2 3 4 5 6; do
  echo "tunnel$i: $(cat /home/tunnel$i/.ssh/authorized_keys)"
done
```

---

## Step 7: Create SSH SOCKS Proxy Service Files

These services sit on top of the dnstt tunnels and expose a SOCKS5 proxy for your apps.

**`/etc/systemd/system/dnstt-socks-1.service`:**

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

Repeat for tunnels 2–6, replacing:
- `7011` → `7021`, `7031`, etc. (SOCKS output port)
- `tunnel1` → `tunnel2`, `tunnel3`, etc.
- `17011` → `17021`, `17031`, etc. (dnstt-client port)
- `dnstt-11.service` → `dnstt-21.service`, etc.
- Service filename: `dnstt-socks-2.service`, etc.

Enable and start all:

```bash
systemctl daemon-reload
for i in 1 2 3 4 5 6; do
  systemctl enable --now dnstt-socks-$i.service
done
```

---

## Step 8: Test Each Tunnel

Test each SOCKS proxy by routing a curl request through it:

```bash
for i in 1 2 3 4 5 6; do
  PORT=$((7000 + $i * 10 + 1))
  echo -n "Tunnel $i (port $PORT): "
  curl -s --max-time 30 --socks5 127.0.0.1:$PORT https://api.ipify.org || echo "FAILED"
done
```

Each should return the server's public IP address. Any that return `FAILED` need troubleshooting (see below).

---

## Troubleshooting

### Tunnel connects but streams fail immediately (`read/write on closed pipe`)
The pubkey on the client doesn't match the server's private key.
- Get the correct pubkey from the server journal (Step 1)
- Update the `.key` file on the client
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
- **`Address already in use`** → another process owns the SOCKS port, choose a different one
- **`kex_exchange_identification: Connection closed`** → pubkey mismatch, fix it

### After server reboot, tunnels break
The server's `server.pub` files may be out of sync with the private keys. Re-run the sync command on the server (Step 1) and update all client pubkey files accordingly.

### dnstt-server generates a new key on every start
This happens if you ran `dnstt-server -gen-key` manually and it overwrote the private key. The service itself does not regenerate keys on restart — only the manual `-gen-key` command does. After a manual overwrite, sync `server.pub` from the journal and update the client keys.

---

## Quick Reference

| Task | Command |
|------|---------|
| Check all tunnel sessions | `for i in 11 21 31 41 51 61; do journalctl -u dnstt-$i -n 2 --no-pager; done` |
| Check all SOCKS services | `systemctl status dnstt-socks-{1,2,3,4,5,6}.service` |
| Test all SOCKS proxies | `for i in 1 2 3 4 5 6; do echo -n "t$i: "; curl -s --socks5 127.0.0.1:$((7000+$i*10+1)) https://api.ipify.org; echo; done` |
| Get server pubkeys | `for i in 1 2 3 4 5 6; do echo "t$i: $(journalctl -u dnstm-t$i-u4o-fit --no-pager \| grep pubkey \| tail -1 \| awk '{print $NF}')"; done` |
| Restart all client services | `for i in 11 21 31 41 51 61; do systemctl restart dnstt-$i.service; done` |
| Restart all SOCKS services | `for i in 1 2 3 4 5 6; do systemctl restart dnstt-socks-$i.service; done` |

---

## Notes

- The dnstt tunnel does not use standard DNS A record lookups for tunnel traffic. It uses NS delegation, so `NXDOMAIN` on the tunnel domain's A record is expected and harmless.
- DNS tunnels are inherently slow due to the overhead of encoding data in DNS queries. This is normal.
- `ExitOnForwardFailure=yes` in the SOCKS service ensures systemd restarts the service if the SOCKS port can't bind, rather than silently running without a working proxy.
- Point your apps (e.g. x-ui outbound) at `127.0.0.1:7011` through `127.0.0.1:7061` as SOCKS5 proxies.
