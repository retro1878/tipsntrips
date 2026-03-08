# Enabling Swap on Ubuntu Server

## Overview

This guide covers setting up swap on a Ubuntu server with 2GB of RAM.

## Recommended Swap Size

For a 2GB RAM server, a **2–4GB swap file** is recommended. The old "2x RAM" rule is outdated for servers — 2–4GB provides a useful safety buffer without wasting disk space.

> **Note:** Swap is slow. If you find yourself hitting it constantly, that's a sign you need more RAM, not more swap.

## Setup

### 1. Check current swap status

```bash
swapon --show
free -h
```

### 2. Create the swap file

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### 3. Make it permanent

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 4. Verify it's active

```bash
swapon --show
free -h
```

## Tuning

### Swappiness

Swappiness controls how aggressively the kernel uses swap. The default is `60`, but for servers, `10` is recommended — it keeps data in RAM as long as possible and only spills to swap under real pressure.

```bash
# Check current value
cat /proc/sys/vm/swappiness

# Set to 10 for the current session
sudo sysctl vm.swappiness=10

# Make permanent across reboots
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
```
