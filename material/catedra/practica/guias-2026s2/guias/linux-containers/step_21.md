# Step 9.2: Container Lifecycle Management

## Learning Objectives

- Understand container lifecycle (create, start, stop, delete)
- Implement container start
- Implement container stop (signal handling)
- Implement container cleanup (remove namespaces, cgroups)
- Handle orphaned processes
- Create a container manager script

## Prerequisites

- Completion of Step 9.1 (Complete Container Script)
- Understanding of process management
- Understanding of signals
- Root access on a Linux system

## Background

Container lifecycle:
1. **Create**: Set up namespaces, cgroups, filesystem
2. **Start**: Launch the container process
3. **Stop**: Send signal to stop (SIGTERM, then SIGKILL)
4. **Delete**: Clean up namespaces, cgroups, filesystem

Important considerations:
- **Signal handling**: Properly handle SIGTERM/SIGINT
- **Orphaned processes**: Clean up child processes
- **Resource cleanup**: Remove namespaces, cgroups, mounts
- **State management**: Track container state

This is how Docker manages containers - create, start, stop, rm commands.

## Commands

### Step 1: Create container manager script structure

```bash
# Create a container manager script
cat > /tmp/container_manager.sh <<'SCRIPT'
#!/bin/bash

CONTAINER_NAME=""
CONTAINER_ROOT=""
CONTAINER_PID=""
STATE_FILE=""

# Functions
create_container() {
    echo "Creating container: $CONTAINER_NAME"
    # Implementation here
}

start_container() {
    echo "Starting container: $CONTAINER_NAME"
    # Implementation here
}

stop_container() {
    echo "Stopping container: $CONTAINER_NAME"
    # Implementation here
}

delete_container() {
    echo "Deleting container: $CONTAINER_NAME"
    # Implementation here
}

list_containers() {
    echo "Listing containers..."
    # Implementation here
}

# Main
case "$1" in
    create)
        CONTAINER_NAME=$2
        create_container
        ;;
    start)
        CONTAINER_NAME=$2
        start_container
        ;;
    stop)
        CONTAINER_NAME=$2
        stop_container
        ;;
    delete)
        CONTAINER_NAME=$2
        delete_container
        ;;
    list)
        list_containers
        ;;
    *)
        echo "Usage: $0 {create|start|stop|delete|list} [container_name]"
        exit 1
        ;;
esac
SCRIPT

chmod +x /tmp/container_manager.sh
```

### Step 2: Implement container creation

