# Step 5.2: User Namespace with Filesystem

## Learning Objectives

- Combine user and mount namespaces
- Configure UID/GID mapping files properly
- Create files as "root" in container
- Verify ownership mapping on host
- Understand file ownership in containers
- Use newuidmap and newgidmap for proper mapping

## Prerequisites

- Completion of Step 5.1 (User Namespace Basics)
- Understanding of UID/GID mapping
- Root access for initial setup
- Understanding of mount namespaces
- `uidmap` package installed (`sudo apt-get install -y uidmap`)
- On Ubuntu 24.04: disable AppArmor user namespace restriction:
  ```bash
  sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
  ```

## Background

Combining user and mount namespaces:
- **User namespace**: Maps UIDs/GIDs
- **Mount namespace**: Isolates filesystem
- **Together**: Proper file ownership in containers

Key concepts:
- Files created as "root" (UID 0) in container
- On host, these files owned by mapped UID (e.g., 100000)
- `/proc/<pid>/uid_map` and `/proc/<pid>/gid_map` define mappings
- `newuidmap` and `newgidmap` tools help configure mappings

This is how Docker handles file ownership - files owned by root in container are owned by a high UID on host, providing security while maintaining functionality.

## Commands

> **⚠️ Environment:** Same AppArmor userns requirement as Step 5.1 — ensure
> `kernel.apparmor_restrict_unprivileged_userns=0` (see README).

### Step 1: Set up UID/GID mapping files

```bash
# As root, configure subuid and subgid
sudo su

# Get your username (for when running as non-root later)
REGULAR_USER=$(who | awk '{print $1}' | head -1)
echo "Regular user: $REGULAR_USER"

# Configure subuid (if not already done)
if ! grep -q "^${REGULAR_USER}:" /etc/subuid 2>/dev/null; then
    echo "${REGULAR_USER}:100000:65536" >> /etc/subuid
    echo "Added UID mapping for $REGULAR_USER"
fi

# Configure subgid
if ! grep -q "^${REGULAR_USER}:" /etc/subgid 2>/dev/null; then
    echo "${REGULAR_USER}:100000:65536" >> /etc/subgid
    echo "Added GID mapping for $REGULAR_USER"
fi

# Verify
cat /etc/subuid | grep "$REGULAR_USER"
cat /etc/subgid | grep "$REGULAR_USER"
```

### Step 2: Create container root filesystem

```bash
# Create container root
CONTAINER_ROOT="/tmp/container_user"
mkdir -p "$CONTAINER_ROOT"
cd "$CONTAINER_ROOT"

# Create directory structure
mkdir -p bin lib lib64 etc proc sys dev tmp root .oldroot usr/bin

# On Ubuntu 24.04, /bin is a symlink to /usr/bin. After pivot_root, bash
# resolves commands via PATH to /usr/bin/. We mirror that structure with
# a symlink so binaries are found regardless of PATH resolution.
rm -rf "$CONTAINER_ROOT/usr/bin"
ln -s ../../bin "$CONTAINER_ROOT/usr/bin"

# Copy essential binaries
cp /bin/bash "$CONTAINER_ROOT/bin/"
cp /bin/ls "$CONTAINER_ROOT/bin/"
cp /bin/touch "$CONTAINER_ROOT/bin/"
cp /bin/mkdir "$CONTAINER_ROOT/bin/"
cp /bin/mount "$CONTAINER_ROOT/bin/"
cp /bin/umount "$CONTAINER_ROOT/bin/"
cp /bin/rmdir "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/rmdir "$CONTAINER_ROOT/bin/"
cp /bin/readlink "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/readlink "$CONTAINER_ROOT/bin/"
cp /bin/head "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/head "$CONTAINER_ROOT/bin/"
cp /bin/cat "$CONTAINER_ROOT/bin/"
cp /bin/whoami "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/whoami "$CONTAINER_ROOT/bin/"
cp /bin/id "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/id "$CONTAINER_ROOT/bin/"
cp /bin/chown "$CONTAINER_ROOT/bin/" 2>/dev/null || cp /usr/bin/chown "$CONTAINER_ROOT/bin/"
cp /bin/echo "$CONTAINER_ROOT/bin/"
cp /bin/grep "$CONTAINER_ROOT/bin/"

# Copy libraries
for binary in /bin/bash /bin/ls /bin/touch /bin/mkdir /bin/mount /bin/umount /usr/bin/rmdir /usr/bin/readlink /usr/bin/head /bin/cat /usr/bin/whoami /usr/bin/id /usr/bin/chown /bin/echo /bin/grep; do
    ldd "$binary" 2>/dev/null | grep -o '/[^ ]*' | while read lib; do
        mkdir -p "$CONTAINER_ROOT$(dirname "$lib")"
        cp "$lib" "$CONTAINER_ROOT$lib" 2>/dev/null || true
    done
done

# Create basic config
cat > "$CONTAINER_ROOT/etc/passwd" <<EOF
root:x:0:0:root:/root:/bin/bash
EOF

cat > "$CONTAINER_ROOT/etc/group" <<EOF
root:x:0:
EOF

# Set ownership to the regular user who will run the container
# (user namespace root maps to this UID, so it needs write access)
REGULAR_USER=$(who | awk '{print $1}' | head -1)
chown -R "$REGULAR_USER:$REGULAR_USER" "$CONTAINER_ROOT" 2>/dev/null || true
```

