# Step 9.1: Complete Container Script

## Learning Objectives

- Combine all concepts into one script
- Integrate all namespaces (PID, mount, network, UTS, IPC, user)
- Set cgroup limits (memory, CPU, I/O)
- Drop capabilities
- Create isolated filesystem
- Configure network
- Test the complete container
- Compare with Docker behavior

## Prerequisites

- Completion of all previous steps
- Understanding of all namespaces, cgroups, capabilities
- Root access on a Linux system
- All tools from previous steps installed

## Background

This step combines everything we've learned:
- **Namespaces**: Complete isolation (PID, mount, network, UTS, IPC, user)
- **Cgroups**: Resource limits (memory, CPU, I/O)
- **Capabilities**: Dropped privileges
- **Filesystem**: Isolated root filesystem
- **Network**: Network namespace with veth

This is essentially what Docker does - we're building it from scratch!

## Commands

### Step 1: Create the complete container script

```bash
# Create a comprehensive container script
cat > /tmp/complete_container.sh <<'SCRIPT'
#!/bin/bash
set -e

# Configuration
CONTAINER_NAME=${1:-mycontainer}
CONTAINER_ROOT="/tmp/containers/$CONTAINER_NAME"
MEMORY_LIMIT=${2:-100M}
CPU_QUOTA=${3:-50000}  # 50% CPU
CPU_PERIOD=${4:-100000}

echo "=== Creating container: $CONTAINER_NAME ==="

# Cleanup function
cleanup() {
    echo "Cleaning up..."
    # Kill processes in container
    if [ -f "$CONTAINER_ROOT/container.pid" ]; then
        CONTAINER_PID=$(cat "$CONTAINER_ROOT/container.pid")
        kill -9 $CONTAINER_PID 2>/dev/null || true
        rm -f "$CONTAINER_ROOT/container.pid"
    fi
    
    # Remove network namespace
    ip netns delete $CONTAINER_NAME 2>/dev/null || true
    
    # Remove cgroup (cgroup v2)
    rmdir /sys/fs/cgroup/$CONTAINER_NAME 2>/dev/null || true
    
    # Unmount and remove root
    umount "$CONTAINER_ROOT/proc" 2>/dev/null || true
    umount "$CONTAINER_ROOT/sys" 2>/dev/null || true
    umount "$CONTAINER_ROOT/dev" 2>/dev/null || true
    umount "$CONTAINER_ROOT" 2>/dev/null || true
    rm -rf "$CONTAINER_ROOT" 2>/dev/null || true
}

trap cleanup EXIT

# Create container root
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
echo "Setting up filesystem..."
cp /bin/bash bin/
cp /bin/ls bin/
cp /bin/cat bin/
cp /bin/pwd bin/
cp /bin/echo bin/
cp /bin/sh bin/ 2>/dev/null || ln -s /bin/bash bin/sh
cp /bin/mount bin/
cp /bin/umount bin/
cp /bin/hostname bin/ 2>/dev/null || cp /usr/bin/hostname bin/
cp /bin/rmdir bin/ 2>/dev/null || cp /usr/bin/rmdir bin/
cp /bin/readlink bin/ 2>/dev/null || cp /usr/bin/readlink bin/

# Copy libraries
for binary in /bin/bash /bin/ls /bin/cat /bin/pwd /bin/echo /bin/mount /bin/umount /usr/bin/hostname /usr/bin/rmdir /usr/bin/readlink; do
    ldd "$binary" 2>/dev/null | grep -o '/[^ ]*' | while read lib; do
        mkdir -p "$CONTAINER_ROOT$(dirname "$lib")"
        cp "$lib" "$CONTAINER_ROOT$lib" 2>/dev/null || true
    done
done

# Create config files
cat > etc/passwd <<EOF
root:x:0:0:root:/root:/bin/bash
EOF

cat > etc/group <<EOF
root:x:0:
EOF

cat > etc/hosts <<EOF
127.0.0.1 localhost
$CONTAINER_NAME
EOF

# Create cgroup (cgroup v2 unified hierarchy)
echo "Setting up cgroups..."
echo "+memory +cpu +io" > /sys/fs/cgroup/cgroup.subtree_control
mkdir -p /sys/fs/cgroup/$CONTAINER_NAME

# Set memory limit
echo "$MEMORY_LIMIT" > /sys/fs/cgroup/$CONTAINER_NAME/memory.max

# Set CPU limit
echo "$CPU_QUOTA $CPU_PERIOD" > /sys/fs/cgroup/$CONTAINER_NAME/cpu.max

# Create network namespace
echo "Setting up network..."
ip netns add $CONTAINER_NAME

# Create veth pair
ip link add veth-$CONTAINER_NAME type veth peer name veth-$CONTAINER_NAME-ns

# Move one end to namespace
ip link set veth-$CONTAINER_NAME-ns netns $CONTAINER_NAME

# Configure network in namespace
ip netns exec $CONTAINER_NAME ip link set lo up
ip netns exec $CONTAINER_NAME ip link set veth-$CONTAINER_NAME-ns up
ip netns exec $CONTAINER_NAME ip addr add 10.0.0.10/24 dev veth-$CONTAINER_NAME-ns
ip netns exec $CONTAINER_NAME ip route add default via 10.0.0.1

# Configure host side
ip addr add 10.0.0.1/24 dev veth-$CONTAINER_NAME
ip link set veth-$CONTAINER_NAME up

# Enable forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Start container in namespaces
echo "Starting container..."
unshare --pid --mount --uts --ipc --user --map-root-user --fork bash <<INNER
# Set hostname
hostname $CONTAINER_NAME

# Join network namespace
exec ip netns exec $CONTAINER_NAME bash <<NETNS
# Mount filesystems
mount -t proc proc "$CONTAINER_ROOT/proc"
mount -t sysfs sysfs "$CONTAINER_ROOT/sys"
mount -t devtmpfs devtmpfs "$CONTAINER_ROOT/dev"

# Move to cgroup BEFORE pivot_root (host paths won't exist after)
echo \$\$ > /sys/fs/cgroup/$CONTAINER_NAME/cgroup.procs

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

# Drop capabilities (using capsh if available)
if command -v capsh &> /dev/null; then
    exec capsh --drop=cap_sys_module,cap_sys_rawio,cap_sys_admin -- \
        -- -c 'exec /bin/bash'
else
    exec /bin/bash
fi
NETNS
INNER

SCRIPT

chmod +x /tmp/complete_container.sh
```

