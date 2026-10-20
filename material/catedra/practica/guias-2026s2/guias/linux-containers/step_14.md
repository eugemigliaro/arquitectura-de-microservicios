# Step 6.3: CPU Limits

## Learning Objectives

- Set CPU limits using cgroups
- Understand CPU controller (`cpu.max`)
- Set CPU quota and period
- Test CPU throttling
- Understand CPU shares vs hard limits
- Monitor CPU usage

## Prerequisites

- Completion of Step 6.2 (Memory Limits)
- Understanding of cgroup CPU controller
- Root access on a Linux system
- System with cgroup v2 and CPU controller

## Background

CPU limits in cgroups:
- **cpu.max**: CPU quota and period (e.g., "50000 100000" = 50% CPU)
- **cpu.weight**: CPU shares (relative priority)
- **cpu.stat**: CPU usage statistics

CPU limits work as:
- **Quota/Period**: Process can use quota microseconds every period
- Example: quota=50000, period=100000 = 50% CPU
- **Weight**: Relative CPU allocation among siblings

This is how Docker limits container CPU - each container gets a cgroup with CPU limits.

## Commands

### Step 1: Create a cgroup for CPU limits

```bash
# Ensure root access
sudo su

# Create a cgroup
cd /sys/fs/cgroup
mkdir cputest

# Enable CPU controller (in parent)
echo "+cpu" > cgroup.subtree_control

# Verify CPU controller is available
cat cputest/cgroup.controllers
# Should include: cpu
```

### Step 2: Set CPU limit using quota and period

```bash
# Set CPU limit to 50% (50000 microseconds out of 100000)
# Format: "quota period" or "max" for unlimited
echo "50000 100000" > cputest/cpu.max

# Verify limit
cat cputest/cpu.max
# Should show: 50000 100000

# This means: can use 50000 microseconds every 100000 microseconds = 50% CPU

# Alternative: 25% CPU
# echo "25000 100000" > cputest/cpu.max

# Unlimited CPU
# echo "max" > cputest/cpu.max
```

### Step 3: Create a CPU-intensive program

```bash
# Create a CPU-intensive program
cat > /tmp/cpu_intensive.py <<'EOF'
#!/usr/bin/env python3
import time
import sys

duration = int(sys.argv[1]) if len(sys.argv) > 1 else 30
print(f"Running CPU-intensive task for {duration} seconds...")

start = time.time()
iterations = 0
while time.time() - start < duration:
    # CPU-intensive calculation
    sum(range(10000))
    iterations += 1
    if iterations % 1000 == 0:
        elapsed = time.time() - start
        print(f"Iterations: {iterations}, Elapsed: {elapsed:.2f}s")

print(f"Completed {iterations} iterations")
EOF

chmod +x /tmp/cpu_intensive.py

# Alternative: Use stress-ng if available
# stress-ng --cpu 1 --timeout 30s
```

### Step 4: Test CPU limit enforcement

```bash
# Start CPU-intensive process
python3 /tmp/cpu_intensive.py 60 &
CPU_PID=$!
echo "CPU test PID: $CPU_PID"

# Move process to cgroup
echo $CPU_PID > cputest/cgroup.procs

# Monitor CPU usage (in another terminal or background)
# Using top or htop
top -p $CPU_PID

# Or use pidstat
# pidstat -p $CPU_PID 1

# Or check cgroup stats
watch -n 1 'echo "CPU usage: $(cat /sys/fs/cgroup/cputest/cpu.stat | grep usage_usec)"; echo "Throttled: $(cat /sys/fs/cgroup/cputest/cpu.stat | grep nr_throttled)"'
```

### Step 5: Check CPU statistics

```bash
# Check CPU statistics
cat cputest/cpu.stat
# Shows:
# - usage_usec: Total CPU time used (microseconds)
# - user_usec: User CPU time
# - system_usec: System CPU time
# - nr_periods: Number of periods
# - nr_throttled: Number of times throttled
# - throttled_usec: Total throttled time

# Monitor in real-time
watch -n 1 'cat /sys/fs/cgroup/cputest/cpu.stat'
```

### Step 6: Test different CPU limits

```bash
# Test 25% CPU limit
echo "25000 100000" > cputest/cpu.max
python3 /tmp/cpu_intensive.py 30 &
PID1=$!
echo $PID1 > cputest/cgroup.procs

# Test 75% CPU limit (in another cgroup)
mkdir cputest2
echo "+cpu" > cgroup.subtree_control
echo "75000 100000" > cputest2/cpu.max
python3 /tmp/cpu_intensive.py 30 &
PID2=$!
echo $PID2 > cputest2/cgroup.procs

# Compare CPU usage
# Both should be limited to their respective limits
```

### Step 7: Test CPU throttling

```bash
# Set very low limit (10% CPU)
echo "10000 100000" > cputest/cpu.max

# Start CPU-intensive process
python3 /tmp/cpu_intensive.py 30 &
CPU_PID=$!
echo $CPU_PID > cputest/cgroup.procs

# Monitor throttling
watch -n 1 'cat /sys/fs/cgroup/cputest/cpu.stat | grep -E "nr_throttled|throttled_usec"'

# Process should be throttled frequently
# Check that nr_throttled increases
```

### Step 8: Use CPU weight (shares)

