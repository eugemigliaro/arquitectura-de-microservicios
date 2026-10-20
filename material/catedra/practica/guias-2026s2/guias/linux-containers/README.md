# Building Docker Containers: Step-by-Step Educational Exercise

This repository contains a comprehensive educational exercise that teaches how Docker creates containers by manually reproducing each step using Linux namespaces, cgroups, and other kernel features.

## Overview

Each step introduces one concept, building complexity gradually. By the end, you'll understand how containers work at the kernel level and appreciate what Docker abstracts away.

## Prerequisites

- Linux system with root access (or sudo)
- Basic Linux command-line knowledge
- Kernel support for namespaces, cgroups, and overlay filesystems (most modern Linux distributions)

## Exercise Structure

The exercises are organized into 9 phases with 21 total steps:

### Phase 1: Filesystem Isolation
- **Step 1.1** (`step_01.md`): Basic Filesystem Isolation with `chroot`
- **Step 1.2** (`step_02.md`): Mount Namespace Isolation
- **Step 1.3** (`step_03.md`): Pivot Root

### Phase 2: Process Isolation
- **Step 2.1** (`step_04.md`): PID Namespace
- **Step 2.2** (`step_05.md`): Combining PID and Mount Namespaces

### Phase 3: Network Isolation
- **Step 3.1** (`step_06.md`): Network Namespace Basics
- **Step 3.2** (`step_07.md`): Bridge Networking

### Phase 4: System Identity Isolation
- **Step 4.1** (`step_08.md`): UTS Namespace
- **Step 4.2** (`step_09.md`): IPC Namespace

### Phase 5: User Isolation
- **Step 5.1** (`step_10.md`): User Namespace Basics
- **Step 5.2** (`step_11.md`): User Namespace with Filesystem

### Phase 6: Resource Limits (Cgroups)
- **Step 6.1** (`step_12.md`): Cgroup v2 Basics
- **Step 6.2** (`step_13.md`): Memory Limits
- **Step 6.3** (`step_14.md`): CPU Limits
- **Step 6.4** (`step_15.md`): I/O Limits

### Phase 7: Security Hardening
- **Step 7.1** (`step_16.md`): Linux Capabilities
- **Step 7.2** (`step_17.md`): Seccomp Filters

### Phase 8: Advanced Topics
- **Step 8.1** (`step_18.md`): Overlay Filesystems
- **Step 8.2** (`step_19.md`): Container Runtime Interface

### Phase 9: Putting It All Together
- **Step 9.1** (`step_20.md`): Complete Container Script
- **Step 9.2** (`step_21.md`): Container Lifecycle Management

## How to Execute the Exercises

### General Instructions

1. **Read the step file**: Each step is documented in `step_XX.md` files
2. **Follow sequentially**: Steps build on previous concepts, so complete them in order
3. **Use root access**: Most operations require root privileges (`sudo su` or `sudo`)
4. **Test each step**: Verify the isolation/feature works before moving to the next step
5. **Clean up**: Remove test containers, namespaces, and cgroups after each step

### Step-by-Step Execution

#### For Each Step:

1. **Open the step file**:
   ```bash
   cat step_01.md  # or use your preferred editor
   ```

2. **Read the Learning Objectives**: Understand what you'll learn

3. **Check Prerequisites**: Ensure you've completed previous steps

4. **Follow the Commands**: Execute commands in the order presented
   - Copy and paste commands one at a time
   - Understand what each command does
   - Don't skip verification steps

5. **Verify the Results**: Use the verification section to confirm it worked

6. **Understand Common Pitfalls**: Read this section to avoid mistakes

7. **Move to Next Step**: Once verified, proceed to the next step

### Quick Start Example

Here's how to execute Step 1.1:

```bash
# 1. Read the step
cat step_01.md

# 2. Switch to root
sudo su

# 3. Follow the commands from the step file
# Create container root directory
mkdir -p /tmp/container_root
cd /tmp/container_root
mkdir -p bin sbin etc proc sys dev usr/bin usr/lib lib lib64

# Copy essential binaries (as shown in step_01.md)
cp /bin/bash /tmp/container_root/bin/
# ... (continue with all commands from the step)

# 4. Verify
chroot /tmp/container_root /bin/bash
# Inside chroot, verify isolation works

# 5. Clean up
exit
rm -rf /tmp/container_root
```

### Important Notes

#### Root Access
Most exercises require root privileges. Use:
```bash
sudo su
```
Or prefix commands with `sudo`:
```bash
sudo unshare --pid --fork bash
```

#### System Requirements
- **Kernel**: Modern Linux kernel (4.0+) with namespace and cgroup support
- **cgroup v2**: Unified hierarchy at `/sys/fs/cgroup` (`stat -fc %T /sys/fs/cgroup` → `cgroup2fs`)
- **Tools**: Basic utilities (`ip`, `mount`, `unshare`) are usually pre-installed
- **Optional tools**: Some steps use `capsh`, `runc`, `python3`, `gcc` - install if needed:
  ```bash
  # On Debian/Ubuntu
  sudo apt-get update
  sudo apt-get install -y libcap2-bin runc python3 gcc iproute2 bridge-utils uidmap
  ```

#### Environment setup (do this before starting)

