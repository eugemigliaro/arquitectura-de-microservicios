# Step 6.4: I/O Limits

## Learning Objectives

- Set disk I/O bandwidth limits using cgroups
- Understand I/O controller (`io.max`)
- Set read/write bandwidth limits
- Test I/O throttling
- Monitor I/O statistics
- Understand I/O limits per device

## Prerequisites

- Completion of Step 6.3 (CPU Limits)
- Understanding of cgroup I/O controller
- Root access on a Linux system
- System with cgroup v2 and I/O controller
- A block device (disk) for testing

## Background

I/O limits in cgroups:
- **io.max**: Maximum I/O bandwidth (read/write) per device
- **io.stat**: I/O statistics (bytes read/written, I/Os)
- **io.weight**: I/O priority (relative allocation)

I/O limits work as:
- **Bandwidth**: Maximum bytes per second (read/write separately)
- **Per device**: Limits apply per block device
- **Throttling**: I/O is throttled when limit exceeded

This is how Docker limits container disk I/O - each container gets a cgroup with I/O limits.

## Commands

### Step 1: Check I/O controller availability

```bash
# Ensure root access
sudo su

# Check if I/O controller is available
cat /sys/fs/cgroup/cgroup.controllers | grep io
# Should show: io

# If not available, may need to enable
# Check kernel support
grep CONFIG_BLK_CGROUP /boot/config-$(uname -r) 2>/dev/null || \
echo "Check if block I/O controller is enabled in kernel"
```

### Step 2: Create a cgroup for I/O limits

```bash
# Create a cgroup
cd /sys/fs/cgroup
mkdir iotest

# Enable I/O controller (in parent)
echo "+io" > cgroup.subtree_control

# Verify I/O controller is available
cat iotest/cgroup.controllers
# Should include: io
```

### Step 3: Find block devices

```bash
# List block devices
lsblk
# Shows all block devices

# Find root device
ROOT_DEV=$(lsblk -n -o NAME,MOUNTPOINT | grep -E '/$' | awk '{print $1}' | head -1)
echo "Root device: $ROOT_DEV"

# Get major:minor numbers
ROOT_MAJMIN=$(lsblk -n -o MAJ:MIN,NAME | grep "^[0-9]*:[0-9]* $ROOT_DEV" | awk '{print $1}')
echo "Device major:minor: $ROOT_MAJMIN"

# Or use stat
stat -c "t%T" /dev/$ROOT_DEV
# Shows major:minor in hex
```

### Step 4: Set I/O bandwidth limit

```bash
# Set I/O limit: 10MB/s read, 5MB/s write
# Format: "major:minor rbps=limit wbps=limit"
# rbps = read bytes per second
# wbps = write bytes per second

# Get device major:minor dynamically
# (8:0 = /dev/sda, 252:x = virtio, 259:x = nvme — varies by system)
DEVICE=$(lsblk -n -o MAJ:MIN,MOUNTPOINT | grep ' /$' | awk '{print $1}' | head -1)
echo "Using device: $DEVICE"

# Set limit: 10MB/s read, 5MB/s write
echo "$DEVICE rbps=10485760 wbps=5242880" > iotest/io.max

# Verify limit
cat iotest/io.max
# Should show: 8:0 rbps=10485760 wbps=5242880

# Alternative: Limit only read or only write
# echo "$DEVICE rbps=10485760" > iotest/io.max  # Read only
# echo "$DEVICE wbps=5242880" > iotest/io.max   # Write only
```

### Step 5: Create I/O-intensive program

```bash
# Create a program that does heavy I/O
cat > /tmp/io_intensive.sh <<'EOF'
#!/bin/bash
TEST_FILE="/tmp/io_test_file"
SIZE_MB=${1:-100}

echo "Creating ${SIZE_MB}MB test file..."

# Write test
echo "Writing ${SIZE_MB}MB..."
dd if=/dev/zero of=$TEST_FILE bs=1M count=$SIZE_MB oflag=direct 2>&1

# Read test
echo "Reading ${SIZE_MB}MB..."
dd if=$TEST_FILE of=/dev/null bs=1M iflag=direct 2>&1

# Cleanup
rm -f $TEST_FILE
echo "Done"
EOF

chmod +x /tmp/io_intensive.sh

# Alternative: Use fio if available
# fio --name=test --ioengine=libaio --iodepth=4 --rw=write --bs=1M --size=100M
```