```bash
# Create a working container manager
cat > /tmp/ctr_manager.sh <<'SCRIPT'
#!/bin/bash
set -e

CONTAINERS_DIR="/tmp/containers"

mkdir -p "$CONTAINERS_DIR"

create_container() {
    local name=$1
    local root="$CONTAINERS_DIR/$name"
    
    if [ -d "$root" ]; then
        echo "Container $name already exists"
        return 1
    fi
    
    echo "Creating container: $name"
    mkdir -p "$root"/{bin,lib,lib64,etc,proc,sys,dev,tmp,root,.oldroot,state,usr/bin}

    # On Ubuntu 24.04, /bin is a symlink to /usr/bin. After pivot_root, bash
    # resolves commands via PATH to /usr/bin/. We mirror that structure with
    # a symlink so binaries are found regardless of PATH resolution.
    rm -rf "$root/usr/bin"
    ln -s ../../bin "$root/usr/bin"

    # Copy essential binaries
    cp /bin/bash "$root/bin/"
    cp /bin/mount "$root/bin/"
    cp /bin/umount "$root/bin/"
    cp /bin/rmdir "$root/bin/" 2>/dev/null || cp /usr/bin/rmdir "$root/bin/"
    cp /bin/hostname "$root/bin/" 2>/dev/null || cp /usr/bin/hostname "$root/bin/"
    cp /bin/readlink "$root/bin/" 2>/dev/null || cp /usr/bin/readlink "$root/bin/"

    # Copy libraries
    for binary in /bin/bash /bin/mount /bin/umount /usr/bin/rmdir /usr/bin/hostname /usr/bin/readlink; do
        ldd "$binary" 2>/dev/null | grep -o '/[^ ]*' | while read lib; do
            mkdir -p "$root$(dirname "$lib")"
            cp "$lib" "$root$lib" 2>/dev/null || true
        done
    done

    # Config
    cat > "$root/etc/passwd" <<EOF
root:x:0:0:root:/root:/bin/bash
EOF

    # State file
    echo "created" > "$root/state/status"
    echo "" > "$root/state/pid"

    # Cgroup (cgroup v2)
    echo "+memory +cpu" > /sys/fs/cgroup/cgroup.subtree_control
    mkdir -p /sys/fs/cgroup/$name
    echo "100M" > /sys/fs/cgroup/$name/memory.max
    
    echo "Container $name created"
}

start_container() {
    local name=$1
    local root="$CONTAINERS_DIR/$name"
    
    if [ ! -d "$root" ]; then
        echo "Container $name does not exist"
        return 1
    fi
    
    local status=$(cat "$root/state/status" 2>/dev/null || echo "unknown")
    if [ "$status" = "running" ]; then
        echo "Container $name is already running"
        return 1
    fi
    
    echo "Starting container: $name"
    
    # Start container in background.
    # IMPORTANT: inside a PID namespace, $$ is the *namespace* PID (often 1).
    # Storing that in state/pid breaks stop/kill on the host (PID 1 is systemd!).
    # After start, read the **host** PID from the cgroup (cgroup.procs uses host PIDs).
    (
        unshare --pid --mount --uts --ipc --fork bash <<INNER
        hostname $name
        mount --make-rprivate /

        mount -t proc proc "$root/proc"
        mount -t sysfs sysfs "$root/sys"
        mount -t tmpfs tmpfs "$root/dev"

        # Move to cgroup BEFORE pivot_root (host paths won't exist after)
        echo \$\$ > /sys/fs/cgroup/$name/cgroup.procs
        echo "running" > "$root/state/status"

        mount --bind "$root" "$root"
        mkdir -p "$root/.oldroot"
        cd "$root"
        pivot_root . .oldroot

        # Update working directory to new root
        cd /

        # Mount proc (needed before umount can resolve mount points)
        mount -t proc proc /proc
        mount -t sysfs sysfs /sys

        # Unmount old root
        umount -l /.oldroot
        rmdir /.oldroot

        # Remount /proc for PID namespace (lazy umount if busy)
        umount -l /proc 2>/dev/null || true
        mount -t proc proc /proc

        # Handle signals
        trap 'exit 0' TERM INT

        exec /bin/bash
INNER
    ) &

    sleep 1

    # Host PID = first process in the container cgroup
    local pid
    pid=$(head -n1 /sys/fs/cgroup/$name/cgroup.procs 2>/dev/null || true)
    if [ -n "$pid" ] && ps -p "$pid" > /dev/null 2>&1; then
        echo "$pid" > "$root/state/pid"
        echo "Container $name started (host PID: $pid)"
    else
        echo "Failed to start container $name"
        echo "stopped" > "$root/state/status"
        return 1
    fi
}

stop_container() {
    local name=$1
    local root="$CONTAINERS_DIR/$name"
    
    if [ ! -d "$root" ]; then
        echo "Container $name does not exist"
        return 1
    fi
    
    local pid=$(cat "$root/state/pid" 2>/dev/null || echo "")
    if [ -z "$pid" ] || ! ps -p $pid > /dev/null 2>&1; then
        echo "Container $name is not running"
        return 1
    fi
    
    echo "Stopping container: $name (PID: $pid)"
    
    # Send SIGTERM
    kill -TERM $pid 2>/dev/null || true
    sleep 2
    
    # If still running, send SIGKILL
    if ps -p $pid > /dev/null 2>&1; then
        echo "Force killing container..."
        kill -KILL $pid 2>/dev/null || true
    fi
    
    # Clean up processes in cgroup (cgroup v2)
    for p in $(cat /sys/fs/cgroup/$name/cgroup.procs 2>/dev/null); do
        kill -KILL $p 2>/dev/null || true
    done
    
    echo "stopped" > "$root/state/status"
    echo "" > "$root/state/pid"
    
    echo "Container $name stopped"
}

delete_container() {
    local name=$1
    local root="$CONTAINERS_DIR/$name"
    
    if [ ! -d "$root" ]; then
        echo "Container $name does not exist"
        return 1
    fi
    
    local status=$(cat "$root/state/status" 2>/dev/null || echo "unknown")
    if [ "$status" = "running" ]; then
        echo "Container is running. Stop it first."
        return 1
    fi
    
    echo "Deleting container: $name"
    
    # Remove cgroup (cgroup v2)
    rmdir /sys/fs/cgroup/$name 2>/dev/null || true
    
    # Unmount
    umount "$root/proc" 2>/dev/null || true
    umount "$root/sys" 2>/dev/null || true
    umount "$root/dev" 2>/dev/null || true
    umount "$root" 2>/dev/null || true
    
    # Remove root
    rm -rf "$root"
    
    echo "Container $name deleted"
}

list_containers() {
    echo "Containers:"
    for container in "$CONTAINERS_DIR"/*; do
        if [ -d "$container" ]; then
            local name=$(basename "$container")
            local status=$(cat "$container/state/status" 2>/dev/null || echo "unknown")
            local pid=$(cat "$container/state/pid" 2>/dev/null || echo "-")
            printf "  %-20s %-10s %s\n" "$name" "$status" "$pid"
        fi
    done
}

# Main
case "$1" in
    create)
        create_container "$2"
        ;;
    start)
        start_container "$2"
        ;;
    stop)
        stop_container "$2"
        ;;
    delete)
        delete_container "$2"
        ;;
    list)
        list_containers
        ;;
    *)
        echo "Usage: $0 {create|start|stop|delete|list} [container_name]"
        exit 1
        ;;
esac
SCRIPT

chmod +x /tmp/ctr_manager.sh
```

