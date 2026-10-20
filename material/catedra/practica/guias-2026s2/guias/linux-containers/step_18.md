# Step 8.1: Overlay Filesystems

## Learning Objectives

- Understand OverlayFS and how it works
- Create overlay mounts
- Understand lower/upper/work directories
- Understand copy-on-write behavior
- Build layered filesystem
- Compare with Docker layers

## Prerequisites

- Completion of Step 7.2 (Seccomp Filters)
- Understanding of Linux mount points
- Root access on a Linux system
- Kernel with OverlayFS support (most modern kernels)

## Background

OverlayFS creates a layered filesystem:
- **Lower layer(s)**: Read-only base layers
- **Upper layer**: Read-write layer for changes
- **Work directory**: Temporary files for atomic operations
- **Merged view**: Combined view of all layers

How it works:
- Reads come from upper layer if exists, else lower layer
- Writes go to upper layer (copy-on-write)
- Deletes create "whiteout" files in upper layer

This is how Docker implements image layers and container filesystems - efficient storage and fast container creation.

## Commands

### Step 1: Check OverlayFS support

```bash
# Check if OverlayFS is available
grep overlay /proc/filesystems
# Should show: nodev overlay

# Or check kernel config
grep CONFIG_OVERLAY_FS /boot/config-$(uname -r) 2>/dev/null || \
echo "Check kernel for OVERLAY_FS support"

# Modern kernels have OverlayFS enabled by default
```

### Step 2: Create directory structure

```bash
# Ensure root access
sudo su

# Create directories for overlay
mkdir -p /tmp/overlay/{lower,upper,work,merged}

# Lower layer (base, read-only)
cd /tmp/overlay/lower
echo "Base file" > base.txt
mkdir dir1
echo "In dir1" > dir1/file1.txt

# Upper layer (changes, read-write) - starts empty
# Work directory (temporary) - starts empty
# Merged (combined view) - will be created by mount
```

### Step 3: Create overlay mount

> **Nota para Docker**: Si están corriendo este lab dentro de un container Docker,
> OverlayFS no puede montarse sobre el filesystem del container (que ya usa overlay2).
> Solución: usar `/dev/shm` (tmpfs) en vez de `/tmp` para los directorios del overlay:
> ```bash
> mkdir -p /dev/shm/overlay/{lower,upper,work,merged}
> # Repetir los pasos usando /dev/shm/overlay en vez de /tmp/overlay
> ```

```bash
# Mount overlay filesystem
mount -t overlay overlay \
  -o lowerdir=/tmp/overlay/lower,upperdir=/tmp/overlay/upper,workdir=/tmp/overlay/work \
  /tmp/overlay/merged

# Verify mount
mount | grep overlay
# Should show the overlay mount

# Check merged directory
ls -la /tmp/overlay/merged
# Should show files from lower layer
```

### Step 4: Test read from lower layer

```bash
# Read file from lower layer (via merged)
cat /tmp/overlay/merged/base.txt
# Should show: Base file

# Read file from subdirectory
cat /tmp/overlay/merged/dir1/file1.txt
# Should show: In dir1

# Verify it's from lower layer
ls -la /tmp/overlay/lower/
ls -la /tmp/overlay/upper/
# Upper should be empty (no changes yet)
```

### Step 5: Test copy-on-write (write to merged)

```bash
# Write to merged view
echo "Modified content" > /tmp/overlay/merged/base.txt

# Check upper layer (should have the file now)
ls -la /tmp/overlay/upper/
# Should show base.txt

# Check content
cat /tmp/overlay/upper/base.txt
# Should show: Modified content

# Lower layer unchanged
cat /tmp/overlay/lower/base.txt
# Should show: Base file (original)

# Merged view shows modified version
cat /tmp/overlay/merged/base.txt
# Should show: Modified content
```

### Step 6: Test creating new files

```bash
# Create new file in merged view
echo "New file" > /tmp/overlay/merged/newfile.txt

# Check upper layer
ls -la /tmp/overlay/upper/
# Should show newfile.txt

# Lower layer doesn't have it
ls /tmp/overlay/lower/newfile.txt
# Should show: No such file

# Merged view shows it
cat /tmp/overlay/merged/newfile.txt
# Should show: New file
```

### Step 7: Test deleting files

```bash
# Delete a file from lower layer
rm /tmp/overlay/merged/dir1/file1.txt

# Check upper layer
ls -la /tmp/overlay/upper/dir1/ 2>/dev/null || echo "dir1 may not exist yet"
# May show a character device (whiteout) or the directory may be marked

# Check if file still exists in lower
ls /tmp/overlay/lower/dir1/file1.txt
# Should still exist (lower is read-only)

# Check merged view
ls /tmp/overlay/merged/dir1/file1.txt
# Should show: No such file (deleted in merged view)

# Whiteout files (for deletions) are special character devices
# Check upper layer for whiteouts
find /tmp/overlay/upper -type c
```

### Step 8: Multiple lower layers (like Docker)