### Step 6: Test I/O limit enforcement

```bash
# Start I/O-intensive process
/tmp/io_intensive.sh 200 &
IO_PID=$!
echo "I/O test PID: $IO_PID"

# Move process to cgroup BEFORE it starts heavy I/O
echo $IO_PID > iotest/cgroup.procs

# Monitor I/O statistics
watch -n 1 'cat /sys/fs/cgroup/iotest/io.stat'
# Press Ctrl+C to stop

# Or check manually
cat iotest/io.stat
# Shows I/O statistics per device
```

### Step 7: Check I/O statistics

```bash
# Check I/O statistics
cat iotest/io.stat
# Format: major:minor rbytes wbytes rios wios dbytes dios
# rbytes: bytes read
# wbytes: bytes written
# rios: read I/O operations
# wios: write I/O operations
# dbytes: bytes discarded
# dios: discard I/O operations

# Monitor in real-time
watch -n 1 'cat /sys/fs/cgroup/iotest/io.stat | head -5'
```

### Step 8: Test read vs write limits separately

```bash
# Set different limits for read and write
DEVICE=$(lsblk -n -o MAJ:MIN,MOUNTPOINT | grep ' /$' | awk '{print $1}' | head -1)

# 20MB/s read, 10MB/s write
echo "$DEVICE rbps=20971520 wbps=10485760" > iotest/io.max

# Test write speed
dd if=/dev/zero of=/tmp/write_test bs=1M count=100 oflag=direct &
WRITE_PID=$!
echo $WRITE_PID > iotest/cgroup.procs

# Monitor write statistics
watch -n 1 'cat /sys/fs/cgroup/iotest/io.stat | grep "$DEVICE"'

# After write completes, test read speed
dd if=/tmp/write_test of=/dev/null bs=1M iflag=direct &
READ_PID=$!
echo $READ_PID > iotest/cgroup.procs

# Monitor read statistics
watch -n 1 'cat /sys/fs/cgroup/iotest/io.stat | grep "$DEVICE"'

# Cleanup
rm -f /tmp/write_test
```

### Step 9: Use I/O weight (priority)

```bash
# I/O weight sets relative I/O priority
# Higher weight = more I/O bandwidth when competing

# Create two cgroups with different weights
mkdir iotest/weight1
mkdir iotest/weight2

# Set weights (default is 100, range 1-10000)
echo "100" > iotest/weight1/io.weight
echo "300" > iotest/weight2/io.weight

# Start I/O processes in each
dd if=/dev/zero of=/tmp/test1 bs=1M count=500 oflag=direct &
PID1=$!
echo $PID1 > iotest/weight1/cgroup.procs

dd if=/dev/zero of=/tmp/test2 bs=1M count=500 oflag=direct &
PID2=$!
echo $PID2 > iotest/weight2/cgroup.procs

# weight2 should get 3x more I/O bandwidth than weight1
# Monitor I/O statistics
watch -n 1 'echo "Weight1:"; cat /sys/fs/cgroup/iotest/weight1/io.stat; echo "Weight2:"; cat /sys/fs/cgroup/iotest/weight2/io.stat'
```

### Step 10: Test I/O throttling

```bash
# Set very low limit (1MB/s)
DEVICE=$(lsblk -n -o MAJ:MIN,MOUNTPOINT | grep ' /$' | awk '{print $1}' | head -1)
echo "$DEVICE rbps=1048576 wbps=1048576" > iotest/io.max

# Start I/O-intensive task
dd if=/dev/zero of=/tmp/throttle_test bs=1M count=100 oflag=direct &
IO_PID=$!
echo $IO_PID > iotest/cgroup.procs

# Monitor - I/O should be throttled to ~1MB/s
# Use iotop or iostat to verify
# iotop -p $IO_PID
# iostat -x 1

# Check cgroup I/O stats
cat iotest/io.stat
```

### Step 11: Set limits for multiple devices

