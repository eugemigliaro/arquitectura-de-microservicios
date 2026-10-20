# Step 4.1: UTS Namespace

## Learning Objectives

- Understand UTS namespace isolation
- Create a UTS namespace
- Set a different hostname in the namespace
- Verify hostname isolation
- Understand why hostname isolation matters for containers

## Prerequisites

- Completion of Step 2.2 (Combining PID and Mount Namespaces)
- Basic understanding of hostnames
- Root access on a Linux system

## Background

### What is a UTS Namespace?

UTS (Unix Timesharing System) namespace isolates system identification information:
- **Hostname**: The system hostname (what `hostname` and `uname -n` show)
- **Domain name**: The NIS domain name (what `domainname` shows)

### How UTS Namespaces Provide Hostname Isolation

UTS namespaces create isolated system identification:

1. **Hostname Isolation**: Each namespace can have its own hostname:
   - Container can set hostname to "web-server-1"
   - Host hostname remains unchanged
   - Multiple containers can have different hostnames
   - Hostname changes in container don't affect host

2. **Domain Name Isolation**: Each namespace can have its own domain name:
   - Used for NIS (Network Information Service)
   - Less commonly used in modern systems
   - Still isolated per namespace

3. **Process Inheritance**: Child processes inherit the UTS namespace:
   - All processes in a container see the same hostname
   - Hostname is consistent across container processes
   - Applications can rely on hostname for identification

### Why UTS Namespace Matters

UTS namespace is one of the simplest namespaces - it only isolates two strings (hostname and domain name). However, it's important for proper containerization because:

- **Application Identification**: Many applications use hostname for identification
- **Service Discovery**: Hostname used in service discovery mechanisms
- **Logging**: Hostname appears in logs, making it easier to identify source
- **Security**: Hostname can be used in security policies
- **Multi-tenancy**: Multiple containers can run with different hostnames without conflict

### UTS Namespace Isolation Benefits

UTS namespace provides:
- **Hostname Isolation**: Each container has its own hostname
- **No Host Impact**: Changing hostname in container doesn't affect host
- **Application Compatibility**: Applications that depend on hostname work correctly
- **Multi-container Support**: Multiple containers can have different hostnames
- **Security**: Prevents hostname-based information leakage

This is why Docker allows setting container hostnames - it uses UTS namespaces to isolate hostname per container.

## Key Commands Explained

Before diving into the practical steps, let's understand the key commands we'll be using:

### `hostname` - Display or Set Hostname

The `hostname` command displays or sets the system hostname.

**Syntax**: `hostname [NEW_HOSTNAME]`

**How it works**:
- Without arguments: Displays current hostname
- With argument: Sets new hostname (requires appropriate privileges)
- Changes are namespace-specific (in UTS namespace)
- Hostname is stored in `/proc/sys/kernel/hostname`

**Example**:
```bash
# Display current hostname
hostname
# Output: myhost

# Set new hostname
hostname mycontainer
# (Requires root or UTS namespace)

# Verify change
hostname
# Output: mycontainer
```

**In UTS namespace context**:
- Setting hostname in namespace only affects that namespace
- Host hostname remains unchanged
- All processes in namespace see the new hostname
- Changes persist for the lifetime of the namespace

### `uname -n` - Display Hostname

The `uname` command displays system information. The `-n` option shows the hostname.

**Syntax**: `uname -n`

**How it works**:
- Reads hostname from kernel (same source as `hostname`)
- Shows the nodename (hostname)
- Returns the same value as `hostname` command

**Example**:
```bash
# Display hostname
uname -n
# Output: mycontainer

# Compare with hostname
hostname
# Output: mycontainer (same value)
```

**Why both commands exist**:
- `hostname` is more commonly used
- `uname -n` is POSIX-compliant
- Both read from `/proc/sys/kernel/hostname`
- Both show namespace-specific hostname

### `/proc/sys/kernel/hostname` - Hostname File

The `/proc/sys/kernel/hostname` file contains the current hostname.

**How it works**:
- Read-only file (read hostname)
- Write to change hostname (requires privileges)
- Namespace-specific (each UTS namespace has its own)

