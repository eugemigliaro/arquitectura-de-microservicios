# Step 6.2: Memory Limits

## Learning Objectives

- Set memory limits for processes using cgroups
- Understand memory controller (`memory.max`)
- Test memory limit enforcement
- Trigger OOM (Out of Memory) condition
- Monitor memory usage
- Understand OOM killer behavior

## Prerequisites

- Completion of Step 6.1 (Cgroup v2 Basics)
- Understanding of cgroup hierarchy
- Root access on a Linux system
- System with cgroup v2 and memory controller

## Background

Memory limits in cgroups:
- **memory.max**: Hard limit on memory usage
- **memory.high**: Soft limit (throttles, doesn't kill)
- **memory.current**: Current memory usage
- **memory.events**: OOM and other memory events

When limit is exceeded:
- OOM (Out of Memory) killer may terminate processes
- Processes may be throttled (if using memory.high)
- System protects itself from memory exhaustion

This is how Docker limits container memory - each container gets a cgroup with `memory.max` set.

## Commands

### Step 1: Create a cgroup for memory limits

```bash
# Ensure root access
sudo su

# Create a cgroup
cd /sys/fs/cgroup
mkdir memtest

# Enable memory controller (in parent)
echo "+memory" > cgroup.subtree_control

# Verify memory controller is available
cat memtest/cgroup.controllers
# Should include: memory
```

### Step 2: Set a memory limit

```bash
# Set memory limit to 100MB (104857600 bytes)
echo "100M" > memtest/memory.max

# Verify limit
cat memtest/memory.max
# Should show: 104857600

# Alternative: Set in bytes directly
# echo "104857600" > memtest/memory.max

# Check current usage
cat memtest/memory.current
# Should show: 0 (no processes yet)
```

### Step 3: Create a memory-intensive program

```bash
# Create a program that allocates memory
cat > /tmp/alloc_memory.py <<'EOF'
#!/usr/bin/env python3
import sys
import time

# Allocate memory in chunks
chunks = []
chunk_size = 10 * 1024 * 1024  # 10MB chunks
target_mb = int(sys.argv[1]) if len(sys.argv) > 1 else 50

print(f"Allocating {target_mb}MB of memory...")
for i in range(target_mb // 10):
    try:
        chunk = bytearray(chunk_size)
        chunks.append(chunk)
        print(f"Allocated {(i+1)*10}MB...")
        time.sleep(0.5)
    except MemoryError:
        print("MemoryError: Cannot allocate more memory")
        break

print("Memory allocated. Holding for 10 seconds...")
time.sleep(10)
print("Done")
EOF

chmod +x /tmp/alloc_memory.py
```

### Step 4: Test memory limit enforcement

```bash
# Start process in cgroup
python3 /tmp/alloc_memory.py 150 &
MEM_PID=$!
echo "Memory test PID: $MEM_PID"

# Move process to cgroup BEFORE it allocates too much
echo $MEM_PID > memtest/cgroup.procs

# Monitor memory usage
watch -n 1 'echo "Current: $(cat /sys/fs/cgroup/memtest/memory.current) bytes ($(($(cat /sys/fs/cgroup/memtest/memory.current) / 1024 / 1024))MB)"; echo "Limit: $(cat /sys/fs/cgroup/memtest/memory.max) bytes ($(($(cat /sys/fs/cgroup/memtest/memory.max) / 1024 / 1024))MB)"; echo "Events: $(cat /sys/fs/cgroup/memtest/memory.events)"'
# Press Ctrl+C to stop watching

# Or check manually
cat memtest/memory.current
cat memtest/memory.max
cat memtest/memory.events
```

### Step 5: Trigger OOM condition

```bash
# Set a very low limit (10MB)
echo "10M" > memtest/memory.max

# Start process that tries to allocate more
python3 /tmp/alloc_memory.py 50 &
MEM_PID=$!

# Move to cgroup
echo $MEM_PID > memtest/cgroup.procs

# Wait a moment
sleep 2

# Check if process was killed
ps -p $MEM_PID
# May show process killed

# Check OOM events
cat memtest/memory.events
# Should show oom_kill count increased
```

### Step 6: Monitor memory statistics

```bash
# Check detailed memory statistics
cat memtest/memory.stat
# Shows various memory statistics

# Check memory events
cat memtest/memory.events
# Shows: low high max oom oom_kill

# Check peak memory usage
cat memtest/memory.peak
# Shows peak memory usage

# Check memory limit
cat memtest/memory.max
# Shows current limit
```

### Step 7: Test with multiple processes

```bash
# Set limit to 50MB
echo "50M" > memtest/memory.max

# Start multiple processes in cgroup
for i in {1..5}; do
    python3 /tmp/alloc_memory.py 20 &
    MEM_PID=$!
    echo $MEM_PID > memtest/cgroup.procs
    sleep 1
done

# Monitor total memory usage
watch -n 1 'cat /sys/fs/cgroup/memtest/memory.current; echo "Limit: $(cat /sys/fs/cgroup/memtest/memory.max)"'

# Check which processes are in cgroup
cat memtest/cgroup.procs
ps -p $(cat memtest/cgroup.procs | tr '\n' ' ')
```

### Step 8: Use memory.high (soft limit)

```bash
# Set soft limit (throttles instead of killing)
echo "30M" > memtest/memory.high

# Set hard limit higher
echo "50M" > memtest/memory.max

# Start process
python3 /tmp/alloc_memory.py 40 &
MEM_PID=$!
echo $MEM_PID > memtest/cgroup.procs

# Monitor - process should be throttled at 30MB, not killed
# until it reaches 50MB
cat memtest/memory.events
# Should show high events (throttling)
```

### Step 9: Test OOM killer behavior

```bash
# Create a script to test OOM
cat > /tmp/test_oom.sh <<'EOF'
#!/bin/bash
CGROUP="/sys/fs/cgroup/memtest"

# Set very low limit
echo "20M" > $CGROUP/memory.max

# Function to allocate memory
allocate_memory() {
    local mb=$1
    local data=$(dd if=/dev/zero bs=1M count=$mb 2>/dev/null | base64)
    echo "Allocated ${mb}MB"
    sleep 5
}

# Start process in cgroup
allocate_memory 30 &
PID=$!
echo $PID > $CGROUP/cgroup.procs

# Monitor
while kill -0 $PID 2>/dev/null; do
    echo "Memory: $(cat $CGROUP/memory.current) / $(cat $CGROUP/memory.max)"
    echo "Events: $(cat $CGROUP/memory.events | grep oom)"
    sleep 1
done

echo "Process terminated"
cat $CGROUP/memory.events
EOF

chmod +x /tmp/test_oom.sh
/tmp/test_oom.sh
```

### Step 10: Clean up

```bash
# Kill any remaining processes
pkill -f alloc_memory.py
pkill -f test_oom.sh

# Wait a moment for cleanup
sleep 2

# Remove cgroup (must be empty)
rmdir /sys/fs/cgroup/memtest

# Verify
ls /sys/fs/cgroup/memtest
# Should show: No such file or directory
```

## Verification

To verify memory limits:

1. **Limit set**: `memory.max` shows the limit
2. **Usage tracked**: `memory.current` shows current usage
3. **Limit enforced**: Process killed or throttled when limit exceeded
4. **OOM events**: `memory.events` shows oom_kill count
5. **Statistics**: `memory.stat` shows detailed memory stats

## Common Pitfalls

1. **Moving process too late**: Move process to cgroup before it allocates memory
   ```bash
   # Start process, then immediately move
   process &
   PID=$!
   echo $PID > cgroup/cgroup.procs
   ```

2. **Memory controller not enabled**: Must enable in parent
   ```bash
   echo "+memory" > cgroup.subtree_control
   ```

3. **Limit too high**: Test with realistic limits
   ```bash
   echo "100M" > memory.max
   ```

4. **Not monitoring events**: Check `memory.events` to see what happened
   ```bash
   cat memory.events
   ```

5. **Process not in cgroup**: Verify process is in cgroup
   ```bash
   cat /proc/$PID/cgroup
   ```

## Understanding Memory Limits

Memory limit behavior:
- **memory.max**: Hard limit - OOM killer activates if exceeded
- **memory.high**: Soft limit - throttles allocation, doesn't kill
- **memory.current**: Current usage (may exceed limit briefly)
- **memory.peak**: Peak usage since creation

OOM killer:
- Kills processes when `memory.max` exceeded
- Chooses process to kill (usually the one using most memory)
- Logs to `memory.events` (oom_kill counter)

## Next Steps

In Step 6.3, we'll learn how to limit CPU usage using cgroups. We'll set CPU quotas and periods, test CPU throttling, and understand the difference between CPU shares and hard limits.
