# Step 6.1: Cgroup v2 Basics

## Learning Objectives

- Understand cgroup hierarchy and controllers
- Navigate the cgroup filesystem
- Create a cgroup
- Move a process to a cgroup
- Understand cgroup v2 structure
- Understand controllers (memory, CPU, I/O)

## Prerequisites

- Completion of Step 5.2 (User Namespace with Filesystem)
- Basic understanding of Linux processes
- Root access on a Linux system
- System with cgroup v2 (most modern Linux systems)

## Background

### What are Cgroups?

Cgroups (control groups) are a Linux kernel feature that limit and account for resource usage. They allow you to:
- **Memory**: Limit memory usage per group
- **CPU**: Limit CPU usage per group
- **I/O**: Limit disk I/O bandwidth per group
- **Processes**: Limit number of processes per group
- **Device Access**: Control device access per group

### How Cgroups Provide Resource Control

Cgroups organize processes into hierarchical groups and apply resource limits:

1. **Hierarchical Organization**: Processes are organized in a tree structure:
   - Root cgroup contains all processes
   - Child cgroups contain subsets of processes
   - Limits apply to cgroup and all its children

2. **Resource Controllers**: Each cgroup can have controllers enabled:
   - **memory**: Memory limits and accounting
   - **cpu**: CPU limits and scheduling
   - **io**: I/O bandwidth limits
   - **pids**: Process count limits
   - **cpuset**: CPU and memory node assignment

3. **Process Assignment**: Processes are assigned to cgroups:
   - Process can only be in one cgroup (leaf cgroup)
   - Moving process to cgroup applies its limits
   - Child processes inherit parent's cgroup

4. **Resource Limits**: Limits are enforced at cgroup level:
   - Memory limit: `memory.max` (cgroup v2)
   - CPU limit: `cpu.max` (cgroup v2)
   - I/O limit: `io.max` (cgroup v2)
   - Process limit: `pids.max` (cgroup v2)

### Cgroup v2 Features

Cgroup v2 (unified hierarchy) features:
- **Unified Hierarchy**: Single tree structure (simpler than v1)
- **Per-cgroup Controllers**: Controllers enabled per cgroup
- **`cgroup.controllers`**: Shows available controllers for cgroup
- **`cgroup.subtree_control`**: Enables controllers for child cgroups
- **Better Resource Management**: More consistent and predictable

### Cgroup Filesystem

Cgroup filesystem is typically mounted at `/sys/fs/cgroup` (unified for v2):
- Each directory is a cgroup
- Files in directory control cgroup behavior
- `cgroup.procs`: Processes in cgroup
- `cgroup.controllers`: Available controllers
- Controller-specific files: `memory.max`, `cpu.max`, etc.

### How Docker Uses Cgroups

This is how Docker limits container resources:
- Each container gets its own cgroup
- Docker sets limits in cgroup files
- Processes in container are moved to cgroup
- Limits are enforced by kernel
- Resource usage is accounted per container

Example:
- Container memory limit: `echo "100M" > /sys/fs/cgroup/docker/<container-id>/memory.max`
- Container CPU limit: `echo "50000 100000" > /sys/fs/cgroup/docker/<container-id>/cpu.max`
- Processes moved: `echo $PID > /sys/fs/cgroup/docker/<container-id>/cgroup.procs`

## Key Commands Explained

Before diving into the practical steps, let's understand how cgroups work:

### Cgroup Filesystem Structure

Cgroups are managed through a filesystem interface at `/sys/fs/cgroup`:

**Key Files**:
- `cgroup.controllers`: Lists available controllers for this cgroup
- `cgroup.subtree_control`: Controllers enabled for child cgroups
- `cgroup.procs`: Processes in this cgroup (write PID to add process)
- `cgroup.threads`: Threads in this cgroup
- `cgroup.events`: Cgroup events (populated, frozen, etc.)

**Controller-specific Files** (when controller enabled):
- `memory.max`: Memory limit
- `memory.current`: Current memory usage
- `cpu.max`: CPU limit (format: "quota period")
- `cpu.stat`: CPU statistics
- `io.max`: I/O bandwidth limits
- `pids.max`: Process limit

### Creating Cgroups

Cgroups are created by creating directories:
```bash
# Create a cgroup
mkdir /sys/fs/cgroup/mycontainer

# This creates a new cgroup
# It inherits controllers from parent
```

**How it works**:
- Creating directory creates cgroup
- Cgroup inherits controllers from parent
- Must enable controllers for children using `cgroup.subtree_control`

### Enabling Controllers

Controllers must be enabled for child cgroups:
```bash
# Enable memory and CPU controllers for children
echo "+memory +cpu" > /sys/fs/cgroup/cgroup.subtree_control

# Now child cgroups can use these controllers
```