**Example**:
```bash
# Read hostname
cat /proc/sys/kernel/hostname
# Output: mycontainer

# Set hostname (alternative to hostname command)
echo "newhostname" > /proc/sys/kernel/hostname
# (Requires root or UTS namespace)
```

**In UTS namespace context**:
- Each namespace has its own `/proc/sys/kernel/hostname`
- Reading shows namespace-specific hostname
- Writing changes namespace-specific hostname
- Host namespace is unaffected

## Commands

### Step 1: Check current hostname

```bash
# Check host hostname
hostname
# Should show current hostname

# Check hostname using uname
uname -n
# Should show same hostname

# Check /etc/hostname (if it exists)
cat /etc/hostname 2>/dev/null || echo "File does not exist"
```

### Step 2: Create a UTS namespace

```bash
# Ensure root access
sudo su

# Create a UTS namespace
unshare --uts bash

# Verify you're in a new UTS namespace
readlink /proc/self/ns/uts
# Note this ID - compare with host namespace
```

### Step 3: Change hostname in the namespace

```bash
# Set a new hostname
hostname mycontainer

# Verify hostname changed
hostname
# Should show: mycontainer

# Verify using uname
uname -n
# Should show: mycontainer

# Check /proc/sys/kernel/hostname
cat /proc/sys/kernel/hostname
# Should show: mycontainer
```

### Step 4: Verify isolation from host

In another terminal (host namespace):

```bash
# Check host hostname
hostname
# Should still show original hostname (not "mycontainer")

# Verify UTS namespace is different
readlink /proc/self/ns/uts
# Should be different from container namespace
```

### Step 5: Test hostname in different namespaces

```bash
# In the UTS namespace, create multiple processes
hostname container1
echo "Hostname in namespace: $(hostname)"

# Start a background process
(sleep 10 &)
# Child processes inherit the UTS namespace
```

### Step 6: Combine UTS with other namespaces

```bash
# Exit current namespace
exit

# Create UTS namespace with other namespaces
sudo su

# Create UTS + PID + mount namespaces
unshare --uts --pid --mount --fork bash

# Set hostname
hostname isolated-container

# Verify
hostname
# Should show: isolated-container

# Remount /proc for PID namespace
umount -l /proc 2>/dev/null || true
mount -t proc proc /proc

# Check process
echo $$
# Should show 1

# Hostname should still be isolated
hostname
# Should show: isolated-container
```

### Step 7: Create a container script with UTS namespace

