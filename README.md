# AVF Terminal — Custom Guest Kernel with Full Docker Support

A complete guide to building a custom Android Virtualization Framework (AVF) guest kernel with full Docker/Podman support for the Android Terminal app .

Tested on **Pixel 10 Pro XL (mustang)** running **Android 16 / CP1A.260305.018**.

---

## Background

The Android Terminal app (`com.android.virtualization.terminal`) runs a Debian VM inside crosvm via AVF. The guest kernel (`vmlinuz`) ships inside the `com.android.virt` APEX and is stored at:

```
/data/data/com.android.virtualization.terminal/files/linux/vmlinuz
```

The stock kernel is missing several configs required by Docker/Podman (`PID_NS`, `IPC_NS`, `CGROUP_DEVICE`, `CGROUP_PIDS`, `POSIX_MQUEUE`, etc.). Since this is a **guest** kernel (not the Android host kernel), you can replace it without unlocking the bootloader or rooting the device.

---

## Requirements

- Linux machine (x86_64) for building
- `repo` tool installed (`sudo apt install repo` or via PATH)
- `adb` installed and phone connected with USB debugging enabled
- ~60GB disk space, 16GB RAM recommended
- Android Terminal app installed and Debian VM set up at least once

---

## Step 1 — Clone the Kernel Source

Find your exact kernel commit from inside the Terminal VM:

```bash
uname -a
# Example output:
# Linux debian 6.12.60-android16-6-g54e1389bda83-ab14631638-4k #1 ...
#                                      ^^^^^^^^^^^^ this is your commit hash
```

Clone the matching source on your Linux machine:

```bash
mkdir android16-guest-kernel && cd android16-guest-kernel
repo init -u https://android.googlesource.com/kernel/manifest -b common-android16-6.12
repo sync -c -j$(nproc) -q
```

Verify your commit is present:

```bash
git -C common log --oneline | grep <your-commit-hash>
```

---

## Step 2 — Pull the Running Config from Inside the VM

The kernel config lives inside the running Debian VM, not on Android directly.

**Inside the Terminal app:**
```bash
zcat /proc/config.gz > /mnt/shared/Download/kernel-config.txt
```

**Then on your Linux machine:**
```bash
adb pull /sdcard/Download/kernel-config.txt original-config-from-phone.txt
```

This is used as the base config so the new kernel is as close to stock as possible, with only Docker-required options added.

---

## Step 3 — Build

Place `build-kernel.sh` (included in this repo) in the `android16-guest-kernel/` directory and run:

```bash
chmod +x build-kernel.sh
./build-kernel.sh
```

The script will:
1. Seed `.config` from your phone's running config
2. Apply all Docker-required kernel options via `scripts/config`
3. Run `olddefconfig` to resolve dependencies
4. Verify every config made it through
5. Build the kernel and output `vmlinuz` in the repo root

Build time: **20–60 minutes** depending on CPU.

---

## Step 4 — Copy Kernel to Phone

```bash
adb push vmlinuz /sdcard/Download/vmlinuz-docker
```

---

## Step 5 — Deploy Inside the VM

Open the Terminal app, then:

```bash
sudo su
# Copy via /tmp to avoid virtiofs deadlock (do NOT cp directly between two virtiofs mounts)
cp /mnt/shared/Download/vmlinuz-docker /tmp/vmlinuz-docker
cp /tmp/vmlinuz-docker /mnt/internal/linux/vmlinuz-docker

# Update vm_config.json to use the new kernel
sed -i 's|"kernel": "$PAYLOAD_DIR/vmlinuz"|"kernel": "$PAYLOAD_DIR/vmlinuz-docker"|' \
    /mnt/internal/linux/vm_config.json

# Verify
grep kernel /mnt/internal/linux/vm_config.json
```

Close the Terminal app from recents and reopen it.

---

## Step 6 — Verify