### Step 3: Test container lifecycle

```bash
# Ensure root
sudo su

# Create container
/tmp/ctr_manager.sh create test1

# List containers
/tmp/ctr_manager.sh list
# Should show: test1 created

# Start container
/tmp/ctr_manager.sh start test1

# List containers (should show running)
/tmp/ctr_manager.sh list

# In another terminal, check process
ps aux | grep test1

# Stop container
/tmp/ctr_manager.sh stop test1

# List containers (should show stopped)
/tmp/ctr_manager.sh list

# Delete container
/tmp/ctr_manager.sh delete test1

# List containers (should be empty)
/tmp/ctr_manager.sh list
```

### Step 4: Handle orphaned processes

```bash
# Create a script to clean up orphaned processes
cat > /tmp/cleanup_orphans.sh <<'EOF'
#!/bin/bash
# Find and clean up orphaned container processes

CONTAINERS_DIR="/tmp/containers"

for container_dir in "$CONTAINERS_DIR"/*; do
    if [ ! -d "$container_dir" ]; then
        continue
    fi
    
    name=$(basename "$container_dir")
    pid_file="$container_dir/state/pid"
    
    if [ -f "$pid_file" ]; then
        pid=$(cat "$pid_file")
        if [ ! -z "$pid" ] && ! ps -p $pid > /dev/null 2>&1; then
            echo "Cleaning up orphaned container: $name (PID: $pid)"
            echo "stopped" > "$container_dir/state/status"
            echo "" > "$pid_file"
        fi
    fi
done

# Clean up orphaned cgroups (cgroup v2 - check direct children of /sys/fs/cgroup)
for cgroup in /sys/fs/cgroup/*/; do
    name=$(basename "$cgroup")
    # Skip system cgroups
    case "$name" in
        system.slice|user.slice|init.scope|*.service) continue ;;
    esac
    if [ ! -d "$CONTAINERS_DIR/$name" ]; then
        echo "Cleaning up orphaned cgroup: $name"
        for pid in $(cat "$cgroup/cgroup.procs" 2>/dev/null); do
            kill -KILL $pid 2>/dev/null || true
        done
        rmdir "$cgroup" 2>/dev/null || true
    fi
done
EOF

chmod +x /tmp/cleanup_orphans.sh
```