```bash
# Create a script that uses UTS namespace
cat > /tmp/run_uts_container.sh <<'SCRIPT'
#!/bin/bash
CONTAINER_ROOT="/tmp/container_uts"
CONTAINER_HOSTNAME="mycontainer"

# Create container root (simplified)
mkdir -p "$CONTAINER_ROOT"/{bin,lib,lib64,etc,proc,sys,dev,tmp,root,.oldroot,usr/bin}

# On Ubuntu 24.04, /bin is a symlink to /usr/bin. After pivot_root, bash
# resolves commands via PATH to /usr/bin/. We mirror that structure with
# a symlink so binaries are found regardless of PATH resolution.
rm -rf "$CONTAINER_ROOT/usr/bin"
ln -s ../../bin "$CONTAINER_ROOT/usr/bin"

# Copy essential binaries
cp /bin/bash "$CONTAINER_ROOT/bin/"
cp /bin/mount "$CONTAINER_ROOT/bin/"
cp /bin/umount "$CONTAINER_ROOT/bin/"
cp /bin/rmdir "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/rmdir "$CONTAINER_ROOT/bin/"
cp /bin/hostname "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/hostname "$CONTAINER_ROOT/bin/"
cp /bin/readlink "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/readlink "$CONTAINER_ROOT/bin/"
cp /bin/ls "$CONTAINER_ROOT/bin/"
cp /bin/cat "$CONTAINER_ROOT/bin/"
cp /bin/head "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/head "$CONTAINER_ROOT/bin/"

# Copy libraries
for binary in /bin/bash /bin/mount /bin/umount /usr/bin/rmdir /usr/bin/hostname /usr/bin/readlink /bin/ls /bin/cat /usr/bin/head; do
    ldd "$binary" 2>/dev/null | grep -o '/[^ ]*' | while read lib; do
        mkdir -p "$CONTAINER_ROOT$(dirname "$lib")"
        cp "$lib" "$CONTAINER_ROOT$lib" 2>/dev/null || true
    done
done

# The script prepares the rootfs. Now we run the container separately
# so that exec /bin/bash gets an interactive terminal.
SCRIPT

chmod +x /tmp/run_uts_container.sh

# Run the rootfs setup
/tmp/run_uts_container.sh

# Create the namespace entry script (runs inside unshare)
cat > /tmp/enter_uts_container.sh <<'SCRIPT'
#!/bin/bash
CONTAINER_ROOT="/tmp/container_uts"
CONTAINER_HOSTNAME="mycontainer"

# Set hostname
hostname "$CONTAINER_HOSTNAME"

# Mount filesystems
mount -t proc proc "$CONTAINER_ROOT/proc"
mount -t sysfs sysfs "$CONTAINER_ROOT/sys"

# Bind mount and pivot
mount --bind "$CONTAINER_ROOT" "$CONTAINER_ROOT"
mkdir -p "$CONTAINER_ROOT/.oldroot"
cd "$CONTAINER_ROOT"
pivot_root . .oldroot

# Update working directory to new root
cd /

# Mount proc (needed before umount can resolve mount points)
mount -t proc proc /proc
mount -t sysfs sysfs /sys

# Unmount old root
umount -l /.oldroot
rmdir /.oldroot

# Remount /proc
umount -l /proc 2>/dev/null || true
mount -t proc proc /proc

# Verify hostname
echo "Container hostname: $(hostname)"
echo "Container PID: $$"

# Start shell
exec /bin/bash
SCRIPT

chmod +x /tmp/enter_uts_container.sh
```

### Step 8: Run the container

```bash
# Enter the container with UTS + PID + mount namespaces
sudo unshare --uts --pid --mount --fork /tmp/enter_uts_container.sh
```

### Step 9: Verify inside container

Inside the container:

```bash
# Check hostname
hostname
# Should show: mycontainer

# Check using uname
uname -n
# Should show: mycontainer

# Check /proc
cat /proc/sys/kernel/hostname
# Should show: mycontainer

# Verify UTS namespace
readlink /proc/self/ns/uts
# Should be different from host
```

### Step 10: Test hostname persistence

```bash
# Change hostname inside container
hostname newname

# Verify change
hostname
# Should show: newname

# Exit and check host
exit

# Check host hostname (in another terminal)
hostname
# Should still be original hostname - isolation works!
```

## Verification

To verify UTS namespace isolation:

1. **UTS namespace ID**: `readlink /proc/self/ns/uts` should be different from host
2. **Hostname isolation**: Hostname in container is different from host
3. **Host unaffected**: Changing hostname in container doesn't affect host
4. **Process inheritance**: Child processes inherit UTS namespace
5. **Multiple containers**: Can run multiple containers with different hostnames

## Common Pitfalls

1. **Hostname not set**: Must explicitly set hostname in namespace
   ```bash
   hostname mycontainer
   ```

2. **Checking wrong namespace**: Verify you're in the right namespace
   ```bash
   readlink /proc/self/ns/uts
   ```

3. **Hostname file**: `/etc/hostname` may not reflect namespace hostname
   ```bash
   # Use hostname command or /proc/sys/kernel/hostname
   hostname
   cat /proc/sys/kernel/hostname
   ```

4. **Domain name**: UTS also isolates domain name (NIS domain)
   ```bash
   domainname mydomain
   ```

## Understanding Hostname Isolation

Why hostname isolation matters:
- **Application identification**: Many apps use hostname for identification
- **Service discovery**: Hostname used in service discovery
- **Logging**: Hostname appears in logs
- **Security**: Hostname can be used in security policies

Example use cases:
- Multiple web servers with different hostnames
- Database clusters where each node needs unique hostname
- Microservices where hostname identifies the service

## Next Steps

In Step 4.2, we'll learn about IPC (Inter-Process Communication) namespaces, which isolate shared memory, semaphores, and message queues. This prevents containers from interfering with each other's IPC resources.