**Important**:
- `cgroup.subtree_control` enables controllers for **children**, not current cgroup
- To use controllers in current cgroup, parent must enable them
- Controllers are inherited down the hierarchy

### Moving Processes to Cgroups

Processes are moved to cgroups by writing PID to `cgroup.procs`:
```bash
# Move process to cgroup
echo $PID > /sys/fs/cgroup/mycontainer/cgroup.procs

# Process is now in cgroup
# Resource limits apply immediately
```

**How it works**:
- Writing PID moves process to cgroup
- Process and all its threads move together
- Child processes inherit parent's cgroup
- Limits apply immediately

### Reading Cgroup Information

Cgroup information is read from files:
```bash
# Check which cgroup process is in
cat /proc/$PID/cgroup
# Output: 0::/mycontainer

# Check available controllers
cat /sys/fs/cgroup/mycontainer/cgroup.controllers

# Check processes in cgroup
cat /sys/fs/cgroup/mycontainer/cgroup.procs
```

**Key files**:
- `/proc/<pid>/cgroup`: Shows cgroup path for process
- `cgroup.controllers`: Available controllers
- `cgroup.procs`: Processes in cgroup
- Controller files: Resource usage and limits

## Commands

### Step 1: Check cgroup version and mount

```bash
# Check if cgroup v2 is mounted
mount | grep cgroup || cat /proc/mounts | grep cgroup
# Should show cgroup2 mounted at /sys/fs/cgroup
# If nothing is shown, cgroups may not be properly mounted

# Verify cgroup filesystem exists
if [ ! -d /sys/fs/cgroup ]; then
    echo "Error: /sys/fs/cgroup does not exist. Cgroups may not be available."
    exit 1
fi

# Check cgroup version
stat -fc %T /sys/fs/cgroup
# Should show: cgroup2fs

# Alternative check
cat /proc/filesystems | grep cgroup
# Should show: nodev cgroup2

# If cgroup v2 is not available, you may need to:
# - Boot with cgroup v2 enabled (systemd.unified_cgroup_hierarchy=1)
# - Or use a system that defaults to cgroup v2 (most modern distributions)
```

### Step 2: Navigate cgroup filesystem

```bash
# Go to cgroup root
cd /sys/fs/cgroup

# List contents
ls -la
# Should show various files and directories

# Check available controllers
cat cgroup.controllers
# Should show: memory cpuset cpu io pids (or similar)

# Check enabled controllers for this cgroup
cat cgroup.subtree_control
# May be empty or show enabled controllers
```

### Step 3: Check current cgroup

```bash
# Check which cgroup current process is in
cat /proc/self/cgroup
# Should show: 0::/ (for cgroup v2)

# Get full path
CGROUP_PATH=$(cat /proc/self/cgroup | cut -d: -f3)
echo "Current cgroup: $CGROUP_PATH"

# Check cgroup path
ls -la /sys/fs/cgroup$CGROUP_PATH
```

### Step 4: Create a cgroup

```bash
# Ensure root access
sudo su

# Create a new cgroup
cd /sys/fs/cgroup
mkdir mycontainer

# Verify it was created
ls -d mycontainer
# Should exist

# Check its controllers
cat mycontainer/cgroup.controllers
# Should show available controllers

# Check if controllers are enabled for children
cat mycontainer/cgroup.subtree_control
# Should be empty initially
```

### Step 5: Enable controllers (parent) and keep a leaf for processes

> **⚠️ cgroup v2 leaf rule:** A cgroup that has controllers listed in `cgroup.subtree_control` is a **non-leaf** and **cannot** contain processes (`echo PID > cgroup.procs` → `Device or resource busy`).  
> Enable controllers on the **parent** so children can use them; put processes only in **leaf** cgroups (empty `subtree_control`).

```bash
# Enable memory and CPU for children of the root cgroup
cd /sys/fs/cgroup
# Controllers may already be enabled by systemd; ignore no-op failures
echo "+memory +cpu" > cgroup.subtree_control 2>/dev/null || true

# mycontainer inherits available controllers from parent
cat mycontainer/cgroup.controllers
# Should include: memory cpu ...

# Keep mycontainer as a LEAF for now (do NOT write subtree_control yet)
cat mycontainer/cgroup.subtree_control
# Should be empty
```

### Step 6: Move a process to cgroup

```bash
# Start a test process
sleep 1000 &
SLEEP_PID=$!
echo "Sleep PID: $SLEEP_PID"

# Check its current cgroup
cat /proc/$SLEEP_PID/cgroup
# Should show current cgroup

# Move process to our leaf cgroup
echo $SLEEP_PID > /sys/fs/cgroup/mycontainer/cgroup.procs

# Verify it was moved
cat /proc/$SLEEP_PID/cgroup
# Should show: 0::/mycontainer

# Check processes in cgroup
cat /sys/fs/cgroup/mycontainer/cgroup.procs
# Should include $SLEEP_PID
```