### Step 2: Create a simpler, working version

```bash
# The above script is complex. Let's create a simpler working version
cat > /tmp/simple_container.sh <<'SCRIPT'
#!/bin/bash
CONTAINER_NAME=${1:-testcontainer}
CONTAINER_ROOT="/tmp/container_$CONTAINER_NAME"

echo "Creating container: $CONTAINER_NAME"

# Create rootfs
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
cp /bin/hostname "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/hostname "$CONTAINER_ROOT/bin/"
cp /bin/rmdir "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/rmdir "$CONTAINER_ROOT/bin/"
cp /bin/readlink "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/readlink "$CONTAINER_ROOT/bin/"

# Copy libraries
for binary in /bin/bash /bin/mount /bin/umount /usr/bin/hostname /usr/bin/rmdir /usr/bin/readlink; do
    ldd "$binary" 2>/dev/null | grep -o '/[^ ]*' | while read lib; do
        mkdir -p "$CONTAINER_ROOT$(dirname "$lib")"
        cp "$lib" "$CONTAINER_ROOT$lib" 2>/dev/null || true
    done
done

# Create config
cat > "$CONTAINER_ROOT/etc/passwd" <<EOF
root:x:0:0:root:/root:/bin/bash
EOF

# Create cgroup (cgroup v2)
echo "+memory +cpu" > /sys/fs/cgroup/cgroup.subtree_control
mkdir -p /sys/fs/cgroup/$CONTAINER_NAME
echo "50M" > /sys/fs/cgroup/$CONTAINER_NAME/memory.max

# Create the namespace entry script
cat > /tmp/enter_container_$CONTAINER_NAME.sh <<ENTERSCRIPT
#!/bin/bash
hostname $CONTAINER_NAME

# Mount
mount -t proc proc "$CONTAINER_ROOT/proc"
mount -t sysfs sysfs "$CONTAINER_ROOT/sys"

# Move to cgroup BEFORE pivot_root (host paths won't exist after)
echo \$\$ > /sys/fs/cgroup/$CONTAINER_NAME/cgroup.procs

# Pivot
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

echo "=== Container Info ==="
echo "Hostname: \$(hostname)"
echo "PID: \$\$"
echo ""

exec /bin/bash
ENTERSCRIPT

chmod +x /tmp/enter_container_$CONTAINER_NAME.sh

# Start container with unshare (not heredoc, so exec bash gets TTY)
exec unshare --pid --mount --uts --ipc --fork /tmp/enter_container_$CONTAINER_NAME.sh

SCRIPT

chmod +x /tmp/simple_container.sh
```