```bash
# CPU weight is for relative allocation among siblings
# Higher weight = more CPU when competing

# Create two cgroups with different weights
mkdir cputest/weight1
mkdir cputest/weight2

# Set weights (default is 100, range 1-10000)
echo "100" > cputest/weight1/cpu.weight
echo "300" > cputest/weight2/cpu.weight

# Start processes in each
python3 /tmp/cpu_intensive.py 60 &
PID1=$!
echo $PID1 > cputest/weight1/cgroup.procs

python3 /tmp/cpu_intensive.py 60 &
PID2=$!
echo $PID2 > cputest/weight2/cgroup.procs

# weight2 should get 3x more CPU than weight1 (300/100 = 3:1 ratio)
# Monitor CPU usage of both processes
```

### Step 9: Compare quota vs weight

```bash
# Quota: Absolute limit (e.g., 50% CPU max)
echo "50000 100000" > cputest/cpu.max
# Process can never use more than 50% CPU

# Weight: Relative allocation (when competing for CPU)
echo "200" > cputest/cpu.weight
# Process gets CPU proportional to weight when CPU is contended

# Both can be used together:
# - cpu.max sets maximum CPU
# - cpu.weight sets relative allocation up to that maximum
```

### Step 10: Monitor CPU usage over time

```bash
# Create monitoring script
cat > /tmp/monitor_cpu.sh <<'EOF'
#!/bin/bash
CGROUP="/sys/fs/cgroup/cputest"

while true; do
    clear
    echo "=== CPU Cgroup Statistics ==="
    echo ""
    echo "Limit: $(cat $CGROUP/cpu.max)"
    echo ""
    cat $CGROUP/cpu.stat
    echo ""
    echo "Processes:"
    cat $CGROUP/cgroup.procs | xargs -I {} ps -p {} -o pid,pcpu,comm 2>/dev/null || echo "No processes"
    sleep 2
done
EOF

chmod +x /tmp/monitor_cpu.sh

# Run in background
/tmp/monitor_cpu.sh &
MONITOR_PID=$!

# Start CPU-intensive task
python3 /tmp/cpu_intensive.py 30 &
CPU_PID=$!
echo $CPU_PID > cputest/cgroup.procs

# Let it run, then stop monitor
sleep 35
kill $MONITOR_PID 2>/dev/null || true
```

### Step 11: Test with multiple CPUs

```bash
# If system has multiple CPUs, test per-CPU limits
# Note: cgroup v2 CPU controller works across all CPUs

# Set limit to 100% of one CPU (on 4-CPU system, this is 25% total)
# For 4 CPUs, 100% of one CPU = 25% of total
# quota = (desired_percent * period * num_cpus) / 100
# For 25% on 4 CPUs: (25 * 100000 * 4) / 100 = 100000

NUM_CPUS=$(nproc)
echo "System has $NUM_CPUS CPUs"

# Set to use 100% of one CPU
QUOTA=$((100000 * NUM_CPUS / NUM_CPUS))  # 100% of 1 CPU
echo "$QUOTA 100000" > cputest/cpu.max

# Or set to 50% of total CPU
QUOTA=$((50000 * NUM_CPUS))
echo "$QUOTA 100000" > cputest/cpu.max
```

### Step 12: Clean up

```bash
# Kill test processes
pkill -f cpu_intensive.py
pkill -f monitor_cpu.sh

# Wait for cleanup
sleep 2

# Remove cgroups (must be empty)
rmdir /sys/fs/cgroup/cputest/weight1
rmdir /sys/fs/cgroup/cputest/weight2
rmdir /sys/fs/cgroup/cputest2
rmdir /sys/fs/cgroup/cputest

# Verify
ls /sys/fs/cgroup/cputest
# Should show: No such file or directory
```

## Verification

To verify CPU limits:

1. **Limit set**: `cpu.max` shows quota and period
2. **Usage tracked**: `cpu.stat` shows usage_usec
3. **Throttling**: `nr_throttled` increases when limit exceeded
4. **CPU usage**: Process CPU% should not exceed limit
5. **Weight works**: Processes with higher weight get more CPU

## Common Pitfalls

1. **CPU controller not enabled**: Must enable in parent
   ```bash
   echo "+cpu" > cgroup.subtree_control
   ```

2. **Quota format**: Must be "quota period" or "max"
   ```bash
   echo "50000 100000" > cpu.max
   ```

3. **Not monitoring**: Use `cpu.stat` to verify limits work
   ```bash
   cat cpu.stat
   ```

4. **Weight vs quota**: Understand difference
   - Quota: Absolute limit
   - Weight: Relative allocation

5. **Multi-CPU systems**: Quota applies to total CPU, not per-core

## Understanding CPU Limits

CPU limit calculation:
- **Quota/Period**: `quota / period = CPU percentage`
- Example: `50000 / 100000 = 0.5 = 50% CPU`
- **Weight**: Relative when CPU is contended
- Example: weight 200 vs 100 = 2:1 ratio

CPU throttling:
- When quota exceeded, process is throttled
- `nr_throttled` counts throttling events
- `throttled_usec` shows total throttled time

## Next Steps

In Step 6.4, we'll learn how to limit disk I/O bandwidth using cgroups. We'll set read/write bandwidth limits, test I/O throttling, and monitor I/O statistics.
