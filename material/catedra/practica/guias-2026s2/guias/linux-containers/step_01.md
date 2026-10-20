# Step 1.1: Basic Filesystem Isolation with `chroot`

## Learning Objectives

- Understand basic root filesystem isolation
- Learn how `chroot` works and its purpose
- Create a minimal root filesystem directory structure
- Copy essential binaries and libraries
- Verify filesystem isolation
- Understand the limitations of `chroot`

## Prerequisites

- Root access on a Linux system
- Basic Linux command-line knowledge
- Understanding of filesystem hierarchy

## Background

### What is chroot?

`chroot` (change root) is one of the oldest Unix system calls for filesystem isolation. It changes the apparent root directory for the current process and its children. This means processes running inside a `chroot` jail can only access files within the specified directory tree, providing basic filesystem isolation.

### How chroot Works (Without Namespaces)

Unlike modern containerization which uses Linux namespaces, `chroot` works at a simpler level:

1. **Filesystem Path Resolution**: When a process calls `chroot("/new/root")`, the kernel changes how that process resolves absolute paths. All paths starting with `/` are now resolved relative to `/new/root` instead of the real root `/`.

2. **No Kernel-Level Isolation**: `chroot` does NOT create a new namespace. It's a simple filesystem operation that:
   - Changes the root directory for the process
   - Affects only path resolution
   - Does NOT isolate processes, network, mounts, or other resources

3. **Shared Resources**: Because `chroot` doesn't use namespaces:
   - Processes still share the host's PID namespace (can see all host processes via `/proc`)
   - Network namespace is shared (same network interfaces)
   - Mount namespace is shared (can see host mounts)
   - IPC namespace is shared (shared memory, semaphores visible)
   - UTS namespace is shared (same hostname)

### Why chroot Doesn't Use Namespaces

`chroot` predates Linux namespaces (which were introduced in Linux 2.6.24+). It's a legacy mechanism that provides minimal isolation. Modern containers use multiple namespaces together:
- **Mount namespace**: Isolates mount points (covered in Step 1.2)
- **PID namespace**: Isolates process IDs (covered in Step 2.1)
- **Network namespace**: Isolates network stack (covered in Step 3.1)
- **UTS namespace**: Isolates hostname (covered in Step 4.1)
- **IPC namespace**: Isolates IPC resources (covered in Step 4.2)
- **User namespace**: Isolates user IDs (covered in Step 5.1)

### Limitations of chroot

`chroot` has significant limitations:
- Processes still share the host's PID namespace (can see all processes)
- Network namespace is shared (same network interfaces)
- Other namespaces are not isolated
- Requires root privileges
- Can be escaped if not properly configured (e.g., using `..` paths or accessing `/proc`)

Despite these limitations, `chroot` is a fundamental concept that Docker builds upon with more advanced isolation mechanisms. Modern containers combine `chroot`-like filesystem isolation with multiple namespaces for complete isolation.

## Key Commands Explained

Before diving into the practical steps, let's understand the key commands we'll be using:

### `chroot` - Change Root Directory

The `chroot` command changes the root directory for a process and its children. It's a system call wrapper that provides basic filesystem isolation.

**Syntax**: `chroot NEWROOT [COMMAND]`

**How it works**:
- Changes the root directory (`/`) to `NEWROOT` for the current process
- All absolute paths are resolved relative to the new root
- The process can only access files within the new root directory tree
- Child processes inherit the chroot jail

**Example**:
```bash
# Change root to /tmp/container_root and run bash
chroot /tmp/container_root /bin/bash

# Inside the chroot, / now refers to /tmp/container_root
pwd  # Shows: /
ls /  # Shows contents of /tmp/container_root
```

**Important notes**:
- Requires root privileges
- Does NOT create a namespace (unlike `unshare`)
- Process can still see host processes via `/proc` (if mounted)
- Can be escaped with careful path manipulation

### `ldd` - List Dynamic Dependencies

The `ldd` command lists shared library dependencies of a binary. It's essential for copying all required libraries into a container root filesystem.

**Syntax**: `ldd [OPTIONS] FILE`

**How it works**:
- Shows which shared libraries a binary needs to run
- Lists library paths that must exist for the binary to execute
- Helps identify all dependencies needed in a container

**Example**:
```bash
# List dependencies of bash
ldd /bin/bash
# Output shows libraries like:
#   libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
#   libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2

# Extract just the library paths
ldd /bin/bash | grep -o '/[^ ]*'
# Output: /lib/x86_64-linux-gnu/libc.so.6
#         /lib/x86_64-linux-gnu/libdl.so.2
```

**Why it's important**:
- Binaries won't run without their libraries
- Must copy all dependencies to container root
- Missing libraries cause "library not found" errors

**Common usage pattern**:
```bash
# Copy all dependencies of a binary to container root
ldd /bin/bash | grep -o '/[^ ]*' | xargs -I {} cp {} /tmp/container_root{}
```

## Commands

### Step 1: Create a minimal root filesystem directory

```bash
# Create a directory for our container root filesystem
mkdir -p /tmp/container_root

# Create essential directory structure
cd /tmp/container_root
mkdir -p bin sbin etc proc sys dev usr/bin usr/lib lib lib64

# On Ubuntu 24.04, /bin is a symlink to /usr/bin. After chroot, bash
# resolves commands via PATH to /usr/bin/. We mirror that structure with
# a symlink so binaries are found regardless of PATH resolution.
rm -rf /tmp/container_root/usr/bin
ln -s ../../bin /tmp/container_root/usr/bin
```