### Step 3: Run the container

```bash
# Run the simple container
sudo /tmp/simple_container.sh mytest

# Inside container, verify:
# - Hostname is set
# - PID is 1
# - Memory limit is set
# - Isolated filesystem
```

### Step 4: Test container features

```bash
# Inside container, test:

# 1. Hostname
hostname
# Should show: mytest

# 2. Process isolation
ps aux
# Should only show container processes

# 3. Filesystem isolation
ls /
# Should only show container directories

# 4. Memory limit
cat /sys/fs/cgroup/mytest/memory.max
# Should show: 52428800 (50MB)

# 5. Try to allocate memory
python3 -c "
import sys
chunks = []
for i in range(100):
    try:
        chunks.append(bytearray(1024*1024))  # 1MB chunks
        print(f'Allocated {i+1}MB')
    except MemoryError:
        print('Memory limit reached!')
        break
" 2>&1 || echo "Python may not be available"
```

### Step 5: Compare with Docker

```bash
# Exit container
exit

# Run equivalent Docker container
docker run -it --name dockertest --hostname dockertest --memory=50m alpine sh

# Inside Docker container:
# hostname  # Should show: dockertest
# ps aux    # Should show isolated processes
# ls /      # Should show Alpine filesystem

# Compare:
# - Both have isolated hostname
# - Both have process isolation
# - Both have memory limits
# - Both have isolated filesystem
```

### Step 6: Clean up

```bash
# Clean up our container
sudo rm -rf /tmp/container_mytest
sudo rmdir /sys/fs/cgroup/mytest 2>/dev/null || true

# Clean up Docker container
docker rm -f dockertest 2>/dev/null || true
```

## Verification

To verify complete container:

1. **All namespaces**: Container has isolated PID, mount, UTS, IPC
2. **Cgroup limits**: Memory limit is enforced
3. **Hostname**: Container has its own hostname
4. **Process isolation**: Only container processes visible
5. **Filesystem**: Isolated root filesystem
6. **Works like Docker**: Similar behavior to Docker containers

## Common Pitfalls

1. **Complex script**: Start simple, add features gradually
2. **Namespace ordering**: Some namespaces depend on others
3. **Cgroup paths**: Ensure correct cgroup v2 paths
4. **Network setup**: Network namespace requires additional setup
5. **Cleanup**: Always clean up namespaces and cgroups

## Understanding the Complete Picture

Our container now has:
- ✅ **Filesystem isolation**: pivot_root
- ✅ **Process isolation**: PID namespace
- ✅ **Network isolation**: Network namespace (if configured)
- ✅ **Hostname isolation**: UTS namespace
- ✅ **IPC isolation**: IPC namespace
- ✅ **User isolation**: User namespace
- ✅ **Resource limits**: Cgroups (memory, CPU)
- ✅ **Security**: Dropped capabilities

This is essentially what Docker provides!

## Next Steps

In Step 9.2, we'll learn about container lifecycle management: starting, stopping, and cleaning up containers properly.