```bash
# Confirm new kernel loaded (build date should match today)
uname -a

# Run Docker's official config check — should show zero MISSING in "Generally Necessary"
curl -fsSL https://raw.githubusercontent.com/moby/moby/master/contrib/check-config.sh | sudo sh
```

---

## Step 7 — Install Docker or Podman

**Docker:**
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker droid
# Re-login or:
newgrp docker
docker run --rm hello-world
```

---

## Kernel Configs Added Over Stock

| Config | Value | Purpose |
|--------|-------|---------|
| `CONFIG_PID_NS` | y | PID namespace isolation |
| `CONFIG_IPC_NS` | y | IPC namespace isolation |
| `CONFIG_USER_NS` | y | User namespace support |
| `CONFIG_CGROUP_DEVICE` | y | Device cgroup controller |
| `CONFIG_CGROUP_PIDS` | y | PID cgroup controller |
| `CONFIG_CGROUP_PERF` | y | Perf cgroup controller |
| `CONFIG_CGROUP_HUGETLB` | y | HugeTLB cgroup controller |
| `CONFIG_BLK_DEV_THROTTLING` | y | Block I/O throttling |
| `CONFIG_POSIX_MQUEUE` | y | POSIX message queues |
| `CONFIG_BRIDGE_NETFILTER` | y | Bridge netfilter |
| `CONFIG_BRIDGE_VLAN_FILTERING` | y | VLAN filtering on bridges |
| `CONFIG_VXLAN` | m | VXLAN overlay networking |
| `CONFIG_IPVLAN` | m | IPVLAN driver |
| `CONFIG_MACVLAN` | m | MACVLAN driver |
| `CONFIG_IP_SCTP` | m | SCTP protocol |
| `CONFIG_IP_VS` | m | IPVS load balancing |
| `CONFIG_IP_VS_NFCT` | y | IPVS connection tracking |
| `CONFIG_IP_VS_PROTO_TCP` | y | IPVS TCP support |
| `CONFIG_IP_VS_PROTO_UDP` | y | IPVS UDP support |
| `CONFIG_IP_VS_RR` | m | IPVS round-robin scheduler |
| `CONFIG_NETFILTER_XT_MATCH_IPVS` | m | iptables IPVS match |
| `CONFIG_NF_TABLES` | y | nftables |
| `CONFIG_NFT_NAT` | m | nftables NAT |
| `CONFIG_NFT_MASQ` | y | nftables masquerade |
| `CONFIG_NFT_CT` | y | nftables conntrack |
| `CONFIG_NFT_FIB` | m | nftables FIB |
| `CONFIG_NFT_FIB_IPV4` | m | nftables FIB IPv4 |
| `CONFIG_NFT_FIB_IPV6` | m | nftables FIB IPv6 |
| `CONFIG_IP6_NF_NAT` | y | IPv6 NAT |
| `CONFIG_IP6_NF_TARGET_MASQUERADE` | y | IPv6 masquerade |
| `CONFIG_NETFILTER_XT_MATCH_ADDRTYPE` | y | addrtype iptables match |
| `CONFIG_SECURITY_APPARMOR` | y | AppArmor LSM |
| `CONFIG_BTRFS_FS` | m | Btrfs filesystem |
| `CONFIG_BTRFS_FS_POSIX_ACL` | y | Btrfs POSIX ACL |

---

## Important Notes

- **Never `cp` directly between two virtiofs mounts** — always stage through `/tmp` first or the terminal will deadlock and require a force reboot.
- **Never `sync` after copying** — let virtiofs flush lazily on graceful VM shutdown.
- The original kernel is preserved as `vmlinuz` and `vm_config.json` only needs a one-line change to switch back.
- This does **not** require root, unlocked bootloader, or any modification to Android itself.
- The guest kernel is architecture-independent from the host — building for `arm64` is correct regardless of the Pixel SoC generation.

---

## Reverting

To go back to the stock kernel:

```bash
sudo sed -i 's|vmlinuz-docker|vmlinuz|' /mnt/internal/linux/vm_config.json
```

Restart the Terminal app.
