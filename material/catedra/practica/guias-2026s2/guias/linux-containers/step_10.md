# Step 5.1: User Namespace Basics

## Learning Objectives

- Understand user namespace isolation
- Create a user namespace
- Map user IDs (UID) and group IDs (GID)
- Verify you appear as root inside namespace
- Understand security implications
- Map root user (UID 0) to non-root on host

## Prerequisites

- Completion of Step 4.2 (IPC Namespace)
- Understanding of Linux user/group IDs
- Root access on a Linux system (for initial setup)
- Understanding of /etc/subuid and /etc/subgid files
- `uidmap` package installed (`sudo apt-get install -y uidmap`)
- On Ubuntu 24.04: disable AppArmor user namespace restriction:
  ```bash
  sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
  ```

## Background

User namespace isolates:
- **User IDs (UIDs)**
- **Group IDs (GIDs)**
- **Capabilities** (what root can do)

Key features:
- Map UID 0 (root) in namespace to non-root UID on host
- Allows running as "root" in container without host root privileges
- Provides additional security layer
- Used by Docker for rootless containers

UID/GID mapping:
- Inside namespace: UID 0 appears as root
- On host: UID 0 maps to a high UID (e.g., 100000)
- Files created as "root" in container owned by mapped UID on host

This is a powerful security feature - containers can run as root internally while being non-root on the host system.

## Commands

> **⚠️ Environment:** On Ubuntu 23.10+ / AppArmor, unprivileged user namespaces are restricted by default.
> Before this phase, run (see also README → *Environment setup*):
> ```bash
> echo 0 | sudo tee /proc/sys/kernel/apparmor_restrict_unprivileged_userns
> ```
> Without this, `unshare --user --map-root-user` fails with `write failed /proc/self/uid_map: Operation not permitted`.

### Step 1: Check current user and UID

```bash
# Check current user
whoami
# Should show current username

# Check UID
id
# Shows UID, GID, and groups

# Check UID directly
echo $UID
# Shows numeric UID
```

### Step 2: Check subuid and subgid files

```bash
# Check if /etc/subuid exists
cat /etc/subuid 2>/dev/null || echo "File does not exist"

# Check if /etc/subgid exists
cat /etc/subgid 2>/dev/null || echo "File does not exist"

# These files define UID/GID ranges available for mapping
```

### Step 3: Create user namespace (simple)

```bash
# As regular user (not root), create user namespace
unshare --user bash

# Verify namespace
readlink /proc/self/ns/user
# Should show a user namespace ID

# Check UID
id
# May show "nobody" or mapped UID

# Try to become root
# Note: Without proper mapping, this may not work as expected
```

### Step 4: Create user namespace with mapping (requires setup)

```bash
# First, as root, set up UID/GID mapping
sudo su

# Check available UID range for your user
# Replace 'youruser' with your username
YOUR_USER=$(whoami)
echo "User: $YOUR_USER"

# Add UID mapping (if not already present)
# This allows mapping UIDs 0-65535 in namespace to 100000-165535 on host
if ! grep -q "^${YOUR_USER}:" /etc/subuid 2>/dev/null; then
    echo "${YOUR_USER}:100000:65536" >> /etc/subuid
fi

# Add GID mapping
if ! grep -q "^${YOUR_USER}:" /etc/subgid 2>/dev/null; then
    echo "${YOUR_USER}:100000:65536" >> /etc/subgid
fi

# Verify mappings
cat /etc/subuid | grep "$YOUR_USER"
cat /etc/subgid | grep "$YOUR_USER"
```

### Step 5: Create user namespace with proper mapping

```bash
# Exit root shell
exit

# As regular user, create user namespace
unshare --user --map-root-user bash

# Verify you appear as root
whoami
# Should show: root

# Check UID
id
# Should show: uid=0(root) gid=0(root)

# Check namespace
readlink /proc/self/ns/user
# Should show user namespace ID
```

### Step 6: Verify UID mapping on host

In another terminal (host):

```bash
# Find the process in user namespace
ps aux | grep bash | grep unshare
# Note the PID

# Check its UID on host
ps -o pid,uid,user -p <PID>
# Should show non-root UID (e.g., 1000) even though inside it's root

# Check /proc/<PID>/uid_map
cat /proc/<PID>/uid_map
# Should show mapping: 0 <host-uid> 1
# This maps UID 0 in namespace to <host-uid> on host
```

