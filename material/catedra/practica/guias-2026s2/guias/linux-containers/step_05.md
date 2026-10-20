# Step 2.2: Combining PID and Mount Namespaces

## Learning Objectives

- Properly combine PID and mount namespaces
- Understand why PID namespace needs mount namespace
- Verify `/proc` shows isolated process list
- Set up a complete container with process isolation
- Understand the relationship between namespaces

## Prerequisites

- Completion of Step 2.1 (PID Namespace)
- Completion of Step 1.2 (Mount Namespace Isolation)
- Understanding of both PID and mount namespaces
- Root access on a Linux system

## Background

PID and mount namespaces must work together:
- **PID namespace** isolates process IDs
- **Mount namespace** isolates mount points
- `/proc` filesystem must be remounted in the mount namespace to reflect the PID namespace's process list

Without combining them:
- PID namespace alone: `/proc` still shows host processes (wrong process list)
- Mount namespace alone: No process isolation

Together:
- Each container sees only its own processes
- Process IDs start from 1
- `/proc` correctly reflects the container's process namespace

This combination is fundamental to containerization - it's what makes `ps aux` inside a container show only container processes.

## Commands

### Step 1: Create both namespaces together

```bash
# Ensure root access
sudo su

# Create both PID and mount namespaces
# --pid: Create PID namespace
# --mount: Create mount namespace  
# --fork: Required for PID namespace
unshare --pid --mount --fork bash
```

### Step 2: Verify namespaces

```bash
# Check PID namespace ID
readlink /proc/self/ns/pid
echo "PID namespace: $(readlink /proc/self/ns/pid)"

# Check mount namespace ID
readlink /proc/self/ns/mnt
echo "Mount namespace: $(readlink /proc/self/ns/mnt)"

# Compare with host (in another terminal)
# Both should be different
```

### Step 3: Remount /proc for PID namespace

```bash
# Unmount the current /proc (from host view of this mount namespace)
# Prefer lazy umount — /proc is often busy
umount -l /proc 2>/dev/null || true

# Mount a new /proc that reflects this PID namespace
mount -t proc proc /proc

# Verify it worked
echo $$
# Should now show 1 (or low number)

ps aux
# Should only show processes in this namespace
```

### Step 4: Verify process isolation

```bash
# Check current PID
echo $$
# Should be 1

# Check process list
ps aux
# Should show very few processes (only in this namespace)

# Check process tree
pstree -p
# Should show a simple tree starting from PID 1

# Check /proc/self
ls -la /proc/self
# Should show process info for PID 1
```

### Step 5: Create child processes

```bash
# Start some background processes
sleep 1000 &
sleep 2000 &
sleep 3000 &

# Check their PIDs
jobs -l
# Should show PIDs 2, 3, 4 (or similar)

# Verify in process list
ps aux
# Should show PID 1 (bash), and PIDs 2, 3, 4 (sleep processes)
```

### Step 6: Test from host namespace

In another terminal (host):

```bash
# Check namespace IDs
readlink /proc/self/ns/pid
readlink /proc/self/ns/mnt
# Should be different from container

# Check process list
ps aux | grep sleep
# May see the processes, but with DIFFERENT PIDs (host PIDs)

# This demonstrates namespace isolation
```

### Step 7: Complete container setup script

