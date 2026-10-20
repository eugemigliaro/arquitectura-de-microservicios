# Step 7.1: Linux Capabilities

## Learning Objectives

- Understand Linux capabilities
- List current capabilities
- Drop capabilities using `capsh`
- Test what operations are blocked
- Understand capability-based security
- Use `setcap` to set capabilities on files

## Prerequisites

- Completion of Step 6.4 (I/O Limits)
- Understanding of Linux security model
- Root access on a Linux system
- `libcap2-bin` package installed (for capsh, setcap, getcap)

## Background

Linux capabilities divide root privileges into smaller units:
- **CAP_NET_BIND_SERVICE**: Bind to ports < 1024
- **CAP_SYS_ADMIN**: Administrative operations
- **CAP_NET_RAW**: Raw network access
- **CAP_SYS_CHROOT**: Use chroot
- And many more...

Why capabilities matter:
- Principle of least privilege
- Drop unnecessary capabilities
- Reduce attack surface
- More secure than full root

Docker uses capabilities to limit what containers can do, even when running as root inside the container.

## Commands

> **⚠️ Modern `capsh` (libcap2):** There is **no** `--add=` option anymore.
> Use capability text with `--caps=` (e.g. `--caps='cap_net_bind_service=ep'`).
> `--drop=LIST` still removes capabilities from the **bounding** set.

### Step 1: Install required tools

```bash
# Install libcap2-bin (provides capsh, setcap, getcap)
# On Debian/Ubuntu:
sudo apt-get update
sudo apt-get install -y libcap2-bin

# Verify installation
which capsh
which setcap
which getcap
```

### Step 2: Check current capabilities

```bash
# Check capabilities of current process
cat /proc/self/status | grep Cap
# Shows: CapInh, CapPrm, CapEff, CapBnd, CapAmb

# Decode capabilities
capsh --print
# Shows human-readable capabilities

# As root, check all capabilities
sudo capsh --print
# Should show many capabilities
```

### Step 3: Understand capability sets

```bash
# Capability sets:
# - CapInh: Inheritable (passed to children)
# - CapPrm: Permitted (can be used)
# - CapEff: Effective (currently effective)
# - CapBnd: Bounding (maximum allowed)
# - CapAmb: Ambient (preserved across exec)

# Check each set
cat /proc/self/status | grep -E "Cap(Inh|Prm|Eff|Bnd|Amb)"
```

### Step 4: Drop capabilities using capsh

```bash
# Drop all capabilities except a few
# Example: Keep only CAP_NET_BIND_SERVICE
capsh --drop=all --caps='cap_net_bind_service=ep' -- -c 'capsh --print'
# Shows only cap_net_bind_service

# Drop specific capabilities
capsh --drop=cap_sys_admin,cap_sys_chroot -- -c 'capsh --print'
# Shows capabilities minus the dropped ones

# Drop all and add only what's needed
capsh --drop=all --caps='cap_net_bind_service,cap_chown=ep' -- -c 'capsh --print'
```

### Step 5: Test operations with dropped capabilities

```bash
# Test 1: Try to bind to port < 1024 without capability
capsh --drop=cap_net_bind_service -- -c 'python3 -m http.server 80' 2>&1
# Should fail (unless already running as root in some cases)

# Test 2: Try to bind with capability
capsh --drop=all --caps='cap_net_bind_service=ep' -- -c 'python3 -m http.server 80 & sleep 2 && curl http://localhost:80 && kill %1' 2>&1
# Should work

# Test 3: Try chroot without capability
capsh --drop=cap_sys_chroot -- -c 'chroot /tmp /bin/bash' 2>&1
# Should fail

# Test 4: Try chroot with capability
capsh --drop=all --caps='cap_sys_chroot=ep' -- -c 'mkdir -p /tmp/chroot_test && chroot /tmp/chroot_test /bin/bash -c "echo chroot works"' 2>&1
# Should work (if chroot_test exists and has bash)
```

### Step 6: Use setcap on files

