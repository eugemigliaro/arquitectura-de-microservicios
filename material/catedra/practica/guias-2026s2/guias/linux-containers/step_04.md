# Step 2.1: PID Namespace

## Learning Objectives

- Understand PID namespace isolation
- Learn how to create a new PID namespace
- Understand the init process (PID 1) responsibilities
- Verify process IDs start from 1 in the namespace
- See how processes appear isolated

## Prerequisites

- Completion of Step 1.3 (Pivot Root)
- Understanding of Linux processes
- Root access on a Linux system

## Background

### What is a PID Namespace?

A PID (Process ID) namespace isolates process ID numbers. In a new PID namespace:
- Process IDs start from 1 (not from the host's PID numbers)
- Processes cannot see processes outside their namespace
- The first process becomes PID 1 (init process)
- PID 1 has special responsibilities (reaping zombie processes, signal handling)

### How PID Namespaces Provide Process Isolation

PID namespaces create an isolated view of the process tree:

1. **Process ID Isolation**: Each namespace has its own PID number space. The same process can have different PIDs in different namespaces:
   - In host namespace: Process might be PID 5000
   - In container namespace: Same process is PID 1

2. **Process Visibility**: Processes in a PID namespace can only see other processes in the same namespace. They cannot see:
   - Host processes
   - Processes in other containers
   - Processes in parent namespaces (unless explicitly shared)

3. **Nested Namespaces**: PID namespaces are hierarchical and nested:
   - A process can have different PIDs in different namespaces
   - The "innermost" namespace shows PID 1
   - Outer namespaces show progressively higher PIDs
   - `/proc/<pid>/status` shows PIDs in all namespaces

4. **Init Process (PID 1)**: The first process in a PID namespace becomes PID 1, which has special responsibilities:
   - **Zombie Reaping**: Must call `wait()` to clean up zombie child processes
   - **Signal Handling**: Handles signals sent to orphaned processes
   - **Shutdown**: Handles system shutdown signals (SIGTERM, SIGINT)

### Why PID Namespace Requires Mount Namespace

**Critical**: PID namespace requires a mount namespace because `/proc` must be remounted to show the correct process list for that namespace.

**How it works**:
1. **Initial `/proc`**: When you create a PID namespace, `/proc` still shows the host's process list
2. **Remount Required**: You must unmount and remount `/proc` to make it reflect the PID namespace
3. **After Remount**: `/proc` now shows only processes in the PID namespace
4. **Process Commands**: Tools like `ps`, `top`, `pstree` read from `/proc`, so they now show isolated processes

**Without remounting `/proc`**:
- `ps aux` shows all host processes
- `echo $$` shows host PID
- Process isolation doesn't work properly

**After remounting `/proc`**:
- `ps aux` shows only container processes
- `echo $$` shows container PID (likely 1)
- Process isolation works correctly

### PID Namespace Isolation Benefits

PID namespace provides:
- **Process Isolation**: Containers can't see host processes
- **Security**: Prevents information leakage about host processes
- **Clean Process Tree**: Each container starts from PID 1
- **Proper Init**: Container can have its own init process
- **Resource Management**: Easier to manage and limit container processes

This is why Docker containers show isolated process lists - they use PID namespaces combined with mount namespaces to remount `/proc`.

## Key Commands Explained

Before diving into the practical steps, let's understand the key commands we'll be using:

### `ps` - Process Status

The `ps` command displays information about running processes. It reads process information from `/proc` filesystem.

**Syntax**: `ps [OPTIONS]`

**Key Options**:
- `aux`: Show all processes with detailed info (BSD-style)
- `-e, -A`: Show all processes
- `-f`: Full format listing
- `-p PID`: Show specific process(es)

**How it works**:
- Reads process information from `/proc/<pid>/` directories
- Shows PID, user, CPU usage, memory, command, etc.
- In PID namespace: Shows only processes in that namespace (after `/proc` remount)

**Example**:
```bash
# Show all processes
ps aux

# Show specific process
ps -p 1

# Show process tree
ps auxf
```

**In PID namespace context**:
- Before remounting `/proc`: Shows host processes
- After remounting `/proc`: Shows only namespace processes
- PID 1 appears as the first process

### `pstree` - Process Tree

The `pstree` command displays processes in a tree format, showing parent-child relationships.

**Syntax**: `pstree [OPTIONS] [PID]`

**Key Options**:
- `-p`: Show PIDs
- `-u`: Show user names
- `-a`: Show command arguments

**How it works**:
- Reads process relationships from `/proc`
- Displays hierarchical tree structure
- Shows which processes are children of which parents

**Example**:
```bash
# Show process tree with PIDs
pstree -p

# Show tree starting from PID 1
pstree -p 1
```

**In PID namespace context**:
- Shows process tree within the namespace
- PID 1 is the root of the tree
- All processes are descendants of PID 1
- Much simpler tree than host (only container processes)

### `echo $$` - Current Process ID

The `$$` variable contains the PID of the current shell process.

**How it works**:
- `$$` is a shell variable that expands to the current shell's PID
- Reads from `/proc/self/pid` (which reads from `/proc`)
- In PID namespace: Shows container PID after `/proc` remount

**Example**:
```bash
# Show current shell PID
echo $$

# In PID namespace (after remount):
# Shows: 1 (or low number)
# In host namespace:
# Shows: High number (e.g., 5000)
```

**Why it's useful**:
- Quick way to check if you're in a PID namespace
- Verifies `/proc` remount worked correctly
- Confirms PID 1 status

### `/proc` Filesystem and PID Namespaces

The `/proc` filesystem is crucial for PID namespace functionality:

**Key files**:
- `/proc/self/pid`: Current process PID
- `/proc/self/status`: Process status (includes namespace info)
- `/proc/<pid>/`: Directory for each process
- `/proc/<pid>/ns/pid`: PID namespace ID

**Remounting `/proc`**:
```bash
# Unmount old /proc (shows host processes)
umount /proc

# Mount new /proc (shows namespace processes)
mount -t proc proc /proc

# Now /proc reflects the PID namespace
```

**Why remount is needed**:
- Initial `/proc` is from host mount namespace
- Shows host processes
- Must remount to show namespace processes
- This is why PID namespace requires mount namespace

## Commands

### Step 1: Create a PID namespace

> **⚠️ IMPORTANTE:** En los pasos 1-3 creamos el PID namespace **sin** mount namespace para entender el problema. Esto es solo demostrativo — `umount /proc` sin mount namespace afecta al host. Si algo sale mal, podés restaurar `/proc` con `mount -t proc proc /proc` desde otra terminal. En el Step 8 combinamos correctamente ambos namespaces.

```bash
# Ensure root access
sudo su

# Create a new PID namespace (WITHOUT mount namespace - to demonstrate the problem)
# Note: --fork is needed because PID namespace requires a new process
unshare --pid --fork bash
```

### Step 2: Verify PID namespace

```bash
# Check PID namespace ID
readlink /proc/self/ns/pid
# Note this ID - compare with host namespace

# Check current PID
echo $$
# Shows 1 - the shell variable $$ reflects the PID namespace
# However, ps aux still shows host processes because /proc isn't remounted

# Check process list
ps aux
# Still shows host processes - /proc needs to be remounted
```

### Step 3: Remount /proc (required for PID namespace)

> **⚠️ CUIDADO:** Sin mount namespace, este `umount /proc` afecta al host. Es solo para demostrar por qué se necesita combinar PID + mount namespace (Step 8).

```bash
# Unmount old /proc (WARNING: affects host without mount namespace!)
umount /proc

# Mount new /proc (this will show processes from PID namespace)
mount -t proc proc /proc

# Now check process list
ps aux
# Should only show processes in this namespace (much fewer processes)
```

### Step 4: Verify PID 1

```bash
# Check what process is PID 1
ps -p 1
# Should show your bash process

# Check process tree
pstree -p
# Should show a simple tree starting from PID 1

# Check process details
cat /proc/1/status | head -20
# Should show details of PID 1 process
```

### Step 5: Create child processes

```bash
# Start a background process
sleep 1000 &
BG_PID=$!

# Check its PID
echo $BG_PID
# Should be 2 (or next available PID)

# Verify it exists
ps -p $BG_PID
# Should show the sleep process

# Check process list
ps aux
# Should show PID 1 (bash) and PID 2 (sleep)
```

### Step 6: Test PID namespace isolation

In another terminal (host namespace):

```bash
# Check PID namespace ID
readlink /proc/self/ns/pid
# Should be different from container namespace

# Try to see container processes
ps aux | grep sleep
# May or may not see it, but PID will be different

# Check if you can signal the process
# (This demonstrates namespace isolation)
```

### Step 7: Understand PID 1 responsibilities

```bash
# Inside the PID namespace, create a zombie process
# (This demonstrates why PID 1 is important)

# Create a simple script that creates a zombie
cat > /tmp/create_zombie.sh <<'EOF'
#!/bin/bash
# Fork a child that exits immediately (becomes zombie)
(exit) &
wait
sleep 5  # Give time to observe
EOF

chmod +x /tmp/create_zombie.sh
/tmp/create_zombie.sh &

# Check for zombie processes
ps aux | grep Z
# Zombies should be reaped by PID 1 (bash does this)
```

### Step 8: Combine with filesystem isolation

```bash
# Exit current namespace
exit

# Create a complete container setup with PID namespace
sudo su

# Create container root
mkdir -p /tmp/container_pid
cd /tmp/container_pid
mkdir -p bin lib lib64 etc proc sys dev tmp usr/bin root

# On Ubuntu 24.04, /bin is a symlink to /usr/bin. After pivot_root, bash
# resolves commands via PATH to /usr/bin/. We mirror that structure with
# a symlink so binaries are found regardless of PATH resolution.
rm -rf /tmp/container_pid/usr/bin
ln -s ../../bin /tmp/container_pid/usr/bin

# Copy binaries
cp /bin/bash /tmp/container_pid/bin/
cp /bin/ls /tmp/container_pid/bin/
cp /bin/ps /tmp/container_pid/bin/
cp /bin/sleep /tmp/container_pid/bin/
cp /bin/mount /tmp/container_pid/bin/
cp /bin/umount /tmp/container_pid/bin/
cp /bin/cat /tmp/container_pid/bin/
cp /bin/rmdir /tmp/container_pid/bin/ 2>/dev/null || cp /usr/bin/rmdir /tmp/container_pid/bin/
cp /bin/readlink /tmp/container_pid/bin/ 2>/dev/null || cp /usr/bin/readlink /tmp/container_pid/bin/
cp /bin/pstree /tmp/container_pid/bin/ 2>/dev/null || cp /usr/bin/pstree /tmp/container_pid/bin/ || true
cp /bin/head /tmp/container_pid/bin/ 2>/dev/null || cp /usr/bin/head /tmp/container_pid/bin/

# Copy libraries
for binary in /bin/bash /bin/ls /bin/ps /bin/sleep /bin/mount /bin/umount /bin/cat /usr/bin/rmdir /usr/bin/readlink /usr/bin/pstree /usr/bin/head; do
    ldd "$binary" 2>/dev/null | grep -o '/[^ ]*' | while read lib; do
        mkdir -p "/tmp/container_pid$(dirname "$lib")"
        cp "$lib" "/tmp/container_pid$lib" 2>/dev/null || true
    done
done

# Create config files
cat > /tmp/container_pid/etc/passwd <<EOF
root:x:0:0:root:/root:/bin/bash
EOF

# Create a script that sets up PID namespace + filesystem
# Note: the script does NOT call unshare — you run it WITH unshare
# so that exec /bin/bash inherits your terminal for interactive use.
cat > /tmp/run_pid_container.sh <<'SCRIPT'
#!/bin/bash
CONTAINER_ROOT="/tmp/container_pid"

# Mount virtual filesystems
mount -t proc proc "$CONTAINER_ROOT/proc"
mount -t sysfs sysfs "$CONTAINER_ROOT/sys"
mount -t devtmpfs devtmpfs "$CONTAINER_ROOT/dev"

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

# Remount /proc for PID namespace
umount -l /proc 2>/dev/null || true
mount -t proc proc /proc

# Verify PID
echo "Container PID: $$"
echo "Process list:"
ps aux

# Start shell
exec /bin/bash
SCRIPT

chmod +x /tmp/run_pid_container.sh
sudo unshare --pid --mount --fork /tmp/run_pid_container.sh
```

## Verification

To verify PID namespace isolation:

1. **PID namespace ID**: Run `readlink /proc/self/ns/pid` - should be different from host
2. **PID 1**: Run `echo $$` after remounting `/proc` - should show 1 or low number
3. **Process list**: Run `ps aux` - should only show processes in namespace
4. **Process tree**: Run `pstree -p` - should start from PID 1
5. **Isolation**: Processes in container should not be visible from host (or have different PIDs)

## Common Pitfalls

1. **Forgetting to remount /proc**: PID namespace requires `/proc` to be remounted
   ```bash
   umount -l /proc 2>/dev/null || true
   mount -t proc proc /proc
   ```

2. **Using --pid without --fork**: PID namespace requires a new process
   ```bash
   unshare --pid --fork bash
   ```

3. **Not seeing PID 1**: If you don't remount `/proc`, you'll still see host PIDs
   ```bash
   # Must remount /proc after creating PID namespace
   mount -t proc proc /proc
   ```

4. **Zombie processes**: PID 1 must reap zombie processes. Bash does this, but a proper init doesn't always.

5. **Signal handling**: PID 1 handles signals differently. Some signals are ignored by default.

## Understanding PID 1 Responsibilities

PID 1 (init process) has special responsibilities:

1. **Reaping zombies**: Must call `wait()` to clean up zombie processes
2. **Signal handling**: Handles signals sent to orphaned processes
3. **Shutdown**: Handles system shutdown signals

Test this:

```bash
# Create a process that becomes a zombie
( : & exec sleep 1 ) &
# The child exits, parent (sleep) continues
# PID 1 should reap the zombie
```

## Next Steps

In Step 2.2, we'll combine PID and mount namespaces properly, ensuring that `/proc` shows the correct process list for the PID namespace. This is essential for proper container isolation.