```bash
# Exit current namespace
exit

# Create a complete container setup
sudo su

# Create container root filesystem
CONTAINER_ROOT="/tmp/container_complete"
mkdir -p "$CONTAINER_ROOT"
cd "$CONTAINER_ROOT"

# Create directory structure
mkdir -p bin lib lib64 etc proc sys dev tmp usr/bin usr/lib root .oldroot

# On Ubuntu 24.04, /bin is a symlink to /usr/bin. After pivot_root, bash
# resolves commands via PATH to /usr/bin/. We mirror that structure with
# a symlink so binaries are found regardless of PATH resolution.
rm -rf "$CONTAINER_ROOT/usr/bin"
ln -s ../../bin "$CONTAINER_ROOT/usr/bin"

# Copy essential binaries
cp /bin/bash "$CONTAINER_ROOT/bin/"
cp /bin/ls "$CONTAINER_ROOT/bin/"
cp /bin/cat "$CONTAINER_ROOT/bin/"
cp /bin/ps "$CONTAINER_ROOT/bin/"
cp /bin/pstree "$CONTAINER_ROOT/bin/" 2>/dev/null || true
cp /bin/sleep "$CONTAINER_ROOT/bin/"
cp /bin/mount "$CONTAINER_ROOT/bin/"
cp /bin/umount "$CONTAINER_ROOT/bin/"
cp /bin/rmdir "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/rmdir "$CONTAINER_ROOT/bin/"
cp /bin/readlink "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/readlink "$CONTAINER_ROOT/bin/"
cp /bin/head "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/head "$CONTAINER_ROOT/bin/"
cp /bin/pwd "$CONTAINER_ROOT/bin/"

# Copy libraries
for binary in /bin/bash /bin/ls /bin/cat /bin/ps /bin/sleep /bin/mount /bin/umount /usr/bin/rmdir /usr/bin/readlink /usr/bin/head /bin/pwd; do
    ldd "$binary" 2>/dev/null | grep -o '/[^ ]*' | while read lib; do
        mkdir -p "$CONTAINER_ROOT$(dirname "$lib")"
        cp "$lib" "$CONTAINER_ROOT$lib" 2>/dev/null || true
    done
done

# Create basic config files
cat > "$CONTAINER_ROOT/etc/passwd" <<EOF
root:x:0:0:root:/root:/bin/bash
EOF

cat > "$CONTAINER_ROOT/etc/group" <<EOF
root:x:0:
EOF

# Create container startup script
# Note: the script does NOT call unshare — you run it WITH unshare
# so that exec /bin/bash inherits your terminal for interactive use.
cat > /tmp/run_complete_container.sh <<'SCRIPT'
#!/bin/bash
CONTAINER_ROOT="/tmp/container_complete"

echo "=== Setting up container ==="
echo "PID namespace: $(readlink /proc/self/ns/pid)"
echo "Mount namespace: $(readlink /proc/self/ns/mnt)"

# Mount virtual filesystems
echo "Mounting virtual filesystems..."
mount -t proc proc "$CONTAINER_ROOT/proc"
mount -t sysfs sysfs "$CONTAINER_ROOT/sys"
mount -t devtmpfs devtmpfs "$CONTAINER_ROOT/dev"

# Bind mount and pivot root
echo "Pivoting root..."
mount --bind "$CONTAINER_ROOT" "$CONTAINER_ROOT"
mkdir -p "$CONTAINER_ROOT/.oldroot"
cd "$CONTAINER_ROOT"
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

# Remount /proc for PID namespace (CRITICAL STEP)
echo "Remounting /proc for PID namespace..."
umount -l /proc 2>/dev/null || true
mount -t proc proc /proc

# Verify setup
echo ""
echo "=== Container Info ==="
echo "Current PID: $$"
echo "PID namespace: $(readlink /proc/self/ns/pid)"
echo ""
echo "Process list:"
ps aux
echo ""
echo "Process tree:"
pstree -p 2>/dev/null || ps aux

# Start shell
echo ""
echo "=== Entering container ==="
exec /bin/bash
SCRIPT

chmod +x /tmp/run_complete_container.sh
```

### Step 8: Run the complete container

```bash
# Execute the container script inside PID and mount namespaces
sudo unshare --pid --mount --fork /tmp/run_complete_container.sh
```

### Step 9: Verify inside container

Inside the container:

```bash
# Check PID
echo $$
# Should be 1

# Check process list
ps aux
# Should only show container processes

# Start a process
sleep 5000 &

# Check its PID
echo $!
# Should be 2

# Verify in process list
ps aux
# Should show PID 1 (bash) and PID 2 (sleep)

# Check process details
cat /proc/2/status | head -10
# Should show details of the sleep process
```

## Verification

To verify PID and mount namespaces are working together:

1. **Namespace IDs**: Both PID and mount namespace IDs should be different from host
2. **PID 1**: Current process should be PID 1 after remounting `/proc`
3. **Process list**: `ps aux` should only show processes in the container
4. **Process tree**: `pstree` should start from PID 1
5. **Isolation**: Processes created in container have container PIDs, not host PIDs

## Common Pitfalls

1. **Forgetting to remount /proc**: This is the most common mistake
   ```bash
   # Must do this after creating PID namespace
   umount -l /proc 2>/dev/null || true
   mount -t proc proc /proc
   ```

2. **Order matters**: Create namespaces first, then mount filesystems
   ```bash
   # Correct order:
   # 1. Create namespaces
   # 2. Mount filesystems
   # 3. Pivot root
   # 4. Remount /proc
   ```

3. **Not using --fork**: PID namespace requires `--fork`
   ```bash
   unshare --pid --mount --fork bash
   ```

4. **Mounting /proc before pivot_root**: Mount /proc in container root, then pivot, then remount
   ```bash
   # Mount in container root first
   mount -t proc proc "$CONTAINER_ROOT/proc"
   # After pivot_root, remount
   umount -l /proc 2>/dev/null || true
   mount -t proc proc /proc
   ```

5. **Checking PID before remounting**: `echo $$` will show host PID until `/proc` is remounted

## Understanding the Relationship

The key insight is:
- **PID namespace** determines which processes are visible
- **Mount namespace** determines which `/proc` is visible
- **Together**: The `/proc` in the mount namespace reflects the PID namespace

This is why Docker containers show isolated process lists - they combine both namespaces and remount `/proc`.

## Next Steps

In Step 3.1, we'll learn about network namespaces, which isolate the network stack. This will allow containers to have their own network interfaces, IP addresses, and routing tables, completely isolated from the host.