```bash
# Unmount current overlay
umount /tmp/overlay/merged

# Create multiple lower layers (like Docker image layers)
mkdir -p /tmp/overlay/{layer1,layer2,layer3,upper,work,merged}

# Layer 1 (base)
echo "Layer 1" > /tmp/overlay/layer1/layer1.txt
mkdir /tmp/overlay/layer1/common
echo "Common in layer1" > /tmp/overlay/layer1/common/file.txt

# Layer 2 (on top of layer1)
echo "Layer 2" > /tmp/overlay/layer2/layer2.txt
echo "Common in layer2" > /tmp/overlay/layer2/common/file.txt  # Overwrites layer1

# Layer 3 (on top of layer2)
echo "Layer 3" > /tmp/overlay/layer3/layer3.txt

# Mount with multiple lower layers (order matters: leftmost is top)
mount -t overlay overlay \
  -o lowerdir=/tmp/overlay/layer3:/tmp/overlay/layer2:/tmp/overlay/layer1,upperdir=/tmp/overlay/upper,workdir=/tmp/overlay/work \
  /tmp/overlay/merged

# Check merged view
ls -la /tmp/overlay/merged/
# Should show files from all layers

# Check which layer a file comes from
cat /tmp/overlay/merged/common/file.txt
# Should show: Common in layer2 (topmost layer with this file)
```

### Step 9: Understand Docker-like layering

```bash
# This mimics Docker's layer structure:
# - layer1, layer2, layer3 = Docker image layers (read-only)
# - upper = Container's writable layer
# - merged = Container's filesystem view

# Create a change in container (upper layer)
echo "Container change" > /tmp/overlay/merged/container.txt

# Modify a file from image
echo "Modified in container" > /tmp/overlay/merged/layer2.txt

# Check what's in upper (container changes)
ls -la /tmp/overlay/upper/
# Should show container.txt and layer2.txt

# Image layers unchanged
cat /tmp/overlay/layer2/layer2.txt
# Should still show: Layer 2 (original)
```

### Step 10: Test directory operations

```bash
# Create directory in merged
mkdir /tmp/overlay/merged/newdir
echo "In newdir" > /tmp/overlay/merged/newdir/file.txt

# Check upper layer
ls -la /tmp/overlay/upper/newdir/
# Should show the new directory and file

# Rename a file
mv /tmp/overlay/merged/layer1.txt /tmp/overlay/merged/layer1_renamed.txt

# Check upper layer
ls /tmp/overlay/upper/
# Should show layer1_renamed.txt (and possibly a whiteout for layer1.txt)
```

### Step 11: Inspect overlay structure

```bash
# Show overlay mount info
findmnt /tmp/overlay/merged
# Shows overlay mount details

# Or use mount
mount | grep overlay
# Shows mount options

# Check disk usage
du -sh /tmp/overlay/*
# Upper layer only stores changes (efficient!)

# Compare with full copy
du -sh /tmp/overlay/merged
# Merged shows combined size, but actual storage is only changes
```

### Step 12: Clean up

```bash
# Unmount overlay
umount /tmp/overlay/merged

# Remove test directories
rm -rf /tmp/overlay

# Verify cleanup
ls /tmp/overlay
# Should show: No such file or directory
```

## Verification

To verify OverlayFS:

1. **Overlay mounted**: `mount | grep overlay` shows overlay mount
2. **Read from lower**: Files from lower layer visible in merged
3. **Write to upper**: Changes appear in upper layer
4. **Copy-on-write**: Lower layer unchanged after modifications
5. **Multiple layers**: Multiple lower layers work correctly
6. **Efficiency**: Only changes stored in upper layer

## Common Pitfalls

1. **OverlayFS not available**: Check kernel support
   ```bash
   grep overlay /proc/filesystems
   ```

2. **Wrong mount order**: Lower layers: leftmost is topmost
   ```bash
   lowerdir=layer3:layer2:layer1  # layer3 is top
   ```

3. **Work directory**: Must be on same filesystem as upper
   ```bash
   workdir=/tmp/overlay/work  # Same FS as upper
   ```

4. **Permission issues**: Need appropriate permissions for directories
   ```bash
   chmod 755 /tmp/overlay/*
   ```

5. **Unmount before cleanup**: Must unmount before removing directories

## Understanding OverlayFS

OverlayFS provides:
- **Layered filesystem**: Multiple read-only layers + one writable layer
- **Copy-on-write**: Changes stored separately
- **Efficiency**: Only changes consume space
- **Speed**: Fast container creation (no copying)

Docker uses this for:
- **Image layers**: Each Dockerfile instruction creates a layer
- **Container filesystem**: Container's writable layer on top
- **Storage efficiency**: Multiple containers share base layers

## Next Steps

In Step 8.2, we'll learn about the Container Runtime Interface (OCI) and `runc`, which is the low-level container runtime that Docker uses. We'll examine OCI bundle structure and create a basic container using `runc`.
