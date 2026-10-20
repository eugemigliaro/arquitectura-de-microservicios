# Step 1.2: Mount Namespace Isolation

## Learning Objectives

- Understand mount namespace isolation
- Learn how to create a new mount namespace
- Mount separate filesystems within a namespace
- Verify mount point isolation
- Understand mount propagation (shared, private, slave)

## Prerequisites

- Completion of Step 1.1 (Basic Filesystem Isolation)
- Understanding of Linux mount points
- Root access on a Linux system

## Background

### What are Mount Namespaces?

A mount namespace is a Linux kernel feature that provides an isolated view of the filesystem mount points. Each mount namespace has its own independent set of mount points, which means processes in different namespaces can see completely different filesystem hierarchies, even though they may be running on the same physical machine.

### How Namespaces Provide Mount Isolation

When a new mount namespace is created, it starts as a copy of the parent namespace's mount points. However, from that point forward, mount and unmount operations in one namespace are completely isolated from other namespaces. This isolation works at multiple levels:

1. **Mount Point Visibility**: Each namespace maintains its own mount table. When you mount a filesystem in namespace A, processes in namespace B cannot see it unless mount propagation is explicitly configured.

2. **Filesystem Hierarchy**: Different namespaces can have different filesystems mounted at the same path. For example:
   - Host namespace: `/tmp` might be on the root filesystem
   - Container namespace: `/tmp` might be a `tmpfs` (RAM-based filesystem)
   - Both exist simultaneously without interfering with each other

3. **Mount/Unmount Operations**: Operations in one namespace don't affect others:
   - Mounting `/proc` in a container namespace doesn't affect the host's `/proc`
   - Unmounting a filesystem in a container doesn't unmount it on the host
   - Each namespace can independently manage its filesystem view

4. **Process Perspective**: When a process runs in a mount namespace, it sees only the mount points that exist in that namespace. The kernel provides this isolated view transparently - the process doesn't need to know it's in a namespace.

### Why Mount Namespaces are Crucial for Containers

Mount namespaces are essential for containers because:
- They allow each container to have its own `/proc`, `/sys`, `/dev`, etc., providing process, system, and device isolation
- They prevent mount/unmount operations in one container from affecting others or the host
- They enable proper filesystem isolation, allowing containers to have different root filesystems
- They support bind mounts and volume mounts that are only visible within the container

### Mount Propagation

Mount propagation determines how mount events propagate between namespaces:
- **Shared**: Mount/unmount events propagate to peer groups (useful for shared volumes)
- **Private**: Mount events don't propagate (default, most secure for containers)
- **Slave**: Receives mount events from master but doesn't propagate its own (useful for host-to-container volumes)
- **Unbindable**: Cannot be bind-mounted elsewhere (prevents accidental sharing)

## Key Commands Explained

Before diving into the practical steps, let's understand the key commands we'll be using:

### `unshare` - Create New Namespaces

The `unshare` command allows you to run a program in new namespaces, effectively isolating it from the parent namespace. It's part of the `util-linux` package.

**Syntax**: `unshare [OPTIONS] [PROGRAM [ARGUMENTS]]`