Run these once on the Linux host/VM where you execute the exercises. Without them, several steps fail on modern Ubuntu and similar distros.

```bash
# 1) AppArmor: allow unprivileged user namespaces (Phase 5)
# Without this, `unshare --user --map-root-user` fails with:
#   write failed /proc/self/uid_map: Operation not permitted
if [ -f /proc/sys/kernel/apparmor_restrict_unprivileged_userns ]; then
  echo 0 | sudo tee /proc/sys/kernel/apparmor_restrict_unprivileged_userns
fi

# 2) Bridge forwarding: if inter-container pings fail on a bridge (Phase 3)
echo 0 | sudo tee /proc/sys/net/bridge/bridge-nf-call-iptables 2>/dev/null || true
sudo iptables -P FORWARD ACCEPT 2>/dev/null || true

# 3) IP forwarding (needed for NAT / external connectivity in Phase 3)
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

**Notes on the environment knobs above:**
- Setting (1) is **session/boot volatile** unless you also add a sysctl drop-in (e.g. `/etc/sysctl.d/99-userns.conf` with `kernel.apparmor_restrict_unprivileged_userns=0`).
- Lima / cloud VMs are fine; prefer a real Linux guest (not Docker-in-Docker) so OverlayFS and cgroup controllers work.
- On **aarch64**, dynamic linker paths differ from x86_64 (`/lib/ld-linux-aarch64.so.1`); the step scripts copy whichever linker exists.

#### Cleanup Between Steps
Always clean up test resources:
```bash
# Remove test containers
rm -rf /tmp/container_* /tmp/containers /tmp/overlay /tmp/oci-bundle

# Remove network namespaces
ip netns list | awk '{print $1}' | xargs -r -I {} ip netns delete {}

# Remove leftover bridges / veths (Phase 3)
ip link del br0 2>/dev/null || true

# Remove cgroups (must be empty / leaf)
for d in /sys/fs/cgroup/mycontainer /sys/fs/cgroup/memtest /sys/fs/cgroup/cputest /sys/fs/cgroup/iotest /sys/fs/cgroup/test*; do
  [ -d "$d" ] || continue
  find "$d" -type d | sort -r | while read sub; do
    for p in $(cat "$sub/cgroup.procs" 2>/dev/null); do kill -9 "$p" 2>/dev/null || true; done
    rmdir "$sub" 2>/dev/null || true
  done
done

# Unmount test mounts
umount /tmp/test_* /tmp/overlay/merged* 2>/dev/null || true
```

### Troubleshooting

#### Common Issues

1. **"Operation not permitted"**
   - Ensure you're running as root (or completed the AppArmor userns setup above for Phase 5)
   - Check kernel support for the feature

2. **"No such file or directory"**
   - Verify paths exist
   - Check if you're in the correct directory

3. **"Device or resource busy"**
   - Unmount before removing directories (`umount -l` if needed)
   - Kill processes before removing namespaces/cgroups
   - **cgroup v2**: you cannot put processes in a cgroup that has controllers enabled in `cgroup.subtree_control` (non-leaf). Put processes in a **child** leaf cgroup instead

4. **Namespace or cgroup already exists**
   - Clean up previous test runs
   - Use unique names for test containers

5. **`capsh: unrecognized option --add=`**
   - Modern `libcap` / `capsh` no longer supports `--add=`. Use `--caps='cap_xxx=ep'` (see Step 7.1)

6. **Port mapping `curl localhost:8080` hangs**
   - `PREROUTING` DNAT does not apply to locally originated traffic; also add an `OUTPUT` DNAT rule (see Step 3.2)

#### Getting Help

- Read the "Common Pitfalls" section in each step
- Check the "Verification" section to ensure setup is correct
- Review previous steps if concepts are unclear

### Execution Order

Execute steps in this order:

1. **Phase 1** (Steps 1-3): Filesystem isolation
2. **Phase 2** (Steps 4-5): Process isolation
3. **Phase 3** (Steps 6-7): Network isolation
4. **Phase 4** (Steps 8-9): System identity isolation
5. **Phase 5** (Steps 10-11): User isolation
6. **Phase 6** (Steps 12-15): Resource limits
7. **Phase 7** (Steps 16-17): Security hardening
8. **Phase 8** (Steps 18-19): Advanced topics
9. **Phase 9** (Steps 20-21): Complete integration

### Expected Outcomes

After completing all exercises, you will:

- ✅ Understand how Linux namespaces provide isolation
- ✅ Know how cgroups limit resources
- ✅ Understand the relationship between different isolation mechanisms
- ✅ Be able to manually create a container-like environment
- ✅ Appreciate what Docker abstracts away
- ✅ Debug container-related issues more effectively

### Additional Resources

- **Kernel Documentation**: `/usr/src/linux/Documentation/` (if kernel sources installed)
- **man pages**: `man namespaces`, `man cgroups`, `man unshare`
- **Online**: Linux kernel documentation on namespaces and cgroups

## Notes

- All commands are executable on a standard Linux system
- Focus is on educational clarity over performance
- Each step includes troubleshooting tips
- Verification commands confirm each step works
- Complexity builds gradually - each step is understandable independently

## License

This educational material is provided for learning purposes.
