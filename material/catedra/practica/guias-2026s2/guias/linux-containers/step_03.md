# Step 1.3: Pivot Root

## Learning Objectives

- Understand `pivot_root` and why it's better than `chroot`
- Learn how to properly switch root filesystems
- Use mount bind to prepare for pivot_root
- Clean up old root mounts
- Compare pivot_root with chroot limitations

## Prerequisites

- Completion of Step 1.1 (Basic Filesystem Isolation)
- Completion of Step 1.2 (Mount Namespace Isolation)
- Understanding of mount namespaces
- Root access on a Linux system

## Background

### What is pivot_root?

`pivot_root` is a Linux system call that changes the root filesystem. Unlike `chroot`, `pivot_root`:
- Properly handles mount points
- Moves the old root to a different location (making it inaccessible)
- Is the method Docker uses internally
- Works better with mount namespaces

### How pivot_root Works with Mount Namespaces

`pivot_root` is designed to work with mount namespaces. Here's how it provides better isolation than `chroot`:

1. **Mount Point Handling**: `pivot_root` properly moves all mount points from the old root to the new root, ensuring that virtual filesystems like `/proc` and `/sys` are correctly positioned.

2. **Old Root Isolation**: The old root filesystem is moved to a subdirectory (e.g., `/.oldroot`) and then unmounted, making it completely inaccessible. This prevents escape attempts.

3. **Namespace Integration**: When used with a mount namespace, `pivot_root` ensures that:
   - The new root is isolated from the host's mount namespace
   - Mount operations in the container don't affect the host
   - The old root cannot be accessed even with path manipulation

### Why pivot_root is Better than chroot

| Feature | chroot | pivot_root |
|---------|--------|------------|
| Mount point handling | Limited | Proper |
| Old root accessible | Yes (via `..`) | No (unmounted) |
| Works with mount namespace | No | Yes |
| Used by Docker | No | Yes |
| Security | Lower | Higher |

### The pivot_root Process

The `pivot_root` process:
1. Create a mount namespace (isolated mount points)
2. Create a directory for the old root (e.g., `/.oldroot`)
3. Bind mount the new root to itself (required by pivot_root)
4. Call `pivot_root` to switch roots
5. Unmount the old root from `/.oldroot`
6. Remove the old root directory

This approach ensures that:
- The old root is completely inaccessible
- Mount points are properly isolated
- The process cannot escape back to the host filesystem

### Mount Namespace Isolation with pivot_root

When `pivot_root` is used within a mount namespace:
- **Isolated Mount Table**: The container has its own mount table, separate from the host
- **Virtual Filesystems**: `/proc`, `/sys`, `/dev` can be mounted independently
- **No Host Access**: The old root is unmounted, preventing access to host filesystem
- **Proper Cleanup**: Old mounts are properly cleaned up

This is why Docker uses `pivot_root` instead of `chroot` - it provides proper filesystem isolation when combined with mount namespaces.

## Key Commands Explained

Before diving into the practical steps, let's understand the key commands we'll be using:

### `pivot_root` - Switch Root Filesystem

The `pivot_root` command changes the root filesystem, moving the old root to a subdirectory. It's a system call that provides better isolation than `chroot`.

**Syntax**: `pivot_root NEW_ROOT PUT_OLD`

**Parameters**:
- `NEW_ROOT`: The new root directory (must be a mount point)
- `PUT_OLD`: Directory where old root will be moved (must be a subdirectory of NEW_ROOT)

**How it works**:
1. Moves the current root filesystem to `PUT_OLD`
2. Makes `NEW_ROOT` the new root (`/`)
3. All mount points are preserved and repositioned
4. The old root becomes inaccessible at its new location

**Requirements**:
- `NEW_ROOT` must be a mount point (hence the bind mount step)
- `PUT_OLD` must be a subdirectory of `NEW_ROOT`
- Must be run from within a mount namespace
- Requires root privileges

**Example**:
```bash
# After bind mounting and creating .oldroot
cd /tmp/container_root_pivot
pivot_root . .oldroot
# Now / refers to /tmp/container_root_pivot
# Old root is at /.oldroot (but will be unmounted)
```

**Why bind mount is required**:
`pivot_root` requires `NEW_ROOT` to be a mount point. Binding the directory to itself creates a mount point:
```bash
mount --bind /tmp/container_root_pivot /tmp/container_root_pivot
```

### `mount --bind` - Bind Mount

The `mount --bind` command creates a bind mount, making one directory tree available at another location. Both locations point to the same underlying filesystem.

**Syntax**: `mount --bind SOURCE DESTINATION`

**How it works**:
- Creates a mount point at `DESTINATION` that points to `SOURCE`
- Both paths access the same files and directories
- Changes in one location are visible in the other
- Creates a mount entry in the mount table