**Key Options**:
- `--mount` or `-m`: Create a new mount namespace
- `--fork` or `-f`: Fork a new process (required when running an interactive shell)
- `--pid`: Create a new PID namespace (we'll use this in later steps)
- `--uts`: Create a new UTS namespace (hostname isolation)
- `--net`: Create a new network namespace

**How it works**:
- When you run `unshare --mount --fork bash`, it:
  1. Creates a new mount namespace (isolated from parent)
  2. Forks a new process (required for interactive programs)
  3. Executes `bash` in that new namespace
  4. All mount operations inside this bash session are isolated

**Example**:
```bash
# Create new mount namespace and run bash
unshare --mount --fork bash

# Now any mounts/unmounts are isolated
mount -t tmpfs tmpfs /tmp/test
# This mount is only visible in this namespace
```

### `readlink` - Read Symbolic Link Values

The `readlink` command reads the value of a symbolic link. In the context of namespaces, it's used to read namespace identifiers.

**Syntax**: `readlink [OPTIONS] FILE`

**In namespace context**:
- `/proc/self/ns/mnt` is a symbolic link pointing to the mount namespace ID
- Each namespace has a unique identifier (inode number)
- Comparing namespace IDs tells you if processes are in the same namespace

**How it works**:
- `/proc/self/ns/` contains symbolic links to namespace identifiers
- `readlink /proc/self/ns/mnt` returns something like: `mnt:[4026531840]`
- The number `4026531840` is the namespace ID (inode number)
- Processes with the same namespace ID share the same namespace

**Example**:
```bash
# Check current mount namespace ID
readlink /proc/self/ns/mnt
# Output: mnt:[4026531840]

# In another terminal (different namespace)
readlink /proc/self/ns/mnt
# Output: mnt:[4026531841]  # Different ID = different namespace
```

**Why it's useful**:
- Verify you're in a new namespace (ID should differ from host)
- Debug namespace isolation issues
- Understand which processes share namespaces

### `findmnt` - Find Mount Points

The `findmnt` command is a modern, powerful tool to find and display mount points. It's part of `util-linux` and provides more detailed information than the traditional `mount` command.

**Syntax**: `findmnt [OPTIONS] [DEVICE|MOUNTPOINT]`

**Key Options**:
- `-o, --output`: Specify output columns (e.g., `TARGET,PROPAGATION`)
- `-t, --types`: Filter by filesystem type
- `-n, --noheadings`: Don't print column headers
- `-r, --raw`: Raw output format

**Common Output Columns**:
- `TARGET`: Mount point path
- `SOURCE`: Source device/filesystem
- `FSTYPE`: Filesystem type
- `PROPAGATION`: Mount propagation type (shared, private, slave, unbindable)
- `OPTIONS`: Mount options

**How it works**:
- Reads `/proc/self/mountinfo` (more detailed than `/proc/mounts`)
- Provides structured, parseable output
- Shows mount propagation information (crucial for understanding namespace behavior)

**Example**:
```bash
# List all mounts with propagation info
findmnt -o TARGET,PROPAGATION

# Check specific mount point
findmnt -o TARGET,PROPAGATION /tmp/test_mount
# Output:
# TARGET          PROPAGATION
# /tmp/test_mount private

# List only tmpfs mounts
findmnt -t tmpfs
```

**Why it's better than `mount`**:
- Shows mount propagation (shared/private/slave/unbindable)
- More structured output (easier to parse)
- Shows mount relationships and dependencies
- Better for understanding namespace behavior

## Commands

### Step 1: Create a new mount namespace

```bash
# Ensure you have root access
sudo su

# Create a new mount namespace and enter it
unshare --mount --fork bash
```

### Step 2: Verify you're in a new mount namespace

```bash
# Check mount namespace ID (should be different from host)
readlink /proc/self/ns/mnt
# Compare with another terminal: readlink /proc/self/ns/mnt

# List current mount points
mount | head -5
```

### Step 3: Create a temporary filesystem to test isolation

```bash
# Create a directory for our test
mkdir -p /tmp/test_mount

# Create a temporary filesystem (tmpfs)
mount -t tmpfs tmpfs /tmp/test_mount

# Verify it's mounted
mount | grep test_mount
# Should show: tmpfs on /tmp/test_mount type tmpfs

# Create a file in the mounted filesystem
echo "Hello from mount namespace" > /tmp/test_mount/test.txt
cat /tmp/test_mount/test.txt
```

### Step 4: Verify isolation from host

In another terminal (outside the namespace):

```bash
# Check if you can see the mount
mount | grep test_mount
# Should show nothing - the mount is isolated!

# Try to access the file
cat /tmp/test_mount/test.txt
# Should fail - the mount doesn't exist in the host namespace
```

### Step 5: Test mount propagation

```bash
# Check current propagation type
findmnt -o TARGET,PROPAGATION /tmp/test_mount
# Default is usually "private"

# Change propagation to shared
mount --make-shared /tmp/test_mount

# Verify
findmnt -o TARGET,PROPAGATION /tmp/test_mount
# Should show "shared"
```

### Step 6: Create a proper container root filesystem with mounts

```bash
# Create container root directory
mkdir -p /tmp/container_root_v2

# Create directory structure
cd /tmp/container_root_v2
mkdir -p bin lib lib64 etc proc sys dev tmp usr/bin usr/lib root

# On Ubuntu 24.04, /bin is a symlink to /usr/bin. After chroot, bash
# resolves commands via PATH to /usr/bin/. We mirror that structure with
# a symlink so binaries are found regardless of PATH resolution.
rm -rf /tmp/container_root_v2/usr/bin
ln -s ../../bin /tmp/container_root_v2/usr/bin

# Copy essential binaries (similar to Step 1.1)
cp /bin/bash /tmp/container_root_v2/bin/
cp /bin/ls /tmp/container_root_v2/bin/
cp /bin/cat /tmp/container_root_v2/bin/
cp /bin/mount /tmp/container_root_v2/bin/
cp /bin/grep /tmp/container_root_v2/bin/

# Copy libraries
for binary in /bin/bash /bin/ls /bin/cat /bin/mount /bin/grep; do
    ldd "$binary" 2>/dev/null | grep -o '/[^ ]*' | while read lib; do
        mkdir -p "/tmp/container_root_v2$(dirname "$lib")"
        cp "$lib" "/tmp/container_root_v2$lib" 2>/dev/null || true
    done
done

# Create basic config files
cat > /tmp/container_root_v2/etc/passwd <<EOF
root:x:0:0:root:/root:/bin/bash
EOF

cat > /tmp/container_root_v2/etc/group <<EOF
root:x:0:
EOF
```

### Step 7: Mount proc and sys in the namespace

```bash
# Ensure you're in a mount namespace (from Step 1)
# If not, run: unshare --mount --fork bash

# Mount proc filesystem
mount -t proc proc /tmp/container_root_v2/proc

# Mount sys filesystem
mount -t sysfs sysfs /tmp/container_root_v2/sys

# Mount dev filesystem
mount -t devtmpfs devtmpfs /tmp/container_root_v2/dev

# Verify mounts
mount | grep container_root_v2
```

### Step 8: Enter the container with chroot

```bash
# Now combine mount namespace with chroot
chroot /tmp/container_root_v2 /bin/bash

# Inside the chroot, verify mounts
mount | grep -E 'proc|sys|dev'
# Should show the mounted filesystems

# Check /proc (should work now!)
ls /proc
# Should show process-related directories

# Check process list (still shows host processes - we'll fix this in Step 2.1)
cat /proc/self/status
```

### Step 9: Test mount isolation

In the container:

```bash
# Create a test mount
mkdir -p /tmp/test
mount -t tmpfs tmpfs /tmp/test
echo "isolated" > /tmp/test/file.txt
```

In another terminal (host):

```bash
# Try to see the mount
mount | grep container_root_v2
# Should not show the /tmp/test mount - it's isolated!
```

## Verification

To verify mount namespace isolation:

1. **Mount namespace ID**: Run `readlink /proc/self/ns/mnt` - should be different from host
2. **Isolated mounts**: Mounts created inside namespace should not be visible on host
3. **Proc/Sys mounts**: `/proc` and `/sys` should be accessible inside container
4. **Mount propagation**: Test shared/private propagation modes

## Common Pitfalls

1. **Not in mount namespace**: Ensure you use `unshare --mount` before mounting
   ```bash
   # Verify namespace
   readlink /proc/self/ns/mnt
   ```

2. **Mount points not accessible**: Ensure mount points exist before mounting
   ```bash
   mkdir -p /tmp/container_root_v2/proc
   mount -t proc proc /tmp/container_root_v2/proc
   ```

3. **Permission denied**: Mount operations require root privileges
   ```bash
   sudo unshare --mount --fork bash
   ```

4. **Cannot unmount**: If you can't unmount, check what's using it:
   ```bash
   lsof /tmp/test_mount
   fuser -m /tmp/test_mount
   ```

5. **Mount propagation confusion**: Understand the difference between shared, private, slave, and unbindable

## Understanding Mount Propagation

Test different propagation types:

```bash
# Create a test mount
mkdir -p /tmp/prop_test
mount -t tmpfs tmpfs /tmp/prop_test

# Check current propagation
findmnt -o TARGET,PROPAGATION /tmp/prop_test

# Change to shared
mount --make-shared /tmp/prop_test

# Change to private
mount --make-private /tmp/prop_test

# Change to slave
mount --make-slave /tmp/prop_test

# Change to unbindable
mount --make-unbindable /tmp/prop_test
```

## Next Steps

In Step 1.3, we'll learn about `pivot_root`, which is Docker's preferred method for switching root filesystems. `pivot_root` is more secure than `chroot` and properly handles mount points.