### Step 7: Understand cgroup hierarchy

```bash
# To create children, mycontainer must become a non-leaf:
# 1) Move any processes out of mycontainer first
kill $SLEEP_PID 2>/dev/null || true

# 2) Enable controllers for children
cd /sys/fs/cgroup/mycontainer
echo "+memory +cpu" > cgroup.subtree_control

# 3) Create nested leaf cgroups (these can hold processes)
mkdir container1
mkdir container2

# Check hierarchy
tree -L 2 /sys/fs/cgroup/mycontainer 2>/dev/null || \
find /sys/fs/cgroup/mycontainer -type d | head -10

# Move process to nested leaf cgroup
sleep 2000 &
SLEEP_PID2=$!
echo $SLEEP_PID2 > container1/cgroup.procs

# Verify
cat /proc/$SLEEP_PID2/cgroup
# Should show: 0::/mycontainer/container1
```

### Step 8: Check cgroup statistics

```bash
# Check memory usage (if memory controller enabled)
cat /sys/fs/cgroup/mycontainer/memory.current
# Shows current memory usage in bytes

# Check memory limit (if set)
cat /sys/fs/cgroup/mycontainer/memory.max
# Should show: max (unlimited) or a number

# Check CPU usage (if cpu controller enabled)
cat /sys/fs/cgroup/mycontainer/cpu.stat
# Shows CPU statistics

# Check number of processes
cat /sys/fs/cgroup/mycontainer/cgroup.procs | wc -l
# Shows number of processes
```

### Step 9: Understand controller availability

```bash
# Check what controllers are available system-wide
cat /sys/fs/cgroup/cgroup.controllers
# Lists all available controllers

# Check what controllers are available in our cgroup
cat /sys/fs/cgroup/mycontainer/cgroup.controllers
# Should show controllers available to this cgroup

# Enable a controller for children
cd /sys/fs/cgroup/mycontainer
echo "+io" > cgroup.subtree_control

# Verify
cat cgroup.subtree_control
# Should include io
```

### Step 10: Clean up

```bash
# Kill test processes
kill $SLEEP_PID $SLEEP_PID2 2>/dev/null || true

# Remove cgroups (must be empty)
# First, move any remaining processes out
# Then remove
rmdir /sys/fs/cgroup/mycontainer/container1
rmdir /sys/fs/cgroup/mycontainer/container2
rmdir /sys/fs/cgroup/mycontainer

# Verify
ls /sys/fs/cgroup/mycontainer
# Should show: No such file or directory
```

## Verification

To verify cgroup basics:

1. **Cgroup v2**: System should use cgroup v2 (check mount)
2. **Cgroup created**: Can create directories in `/sys/fs/cgroup`
3. **Process moved**: Process appears in `cgroup.procs` file
4. **Controllers available**: `cgroup.controllers` shows available controllers
5. **Hierarchy**: Can create nested cgroups

## Common Pitfalls

1. **Cgroup v1 vs v2**: Ensure system uses cgroup v2
   ```bash
   mount | grep cgroup
   # Should show cgroup2
   ```

2. **Controllers not enabled / leaf rule**: Enable controllers on the **parent**. Do not enable `subtree_control` on a cgroup that still has processes.
   ```bash
   echo "+memory +cpu" > /sys/fs/cgroup/cgroup.subtree_control
   # Processes go in leaf children only after parent has subtree_control set
   ```

3. **Cgroup not empty / Device or resource busy**: Cannot remove a cgroup with processes; cannot put processes in a non-leaf cgroup
   ```bash
   # Move processes out first
   echo $PID > /sys/fs/cgroup/user.slice/cgroup.procs
   ```
4. **Wrong path**: Use full path or relative to `/sys/fs/cgroup`
   ```bash
   echo $PID > /sys/fs/cgroup/mycontainer/cgroup.procs
   ```

5. **Permission denied**: Need root to create cgroups
   ```bash
   sudo su
   ```

## Understanding Cgroup Hierarchy

Cgroup structure:
```
/sys/fs/cgroup (root)
├── cgroup.controllers (available controllers)
├── cgroup.subtree_control (enabled for children)
├── mycontainer/
│   ├── cgroup.procs (processes in this cgroup)
│   ├── cgroup.controllers (available controllers)
│   ├── memory.current (memory usage)
│   ├── memory.max (memory limit)
│   └── container1/ (nested cgroup)
```

Key points:
- Controllers enabled in parent are available to children
- Processes can only be in leaf cgroups (with some exceptions)
- Limits apply to the cgroup and all its children

## Next Steps

In Step 6.2, we'll learn how to set memory limits using cgroups. We'll configure `memory.max`, test limit enforcement, and trigger OOM (Out of Memory) conditions to see how cgroups protect the system.