### Step 2: Copy essential binaries

```bash
# Copy basic shell and utilities
cp /bin/bash /tmp/container_root/bin/
cp /bin/ls /tmp/container_root/bin/
cp /bin/cat /tmp/container_root/bin/
cp /bin/pwd /tmp/container_root/bin/
cp /bin/echo /tmp/container_root/bin/

# Copy binaries needed for verification (ps, ip, whoami)
cp /usr/bin/whoami /tmp/container_root/bin/
cp /usr/bin/ps /tmp/container_root/bin/ 2>/dev/null || true
cp /sbin/ip /tmp/container_root/bin/ 2>/dev/null || cp /usr/sbin/ip /tmp/container_root/bin/ 2>/dev/null || true

# Copy essential system binaries
cp /bin/sh /tmp/container_root/bin/ 2>/dev/null || ln -s /bin/bash /tmp/container_root/bin/sh
```

### Step 3: Copy required libraries

```bash
# Copy all required libraries for each binary
# Uses 'while read' so dirname evaluates correctly per library
for binary in /bin/bash /bin/ls /bin/cat /bin/pwd /bin/echo /usr/bin/whoami /bin/sh /usr/bin/ps /sbin/ip; do
    ldd "$binary" 2>/dev/null | grep -o '/[^ ]*' | while read lib; do
        mkdir -p "/tmp/container_root$(dirname "$lib")"
        cp "$lib" "/tmp/container_root$lib" 2>/dev/null || true
    done
done

# Ensure the dynamic linker is available (x86_64 and aarch64)
if [ -e /lib64/ld-linux-x86-64.so.2 ]; then
    mkdir -p /tmp/container_root/lib64
    cp /lib64/ld-linux-x86-64.so.2 /tmp/container_root/lib64/ 2>/dev/null || true
fi
if [ -e /lib/ld-linux-aarch64.so.1 ]; then
    mkdir -p /tmp/container_root/lib
    cp -a /lib/ld-linux-aarch64.so.1 /tmp/container_root/lib/ 2>/dev/null || true
fi
```

### Step 4: Create basic configuration files

```bash
# Create a minimal /etc/passwd
cat > /tmp/container_root/etc/passwd <<EOF
root:x:0:0:root:/root:/bin/bash
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
EOF

# Create a minimal /etc/group
cat > /tmp/container_root/etc/group <<EOF
root:x:0:
nogroup:x:65534:
EOF

# Create a minimal /etc/hosts
cat > /tmp/container_root/etc/hosts <<EOF
127.0.0.1 localhost
::1 localhost ip6-localhost ip6-loopback
EOF
```

### Step 5: Enter the chroot environment

```bash
# Switch to root if not already
sudo su

# Enter the chroot jail
chroot /tmp/container_root /bin/bash
```

### Step 6: Verify isolation

Inside the chroot environment, run:

```bash
# Verify you're in the chroot
pwd
# Should show: /

# List files in root
ls -la /
# Should only show directories we created

# Try to access host filesystem
ls /home
# Should fail or show nothing (depending on setup)

# Check process list
# Note: without /proc mounted, `ps` fails with "mount -t proc proc /proc".
# Mount it from *outside* the chroot (same mount namespace — chroot does not isolate mounts):
#   mount -t proc proc /tmp/container_root/proc
# Then inside the chroot:
ps aux
# Should still show host processes (limitation of chroot)

# Check network interfaces
# Note: 'ip' may not be available on minimal installations (requires iproute2 package)
ip addr
# Should show host interfaces (limitation of chroot)
```

### Step 7: Exit and clean up

```bash
# Exit the chroot
exit

# Clean up (optional)
# rm -rf /tmp/container_root
```

## Verification

To verify that filesystem isolation is working:

1. **Inside chroot**: Run `pwd` - should show `/`
2. **Inside chroot**: Run `ls /` - should only show directories you created
3. **Inside chroot**: Try `ls /home` - should fail or be empty
4. **Inside chroot**: After mounting `/proc` on the host path (see Step 6), run `ps aux` - notice you can still see host processes (limitation)
5. **Inside chroot**: Run `ip addr` - notice you can still see host network (limitation)

## Common Pitfalls

1. **Missing libraries**: If binaries fail to run, check with `ldd` and copy missing libraries
   ```bash
   ldd /tmp/container_root/bin/bash
   ```

2. **Permission issues**: Ensure you have root access when using `chroot`

3. **Broken symlinks**: Some binaries may have broken symlinks. Check and fix:
   ```bash
   find /tmp/container_root -type l -exec ls -l {} \;
   ```

4. **Missing directories**: Some programs expect certain directories. Create them:
   ```bash
   mkdir -p /tmp/container_root/{tmp,var,run,root}
   ```

5. **Cannot access /proc or /sys**: These are virtual filesystems that need to be mounted separately (covered in next steps)

## Understanding the Limitations

After completing this step, you should understand that `chroot` provides:
- ✅ Basic filesystem isolation
- ❌ No process isolation (can see all host processes)
- ❌ No network isolation (shares host network)
- ❌ No mount point isolation
- ❌ No user/group isolation

## Next Steps

In the next step (Step 1.2), we'll learn about mount namespaces, which provide better isolation for mount points and are essential for proper containerization. Mount namespaces allow each container to have its own view of mounted filesystems, including `/proc` and `/sys`.