```bash
# If you have multiple block devices, set limits for each
# Format: one device per line in io.max

DEVICE1="8:0"   # e.g., /dev/sda
DEVICE2="8:16"   # e.g., /dev/sdb

# Set limits for both devices
cat > iotest/io.max <<EOF
$DEVICE1 rbps=10485760 wbps=5242880
$DEVICE2 rbps=20971520 wbps=10485760
EOF

# Verify
cat iotest/io.max
# Should show limits for both devices
```

### Step 12: Monitor I/O over time

```bash
# Create monitoring script
cat > /tmp/monitor_io.sh <<'EOF'
#!/bin/bash
CGROUP="/sys/fs/cgroup/iotest"

while true; do
    clear
    echo "=== I/O Cgroup Statistics ==="
    echo ""
    echo "Limits:"
    cat $CGROUP/io.max
    echo ""
    echo "Statistics:"
    cat $CGROUP/io.stat
    echo ""
    echo "Weight:"
    cat $CGROUP/io.weight 2>/dev/null || echo "N/A"
    sleep 2
done
EOF

chmod +x /tmp/monitor_io.sh

# Run monitor
/tmp/monitor_io.sh &
MONITOR_PID=$!

# Start I/O task
/tmp/io_intensive.sh 100 &
IO_PID=$!
echo $IO_PID > iotest/cgroup.procs

# Let it run
sleep 30
kill $MONITOR_PID 2>/dev/null || true
```

### Step 13: Clean up

```bash
# Kill test processes
pkill -f io_intensive.sh
pkill -f monitor_io.sh
pkill -f dd

# Wait for cleanup
sleep 2

# Remove test files
rm -f /tmp/io_test_file /tmp/write_test /tmp/test1 /tmp/test2 /tmp/throttle_test

# Remove cgroups (must be empty)
rmdir /sys/fs/cgroup/iotest/weight1
rmdir /sys/fs/cgroup/iotest/weight2
rmdir /sys/fs/cgroup/iotest

# Verify
ls /sys/fs/cgroup/iotest
# Should show: No such file or directory
```

## Verification

To verify I/O limits:

1. **Limit set**: `io.max` shows device and limits
2. **Statistics tracked**: `io.stat` shows rbytes, wbytes, etc.
3. **Limit enforced**: I/O speed should not exceed limit
4. **Throttling**: I/O throttled when limit exceeded
5. **Weight works**: Processes with higher weight get more I/O

## Common Pitfalls

1. **I/O controller not enabled**: Must enable in parent
   ```bash
   echo "+io" > cgroup.subtree_control
   ```

2. **Wrong device major:minor**: Must use correct device numbers
   ```bash
   lsblk -n -o MAJ:MIN,NAME
   ```

3. **Format error**: io.max format is strict
   ```bash
   echo "8:0 rbps=10485760 wbps=5242880" > io.max
   ```

4. **Not monitoring**: Use `io.stat` to verify limits work
   ```bash
   cat io.stat
   ```

5. **Cache effects**: Use `oflag=direct` and `iflag=direct` to bypass cache

6. **Virtio / cloud disks**: On VMs (Lima, cloud images), `io.max` may be set successfully while `dd` still looks very fast and `io.stat` stays sparse. Limits apply to the block device; filesystem/page-cache and virtio paths can hide throttling. Prefer watching `io.stat` / `nr_throttled`-style counters over wall-clock `dd` alone.
## Understanding I/O Limits

I/O limit format:
- **Device**: `major:minor` (e.g., `8:0` for `/dev/sda`)
- **Read limit**: `rbps=bytes_per_second`
- **Write limit**: `wbps=bytes_per_second`
- **Multiple devices**: One per line

I/O statistics:
- **rbytes/wbytes**: Total bytes read/written
- **rios/wios**: Number of read/write I/O operations
- **Per device**: Statistics shown per device

I/O throttling:
- When limit exceeded, I/O is throttled
- Process waits until bandwidth available
- No data loss, just slower I/O

## Next Steps

In Step 7.1, we'll learn about Linux capabilities, which allow fine-grained control over what root can do. We'll learn how to drop unnecessary privileges, making containers more secure.