**Why it's needed for pivot_root**:
`pivot_root` requires the new root to be a mount point. Binding a directory to itself creates that mount point:
```bash
mount --bind /tmp/container_root_pivot /tmp/container_root_pivot
```

**Other uses**:
- Sharing directories between namespaces
- Making host directories available in containers
- Creating mount points for `pivot_root`

### `umount` - Unmount Filesystem

The `umount` command detaches a filesystem from the filesystem hierarchy.

**Syntax**: `umount [OPTIONS] MOUNTPOINT`

**Key Options**:
- `-l, --lazy`: Lazy unmount - detach immediately, clean up when not busy
- `-f, --force`: Force unmount (dangerous)
- `-r, --remount`: Remount in read-only mode

**How it works**:
- Removes the mount point from the mount table
- Makes the filesystem inaccessible at that location
- Filesystem remains mounted if still in use

**Example**:
```bash
# Unmount old root (lazy unmount if busy)
umount -l /.oldroot

# Unmount /proc (prefer lazy if busy)
umount -l /proc
```

**Why `-l` (lazy) is used**:
When unmounting `/.oldroot` or `/proc`, processes might still have them open. Lazy unmount detaches immediately but delays cleanup until no longer in use.

**Common usage**:
```bash
# Unmount a specific mount point
umount /tmp/test_mount

# Lazy unmount (if busy)
umount -l /tmp/test_mount

# Unmount all mounts under a directory
umount -R /mnt
```

## Commands

### Step 1: Create a container root filesystem

```bash
# Ensure root access
sudo su

# Create container root directory
mkdir -p /tmp/container_root_pivot

# Create directory structure
cd /tmp/container_root_pivot
mkdir -p bin lib lib64 etc proc sys dev tmp usr/bin usr/lib root .oldroot

# Copy essential binaries
# On Ubuntu 24.04, /bin is a symlink to /usr/bin. After pivot_root, bash
# resolves commands via PATH to /usr/bin/. We mirror that structure with
# a symlink so binaries are found regardless of PATH resolution.
rm -rf /tmp/container_root_pivot/usr/bin
ln -s ../../bin /tmp/container_root_pivot/usr/bin

cp /bin/bash /tmp/container_root_pivot/bin/
cp /bin/ls /tmp/container_root_pivot/bin/
cp /bin/cat /tmp/container_root_pivot/bin/
cp /bin/pwd /tmp/container_root_pivot/bin/
cp /bin/mount /tmp/container_root_pivot/bin/
cp /bin/umount /tmp/container_root_pivot/bin/
cp /bin/rmdir /tmp/container_root_pivot/bin/ 2>/dev/null || cp /usr/bin/rmdir /tmp/container_root_pivot/bin/
cp /bin/readlink /tmp/container_root_pivot/bin/ 2>/dev/null || cp /usr/bin/readlink /tmp/container_root_pivot/bin/
cp /bin/mkdir /tmp/container_root_pivot/bin/ 2>/dev/null || cp /usr/bin/mkdir /tmp/container_root_pivot/bin/
cp /bin/ps /tmp/container_root_pivot/bin/ 2>/dev/null || cp /usr/bin/ps /tmp/container_root_pivot/bin/
cp /bin/head /tmp/container_root_pivot/bin/ 2>/dev/null || cp /usr/bin/head /tmp/container_root_pivot/bin/

# Copy libraries
for binary in /bin/bash /bin/ls /bin/cat /bin/pwd /bin/mount /bin/umount /usr/bin/rmdir /usr/bin/readlink /usr/bin/mkdir /usr/bin/ps /usr/bin/head; do
    ldd "$binary" 2>/dev/null | grep -o '/[^ ]*' | while read lib; do
        mkdir -p "/tmp/container_root_pivot$(dirname "$lib")"
        cp "$lib" "/tmp/container_root_pivot$lib" 2>/dev/null || true
    done
done

# Create basic config files
cat > /tmp/container_root_pivot/etc/passwd <<EOF
root:x:0:0:root:/root:/bin/bash
EOF

cat > /tmp/container_root_pivot/etc/group <<EOF
root:x:0:
EOF

cat > /tmp/container_root_pivot/etc/hosts <<EOF
127.0.0.1 localhost
EOF
```

### Step 2: Create a script to use pivot_root