### Step 3: Create script with user + mount namespaces

> **Nota:** Para ver el efecto real del UID mapping (UID 0 en container → UID alto en host), ejecutá este paso como tu **usuario regular** (no root). Si lo ejecutás como root, el mapping es trivial (0→0) y no se ve la diferencia.

```bash
# Exit root
exit

# Create container script
# Note: the script does NOT call unshare — you run it WITH unshare
# so that exec /bin/bash inherits your terminal for interactive use.
cat > /tmp/run_user_container.sh <<'SCRIPT'
#!/bin/bash
CONTAINER_ROOT="/tmp/container_user"

echo "=== Inside User Namespace ==="
echo "Current user: $(whoami)"
echo "Current UID: $(id -u)"
echo "User namespace: $(readlink /proc/self/ns/user)"

# Bind mount and pivot
# Note: in user namespace, mount proc/sysfs/dev AFTER pivot_root
# (mounting to host paths fails with permission denied in user ns)
mount --bind "$CONTAINER_ROOT" "$CONTAINER_ROOT"
mkdir -p "$CONTAINER_ROOT/.oldroot"
cd "$CONTAINER_ROOT"
pivot_root . .oldroot

# Update working directory to new root
cd /

# Mount proc (needed before umount can resolve mount points)
mount -t proc proc /proc

# Unmount old root
umount -l /.oldroot
rmdir /.oldroot

echo ""
echo "=== Container Info ==="
echo "PID: $$"
echo "User: $(whoami)"
echo "UID: $(id -u)"
echo ""

# Create a file as root
echo "Creating file as root..."
touch /tmp/root_created_file
mkdir -p /tmp/root_created_dir

# Show ownership
echo "File ownership in container:"
ls -l /tmp/root_created_file
ls -ld /tmp/root_created_dir

# Show UID mapping
echo ""
echo "UID mapping:"
cat /proc/self/uid_map

echo ""
echo "GID mapping:"
cat /proc/self/gid_map

# Start shell
exec /bin/bash
SCRIPT

chmod +x /tmp/run_user_container.sh
```

### Step 4: Run the container

```bash
# Execute container with user + mount + PID namespaces
# (PID namespace is needed for mounting /proc inside the user namespace)
unshare --user --mount --pid --fork --map-root-user /tmp/run_user_container.sh
```

### Step 5: Verify inside container

Inside the container:

```bash
# Check user
whoami
# Should show: root

# Check UID
id
# Should show: uid=0(root) gid=0(root)

# Check UID mapping
cat /proc/self/uid_map
# Should show: 0 <mapped-uid> 1
# Example: 0 100000 1

# Create files as root
touch /tmp/test_file1
mkdir /tmp/test_dir1
echo "test content" > /tmp/test_file2

# Check ownership
ls -l /tmp/test_file*
ls -ld /tmp/test_dir*
# Should show root:root ownership
```

### Step 6: Verify on host

In another terminal (host):

