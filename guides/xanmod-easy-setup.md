# Ubuntu Kernel Upgrade to XanMod (AMD64v3)

This guide installs the XanMod kernel optimized for **x86-64-v3 (AMD64v3)** CPUs on Ubuntu.

---

## Requirements

- Ubuntu system with sudo privileges
- CPU supporting x86-64-v3 (AVX2, BMI1, BMI2, FMA, etc.)

To verify CPU support:

```bash
grep -m1 flags /proc/cpuinfo
```

Look for:
- avx2
- bmi1
- bmi2
- fma

---

## Installation Steps

### 1. Install required packages

```bash
sudo apt update
sudo apt install -y wget gnupg2 lsb-release software-properties-common
```

### 2. Import XanMod GPG key

```bash
wget -qO - https://dl.xanmod.org/gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/xanmod-archive-keyring.gpg
```

### 3. Add XanMod repository

```bash
echo 'deb [signed-by=/usr/share/keyrings/xanmod-archive-keyring.gpg] http://deb.xanmod.org releases main' | sudo tee /etc/apt/sources.list.d/xanmod-release.list
```

### 4. Update package list

```bash
sudo apt update
```

### 5. Install AMD64v3 optimized kernel

```bash
sudo apt install -y linux-xanmod-x64v3
```

### 6. Reboot

```bash
sudo reboot
```

---

## Verify Installation

After reboot:

```bash
uname -r
```

You should see `xanmod` in the kernel version string.

---

## Updating the Kernel

Future kernel updates will be handled automatically via:

```bash
sudo apt upgrade
```

---

## Reverting to Ubuntu Stock Kernel

If needed:

```bash
sudo apt remove linux-xanmod-x64v3
sudo update-grub
sudo reboot
```

Ensure at least one stock Ubuntu kernel remains installed before removal.

---

## Security Notes

- Keep a fallback kernel installed.
- Disable remote root login if applicable.
- Prefer SSH key authentication over password authentication.
- Test kernel stability on non-production systems first.

---

## Summary

1. Verify CPU supports x86-64-v3.
2. Add XanMod repository.
3. Install `linux-xanmod-x64v3`.
4. Reboot and verify with `uname -r`.
5. Maintain a fallback kernel.