### Step 5: Test signal handling

```bash
# Start a container
/tmp/ctr_manager.sh create sigtest
/tmp/ctr_manager.sh start sigtest

# Get PID
PID=$(cat /tmp/containers/sigtest/state/pid)

# Send SIGTERM
kill -TERM $PID

# Wait a moment
sleep 1

# Check status
cat /tmp/containers/sigtest/state/status
# Should show: stopped

# Clean up
/tmp/ctr_manager.sh delete sigtest
```

### Step 6: Compare with Docker lifecycle

```bash
# Docker lifecycle:
# docker create  -> Creates container (doesn't start)
# docker start   -> Starts container
# docker stop    -> Stops container (SIGTERM, then SIGKILL)
# docker rm      -> Deletes container
# docker ps      -> Lists running containers
# docker ps -a   -> Lists all containers

# Our lifecycle:
# ctr_manager.sh create  -> Creates container
# ctr_manager.sh start   -> Starts container
# ctr_manager.sh stop    -> Stops container
# ctr_manager.sh delete  -> Deletes container
# ctr_manager.sh list    -> Lists containers

# Similar concepts, different implementation!
```

### Step 7: Clean up

```bash
# Clean up all test containers
for container in test1 sigtest; do
    /tmp/ctr_manager.sh stop $container 2>/dev/null || true
    /tmp/ctr_manager.sh delete $container 2>/dev/null || true
done

# Run orphan cleanup
sudo /tmp/cleanup_orphans.sh

# Remove scripts
rm -f /tmp/container_manager.sh /tmp/ctr_manager.sh /tmp/cleanup_orphans.sh
```

## Verification

To verify container lifecycle:

1. **Create works**: Container directory and cgroup created
2. **Start works**: Container process starts, status is "running"
3. **Stop works**: Process receives signal, status is "stopped"
4. **Delete works**: Namespaces, cgroups, filesystem cleaned up
5. **List works**: Shows all containers with status
6. **Orphan cleanup**: Handles orphaned processes

## Common Pitfalls

1. **State management**: Track container state properly
   ```bash
   echo "running" > state/status
   ```

2. **PID tracking**: Store the **host** PID (from `cgroup.procs`), not `$$` inside the PID namespace
   ```bash
   # WRONG inside PID ns (often writes 1 = systemd on the host):
   # echo $$ > state/pid
   # RIGHT after start:
   head -n1 /sys/fs/cgroup/$name/cgroup.procs > state/pid
   ```

3. **Signal handling**: Handle SIGTERM gracefully
   ```bash
   trap 'cleanup; exit' TERM INT
   ```

4. **Cleanup order**: Unmount before removing directories
   ```bash
   umount proc sys dev
   rm -rf root
   ```

5. **Orphaned processes**: Clean up processes in cgroups

## Understanding Container Lifecycle

Container lifecycle stages:
1. **Created**: Filesystem and cgroups set up
2. **Running**: Process started in namespaces
3. **Stopped**: Process terminated, resources preserved
4. **Deleted**: All resources cleaned up

State management:
- Track status (created/running/stopped)
- Track PID for process management
- Clean up on exit

## Summary

We've now built a complete container system from scratch:
- ✅ All namespaces (isolation)
- ✅ Cgroups (resource limits)
- ✅ Capabilities (security)
- ✅ Filesystem (overlay, pivot_root)
- ✅ Network (namespaces, veth)
- ✅ Lifecycle management (create, start, stop, delete)

This is essentially what Docker does - we've recreated it manually!

## Next Steps

Congratulations! You've completed all the exercises. You now understand:
- How Linux namespaces provide isolation
- How cgroups limit resources
- How capabilities provide security
- How containers work at a low level
- How Docker abstracts these concepts

You can now:
- Debug container issues more effectively
- Understand Docker's internals
- Build custom container solutions
- Appreciate what container runtimes do