### Step 7: Test capabilities in user namespace

```bash
# Inside user namespace (as "root")
# Check capabilities
cat /proc/self/status | grep Cap
# Shows capabilities - may be limited

# Try to mount (may fail - user namespace has limited capabilities)
mount -t tmpfs tmpfs /tmp/test 2>&1
# May fail - user namespace root doesn't have all capabilities

# Try to change hostname (may work)
hostname testns 2>&1
# May work if combined with UTS namespace
```

### Step 8: Create file as "root" in namespace

```bash
# Inside user namespace
# Create a file as root
touch /tmp/root_file
ls -l /tmp/root_file
# Should show owned by root:root

# Exit namespace
exit
```

### Step 9: Check file ownership on host

```bash
# On host, check the file
ls -l /tmp/root_file
# Should show owned by mapped UID (e.g., 100000:100000)
# This demonstrates UID mapping
```

### Step 10: Combine user namespace with other namespaces

```bash
# Create user namespace with UTS (to test hostname)
unshare --user --uts --map-root-user bash

# Set hostname (should work now)
hostname mycontainer

# Verify
hostname
# Should show: mycontainer

# Check UID
id
# Should show: uid=0(root)
```

### Step 11: Test user namespace with mount (requires unprivileged)

```bash
# User namespace allows unprivileged users to create mount namespaces
# Create user + mount namespaces
unshare --user --mount --map-root-user bash

# Now you can mount (as "root" in namespace)
mount -t tmpfs tmpfs /tmp/test_mount

# Verify
mount | grep test_mount
# Should show the mount

# This works because user namespace + mount namespace together
# allow unprivileged mounting
```

### Step 12: Understand security implications

```bash
# Inside user namespace as "root"
# Try operations that would require real root:

# 1. Access host files (should be restricted by mount namespace)
ls /home
# May or may not work depending on mount setup

# 2. Modify host system (should fail)
echo "test" > /etc/test 2>&1
# Should fail - no access to host /etc

# 3. Network operations (may work if combined with network namespace)
ip link 2>&1
# May show limited or no interfaces
```

## Verification

To verify user namespace:

1. **User namespace ID**: `readlink /proc/self/ns/user` should be different from host
2. **Appear as root**: `whoami` should show "root" inside namespace
3. **UID mapping**: `/proc/self/uid_map` should show mapping
4. **Host UID**: Process should have non-root UID on host
5. **File ownership**: Files created as "root" mapped to high UID on host

## Common Pitfalls

1. **Missing subuid/subgid**: Need UID/GID ranges configured
   ```bash
   # Add to /etc/subuid and /etc/subgid
   echo "user:100000:65536" >> /etc/subuid
   ```

2. **Not using --map-root-user**: Without this, you may not appear as root
   ```bash
   unshare --user --map-root-user bash
   ```

3. **Limited capabilities**: User namespace root has fewer capabilities
   ```bash
   # Check capabilities
   cat /proc/self/status | grep Cap
   ```

4. **Mount requires both**: Unprivileged mounting needs user + mount namespaces
   ```bash
   unshare --user --mount --map-root-user bash
   ```

5. **File ownership confusion**: Files owned by "root" in container owned by mapped UID on host

## Understanding UID/GID Mapping

UID mapping works like this:
- **Inside namespace**: UID 0 = root
- **On host**: UID 0 maps to 100000 (or configured range start)
- **Mapping file**: `/proc/<pid>/uid_map` shows the mapping
- **Range**: Typically maps 65536 UIDs (0-65535 in namespace)

Example mapping:
```
0 100000 65536
```
This means:
- UID 0 in namespace = UID 100000 on host
- UID 1 in namespace = UID 100001 on host
- And so on...

## Security Benefits

User namespace provides:
- **Reduced privileges**: Root in container ≠ root on host
- **Attack surface reduction**: Compromised container can't affect host
- **Rootless containers**: Can run containers without host root
- **Capability dropping**: Can drop capabilities for container root

## Next Steps

In Step 5.2, we'll combine user namespace with mount namespace to properly handle file ownership in containers. We'll configure UID/GID mapping files and verify that files created as "root" in the container are properly mapped on the host.