> **Nota:** `setcap` solo funciona en binarios ELF, NO en scripts interpretados (Python, Bash, etc.). Para scripts, el kernel ejecuta el intérprete (python3), no el script, así que las capabilities del archivo script se ignoran. Por eso usamos un programa C compilado.

```bash
# Create a test program (must be compiled C, not a script)
cat > /tmp/test_bind.c <<'EOF'
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char *argv[]) {
    int port = argc > 1 ? atoi(argv[1]) : 80;
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    addr.sin_addr.s_addr = INADDR_ANY;
    if (bind(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        printf("Permission denied: Cannot bind to port %d\n", port);
        return 1;
    }
    printf("Successfully bound to port %d\n", port);
    close(sock);
    return 0;
}
EOF

# Compile
gcc -o /tmp/test_bind /tmp/test_bind.c

# Try without capability (as non-root)
/tmp/test_bind 80
# Should fail: Permission denied

# Set capability on the compiled binary
sudo setcap cap_net_bind_service+ep /tmp/test_bind

# Verify capability
getcap /tmp/test_bind
# Should show: /tmp/test_bind cap_net_bind_service=ep

# Now try again (even as non-root)
/tmp/test_bind 80
# Should work!
```

### Step 7: Test common capabilities

```bash
# Test CAP_SYS_ADMIN (needed for mount)
capsh --drop=all --caps='cap_sys_admin=ep' -- -c 'mount -t tmpfs tmpfs /tmp/test_mount && umount /tmp/test_mount && echo "Mount works"' 2>&1
# Should work

# Test CAP_NET_RAW (needed for raw sockets, ping)
capsh --drop=all --caps='cap_net_raw=ep' -- -c 'ping -c 1 127.0.0.1' 2>&1
# Should work (if ping available)

# Test CAP_CHOWN (needed to change file ownership)
capsh --drop=all --caps='cap_chown=ep' -- -c 'touch /tmp/test_chown && chown nobody:nogroup /tmp/test_chown && ls -l /tmp/test_chown' 2>&1
# Should work
```

### Step 8: Understand capability inheritance

```bash
# Create a script that checks capabilities
cat > /tmp/check_caps.sh <<'EOF'
#!/bin/bash
echo "=== Capabilities ==="
capsh --print
echo ""
echo "=== From /proc ==="
cat /proc/self/status | grep Cap
EOF

chmod +x /tmp/check_caps.sh

# Run with dropped capabilities
capsh --drop=cap_sys_admin -- -c '/tmp/check_caps.sh'
# Child inherits dropped capabilities

# Run with added capabilities
capsh --drop=all --caps='cap_net_bind_service=ep' -- -c '/tmp/check_caps.sh'
# Child gets the added capability
```

### Step 9: Combine capabilities with namespaces

```bash
# Create container with limited capabilities
capsh --drop=all --caps='cap_sys_chroot,cap_sys_admin=ep' -- \
    --unshare --mount -- -c '
    mkdir -p /tmp/cap_container
    mount -t tmpfs tmpfs /tmp/cap_container
    mount --bind /tmp/cap_container /tmp/cap_container
    mkdir -p /tmp/cap_container/.oldroot
    cd /tmp/cap_container
    pivot_root . .oldroot
    umount -l /.oldroot
    rmdir /.oldroot
    capsh --print
    echo "In container with limited capabilities"
    exec /bin/bash
'
```

### Step 10: List all capabilities