```bash
# Find container process
ps aux | grep "run_user_container" | grep -v grep
# Note the PID

# Check UID mapping
CONTAINER_PID=$(ps aux | grep "run_user_container" | grep -v grep | awk '{print $2}' | head -1)
if [ ! -z "$CONTAINER_PID" ]; then
    echo "Container PID: $CONTAINER_PID"
    echo "UID map:"
    cat /proc/$CONTAINER_PID/uid_map
    echo "GID map:"
    cat /proc/$CONTAINER_PID/gid_map
    
    # Check process UID on host
    echo "Process UID on host:"
    ps -o pid,uid,user -p $CONTAINER_PID
fi

# Check file ownership on host
ls -l /tmp/container_user/tmp/test_file* 2>/dev/null || \
ls -l /tmp/test_file* 2>/dev/null || \
echo "Files may be in container filesystem"

# If files are in container root, check there
ls -l /tmp/container_user/tmp/ 2>/dev/null | head -10
# Should show files owned by mapped UID (e.g., 100000)
```

### Step 7: Test file creation and ownership

```bash
# Inside container, create more files
# Exit container first if needed, then re-enter
exit

# Re-run container
/tmp/run_user_container.sh

# Inside container
# Create file with specific ownership
touch /tmp/owned_by_root
chown root:root /tmp/owned_by_root

# Create file as if by different user (if we had one)
# Note: In this simple setup, we only have root

# Check all files
ls -la /tmp/ | head -10
```

### Step 8: Use newuidmap and newgidmap (advanced)

```bash
# Exit container
exit

# These tools allow more complex mappings
# Check if they're available
which newuidmap
which newgidmap

# Example: Map multiple UID ranges
# This is more advanced and typically not needed for basic containers
```

### Step 9: Verify mapping persistence

```bash
# Create a file in container
/tmp/run_user_container.sh

# Inside container
echo "test" > /root/container_file.txt
ls -l /root/container_file.txt
# Should show root:root

# Exit
exit

# On host, check ownership
ls -l /tmp/container_user/root/container_file.txt
# Should show mapped UID (e.g., 100000:100000)
```

### Step 10: Understand the mapping

```bash
# Inside container, check the mapping details
/tmp/run_user_container.sh

# Inside container
cat /proc/self/uid_map
cat /proc/self/gid_map
cat /proc/self/status | grep -i "uid\|gid" | head -10

# This shows how UIDs are mapped
```

## Verification

To verify user namespace with filesystem:

1. **User namespace**: `readlink /proc/self/ns/user` should be different from host
2. **Appear as root**: `whoami` should show "root" inside container
3. **UID mapping**: `/proc/self/uid_map` should show mapping (e.g., `0 100000 1`)
4. **File ownership in container**: Files show as root:root
5. **File ownership on host**: Same files show as mapped UID (e.g., 100000:100000)
6. **Process UID on host**: Container process has non-root UID on host

## Common Pitfalls

1. **Missing subuid/subgid**: Must configure before creating namespace
   ```bash
   echo "user:100000:65536" >> /etc/subuid
   ```

2. **Not using --map-root-user**: Need this flag to appear as root
   ```bash
   unshare --user --map-root-user bash
   ```

3. **Mount namespace required**: For proper filesystem isolation
   ```bash
   unshare --user --mount --map-root-user bash
   ```

4. **File location**: Files in container root may not be directly accessible on host
   ```bash
   # Check in container root directory
   ls -l /tmp/container_user/tmp/
   ```

5. **Permission issues**: Some operations may still fail due to limited capabilities

## Understanding File Ownership Mapping

How it works:
1. **In container**: Create file as UID 0 (root)
2. **UID mapping**: Maps UID 0 → 100000 (or configured)
3. **On host**: File owned by UID 100000
4. **Security**: Container "root" can't affect host files owned by real root

Example:
```bash
# In container
touch /tmp/file
# ls shows: root:root

# On host  
ls -l /tmp/container_user/tmp/file
# Shows: 100000:100000
```

## Security Benefits

This combination provides:
- **Rootless containers**: Run containers without host root
- **File isolation**: Container files mapped to non-root UIDs
- **Attack mitigation**: Compromised container can't access host root files
- **Capability limits**: Container root has fewer capabilities than host root

## Next Steps

In Step 6.1, we'll learn about cgroups (control groups), which limit and account for resource usage. This allows us to limit CPU, memory, and I/O for containers - essential for multi-tenant environments.