```bash
# Create a script that performs pivot_root setup
# Note: the script does NOT call unshare internally — you run it WITH unshare
# in Step 3. This ensures the interactive shell gets your terminal's TTY.
cat > /tmp/pivot_root_demo.sh <<'SCRIPT'
#!/bin/bash
CONTAINER_ROOT="/tmp/container_root_pivot"

echo "Bind mounting new root..."
mount --bind "$CONTAINER_ROOT" "$CONTAINER_ROOT"

mkdir -p "$CONTAINER_ROOT/.oldroot"

cd "$CONTAINER_ROOT"

echo "Pivoting root..."
pivot_root . .oldroot

# Update working directory to new root
cd /

# Mount proc (needed before umount can resolve mount points)
echo "Mounting virtual filesystems..."
mount -t proc proc /proc
mount -t sysfs sysfs /sys

# Unmount old root
echo "Unmounting old root..."
umount -l /.oldroot
rmdir /.oldroot

# Verify
echo "Verification:"
echo "Current directory: $(pwd)"
echo "Root filesystem:"
ls -la /

# Start an interactive shell in the container
echo "Starting shell in container..."
exec /bin/bash
SCRIPT

chmod +x /tmp/pivot_root_demo.sh
```

### Step 3: Run the pivot_root script

```bash
# Run the script inside a new mount namespace
# (unshare is called here, not inside the script, so exec /bin/bash
# inherits your terminal and gives you an interactive shell)
sudo unshare --mount --fork /tmp/pivot_root_demo.sh
```

### Step 4: Verify pivot_root worked

Inside the container (after running the script):

```bash
# Check current directory
pwd
# Should show: /

# List root filesystem
ls -la /
# Should only show container directories

# Try to access old root
ls /.oldroot
# Should fail - old root was unmounted and removed

# Check /proc (should work)
ls /proc
# Should show process directories

# Check process list (still shows host - we'll fix in Step 2.1)
ps aux | head -5

# Verify mount namespace
readlink /proc/self/ns/mnt
# Should show a different namespace ID
```

### Step 5: Compare with chroot limitations

Exit the container and test chroot escape:

```bash
# Exit container
exit

# Test: Can we access host filesystem from chroot?
# (This demonstrates why pivot_root is more secure)
```

### Step 6: Manual pivot_root demonstration

For better understanding, let's do it step by step:

```bash
# Start fresh
sudo su

# Create mount namespace
unshare --mount --fork bash

# Bind mount new root (required by pivot_root)
mount --bind /tmp/container_root_pivot /tmp/container_root_pivot

# Create .oldroot
mkdir -p /tmp/container_root_pivot/.oldroot

# Change directory
cd /tmp/container_root_pivot

# Pivot root
pivot_root . .oldroot

# Update working directory to new root
cd /

# Verify old root is at .oldroot
ls /.oldroot
# Should show host root filesystem

# Mount proc (needed before umount can resolve mount points)
mount -t proc proc /proc
mount -t sysfs sysfs /sys

# Unmount old root
umount -l /.oldroot

# Remove old root directory
rmdir /.oldroot

# Verify isolation
ls /
# Should only show container filesystem

# Verify /proc works
cat /proc/self/status | head -5
# Should show process info

# Start shell
exec /bin/bash
```

## Verification

To verify pivot_root worked correctly:

1. **Root directory**: Run `ls /` - should only show container directories
2. **Old root inaccessible**: Try `ls /.oldroot` - should fail (directory removed)
3. **Mount namespace**: Run `readlink /proc/self/ns/mnt` - should be different from host
4. **Virtual filesystems**: Check `/proc` and `/sys` - should be accessible
5. **Cannot escape**: Try to access host filesystem - should be impossible

## Common Pitfalls

1. **Not in mount namespace**: `pivot_root` requires a mount namespace
   ```bash
   unshare --mount --fork bash
   ```

2. **Forgetting bind mount**: New root must be bind-mounted to itself
   ```bash
   mount --bind "$CONTAINER_ROOT" "$CONTAINER_ROOT"
   ```

3. **Old root not unmounted**: Must unmount old root before removing directory
   ```bash
   umount -l /.oldroot
   rmdir /.oldroot
   ```

4. **Wrong directory**: Must `cd` into new root before calling `pivot_root`
   ```bash
   cd "$CONTAINER_ROOT"
   pivot_root . .oldroot
   ```

5. **Permission denied**: Requires root privileges
   ```bash
   sudo su
   ```

## Understanding the Difference: chroot vs pivot_root

### chroot limitations:
- Old root still accessible via `..` or absolute paths
- Mount points not properly isolated
- Can potentially escape with careful path manipulation

### pivot_root advantages:
- Old root moved to `/.oldroot` and then removed
- Proper mount namespace isolation
- Cannot access old root filesystem
- Docker uses this method internally

Test the difference:

```bash
# With chroot (Step 1.1), you could potentially do:
chroot /tmp/container_root /bin/bash
cd /..  # Might access host filesystem

# With pivot_root, this is impossible:
# Old root is unmounted and removed
```

## Next Steps

In Step 2.1, we'll learn about PID namespaces, which will isolate process IDs. This means processes inside the container will start from PID 1, and they won't be able to see host processes. This is a crucial step toward proper containerization.