```bash
# List all available capabilities
capsh --print | grep "Current:" | sed 's/Current: //' | tr ',' '\n' | sort

# Or use capsh to decode
capsh --decode=$(cat /proc/self/status | grep CapEff | awk '{print $2}')
# Shows effective capabilities in human-readable form

# Common capabilities:
# - cap_chown: Change file ownership
# - cap_dac_override: Bypass file permissions
# - cap_dac_read_search: Bypass read/search permissions
# - cap_fowner: Bypass file ownership checks
# - cap_fsetid: Set setuid/setgid
# - cap_kill: Send signals
# - cap_setgid: Set GID
# - cap_setuid: Set UID
# - cap_setpcap: Set capabilities
# - cap_net_bind_service: Bind to ports < 1024
# - cap_net_broadcast: Broadcast/multicast
# - cap_net_admin: Network administration
# - cap_net_raw: Raw network access
# - cap_ipc_lock: Lock memory
# - cap_ipc_owner: IPC ownership
# - cap_sys_module: Load/unload modules
# - cap_sys_rawio: Raw I/O
# - cap_sys_chroot: chroot
# - cap_sys_ptrace: ptrace
# - cap_sys_pacct: Process accounting
# - cap_sys_admin: System administration
# - cap_sys_boot: Reboot
# - cap_sys_nice: Nice priority
# - cap_sys_resource: Resource limits
# - cap_sys_time: System time
# - cap_sys_tty_config: TTY configuration
# - cap_mknod: Create special files
# - cap_lease: File leases
# - cap_audit_write: Write to audit log
# - cap_audit_control: Audit control
# - cap_setfcap: Set file capabilities
# - cap_mac_override: MAC override
# - cap_mac_admin: MAC administration
# - cap_syslog: syslog
# - cap_wake_alarm: Wake alarms
# - cap_block_suspend: Block suspend
# - cap_audit_read: Read audit log
```

### Step 11: Docker-like capability dropping

```bash
# Docker drops many capabilities by default
# Create a script that mimics Docker's default capabilities
cat > /tmp/docker_caps.sh <<'EOF'
#!/bin/bash
# Docker's default capabilities (what it keeps)
DOCKER_CAPS="cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap"

# Drop all, then add Docker defaults
capsh --drop=all --caps="${DOCKER_CAPS}=ep" -- -c 'capsh --print'
EOF

chmod +x /tmp/docker_caps.sh
/tmp/docker_caps.sh
```

### Step 12: Clean up

```bash
# Remove test files
rm -f /tmp/test_bind /tmp/test_bind.c /tmp/check_caps.sh /tmp/docker_caps.sh

# Remove capabilities from files
sudo setcap -r /tmp/test_bind 2>/dev/null || true

# Remove test directories
rm -rf /tmp/chroot_test /tmp/test_mount /tmp/cap_container
```

## Verification

To verify Linux capabilities:

1. **Capabilities listed**: `capsh --print` shows current capabilities
2. **Capabilities dropped**: Operations fail without required capability
3. **Capabilities added**: Operations work with capability
4. **File capabilities**: `setcap` allows non-root to use capabilities
5. **Inheritance**: Child processes inherit capabilities

## Common Pitfalls

1. **capsh not installed**: Install libcap2-bin package
   ```bash
   sudo apt-get install libcap2-bin
   ```

2. **Wrong capability name**: Use lowercase with `cap_` prefix
   ```bash
   cap_net_bind_service  # Correct
   CAP_NET_BIND_SERVICE  # Wrong
   ```

3. **Not dropping all first**: Drop all, then add what's needed
   ```bash
   capsh --drop=all --caps='cap_net_bind_service=ep'
   ```

4. **File capabilities**: Must use `+ep` (effective, permitted)
   ```bash
   setcap cap_net_bind_service+ep /path/to/file
   ```

5. **Testing as root**: Some operations work as root even without capability

## Understanding Capabilities

Capability model:
- **Divides root power**: Instead of all-or-nothing root
- **Granular control**: Enable only what's needed
- **Security**: Reduce attack surface
- **Docker uses this**: Containers run with limited capabilities

Common use cases:
- **Web server**: Needs `cap_net_bind_service` for port 80/443
- **Network tools**: Need `cap_net_raw` for ping, tcpdump
- **Container runtime**: Needs `cap_sys_admin`, `cap_sys_chroot` for namespaces

## Next Steps

In Step 7.2, we'll learn about seccomp filters, which restrict system calls. This provides another layer of security by preventing containers from using dangerous system calls.
