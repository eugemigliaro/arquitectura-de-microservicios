# Step 7.2: Seccomp Filters

## Learning Objectives

- Understand seccomp modes
- Understand system call filtering
- Create a basic seccomp filter
- Block specific system calls
- Test blocked system calls
- Understand seccomp-bpf

## Prerequisites

- Completion of Step 7.1 (Linux Capabilities)
- Understanding of system calls
- Root access on a Linux system
- `seccomp` library and tools (usually built into kernel)

## Background

Seccomp (Secure Computing Mode) restricts system calls:
- **Mode 1 (strict)**: Only allow read, write, exit, sigreturn
- **Mode 2 (filter)**: Custom BPF filter to allow/deny specific syscalls

Why seccomp matters:
- Prevents use of dangerous system calls
- Reduces attack surface
- Defense in depth
- Used by Docker to restrict container syscalls

Docker uses seccomp filters to prevent containers from using system calls they don't need, making containers more secure.

## Commands

### Step 1: Check seccomp support

```bash
# Check if seccomp is available
grep CONFIG_SECCOMP /boot/config-$(uname -r) 2>/dev/null || \
grep SECCOMP /proc/config.gz 2>/dev/null || \
echo "Check kernel config for SECCOMP support"

# Check seccomp in kernel
cat /proc/sys/kernel/seccomp/actions_avail
# Should show available actions (e.g., kill_process, kill_thread, trap, etc.)

# Modern kernels have seccomp enabled by default
```

### Step 2: Understand seccomp modes

```bash
# Seccomp has two modes:
# 1. SECCOMP_MODE_STRICT: Only allow read, write, exit, sigreturn
# 2. SECCOMP_MODE_FILTER: Custom BPF filter

# We'll focus on mode 2 (filter mode) as it's more flexible
```

### Step 3: Create a simple seccomp test program

```bash
# Create a C program to test seccomp
cat > /tmp/seccomp_test.c <<'EOF'
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <sys/prctl.h>
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <linux/audit.h>
#include <linux/signal.h>
#include <sys/socket.h>
#include <errno.h>
#include <string.h>

int main() {
    printf("Testing system calls...\n");
    
    // Test 1: Allowed syscall (write)
    printf("1. Testing write()...\n");
    write(1, "Write works\n", 12);
    
    // Test 2: Try to create socket (may be blocked)
    printf("2. Testing socket()...\n");
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        printf("   socket() failed: %s\n", strerror(errno));
    } else {
        printf("   socket() succeeded\n");
        close(sock);
    }
    
    // Test 3: Try fork (may be blocked)
    printf("3. Testing fork()...\n");
    pid_t pid = fork();
    if (pid < 0) {
        printf("   fork() failed: %s\n", strerror(errno));
    } else if (pid == 0) {
        printf("   fork() succeeded (child)\n");
        _exit(0);
    } else {
        printf("   fork() succeeded (parent)\n");
        wait(NULL);
    }
    
    return 0;
}
EOF

# Compile
gcc -o /tmp/seccomp_test /tmp/seccomp_test.c
```

### Step 4: Use prctl for seccomp strict mode

```bash
# Create a program using seccomp strict mode
cat > /tmp/seccomp_strict.c <<'EOF'
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>
#include <sys/prctl.h>
#include <linux/seccomp.h>

int main() {
    printf("Before seccomp: Can do anything\n");
    
    // Enable seccomp strict mode
    printf("Enabling seccomp strict mode...\n");
    if (prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT) < 0) {
        perror("prctl");
        return 1;
    }
    
    printf("After seccomp: Only read, write, exit, sigreturn allowed\n");
    
    // This will work (write is allowed)
    write(1, "Write still works\n", 19);
    
    // This will be killed (fork not allowed)
    printf("Trying fork (will be killed)...\n");
    fork();
    
    return 0;
}
EOF

# Compile
gcc -o /tmp/seccomp_strict /tmp/seccomp_strict.c

# Run (will be killed when trying fork)
/tmp/seccomp_strict
echo "Exit code: $?"
# Process should be killed by seccomp
```

### Step 5: Use seccomp with BPF filter (advanced)

```bash
# Create a program with custom seccomp filter
cat > /tmp/seccomp_filter.c <<'EOF'
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <sys/prctl.h>
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <linux/audit.h>
#include <sys/socket.h>
#include <errno.h>
#include <string.h>

// Simple filter: allow read, write, exit, but block socket
struct sock_filter filter[] = {
    // Load syscall number
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS, offsetof(struct seccomp_data, nr)),
    
    // Allow read (syscall 0 on x86_64)
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    
    // Allow write (syscall 1)
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    
    // Allow exit (syscall 60)
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    
    // Allow exit_group (syscall 231)
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit_group, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    
    // Block everything else
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL_PROCESS),
};

struct sock_fprog prog = {
    .len = sizeof(filter) / sizeof(filter[0]),
    .filter = filter,
};

int main() {
    printf("Installing seccomp filter...\n");
    
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) < 0) {
        perror("prctl PR_SET_NO_NEW_PRIVS");
        return 1;
    }
    
    if (prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog) < 0) {
        perror("prctl PR_SET_SECCOMP");
        return 1;
    }
    
    printf("Filter installed. Testing...\n");
    
    // This should work (write allowed)
    write(1, "Write works\n", 12);
    
    // This should be killed (socket not allowed)
    printf("Trying socket() (will be killed)...\n");
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    
    printf("This should not print\n");
    return 0;
}
EOF

# Compile
gcc -o /tmp/seccomp_filter /tmp/seccomp_filter.c

# Run (will be killed when trying socket)
/tmp/seccomp_filter
echo "Exit code: $?"
```

### Step 6: Use seccomp-tools (if available)

```bash
# Install seccomp-tools (optional, for analysis)
# On Debian/Ubuntu:
# sudo apt-get install seccomp-tools

# Check if available
which seccomp-tools

# If available, can analyze seccomp profiles
# seccomp-tools dump <pid>
```

### Step 7: Test with Docker's seccomp profile

```bash
# Docker uses a default seccomp profile
# Check Docker's default profile (if Docker is installed)
if command -v docker &> /dev/null; then
    # Docker's default seccomp profile location
    cat /etc/docker/seccomp.json 2>/dev/null || \
    echo "Docker seccomp profile not found in default location"
    
    # Or check what Docker uses
    docker info 2>/dev/null | grep -i seccomp || true
fi
```

### Step 8: Create a Python seccomp test (using python-seccomp)

```bash
# Install python3-seccomp if available
# On Debian/Ubuntu:
# sudo apt-get install python3-seccomp

# Create Python seccomp test
cat > /tmp/seccomp_python.py <<'EOF'
#!/usr/bin/env python3
import seccomp
import sys

# Create filter
f = seccomp.SyscallFilter(seccomp.KILL)

# Allow specific syscalls
f.add_rule(seccomp.ALLOW, "read")
f.add_rule(seccomp.ALLOW, "write")
f.add_rule(seccomp.ALLOW, "exit")
f.add_rule(seccomp.ALLOW, "exit_group")

# Load filter
f.load()

# Test: write should work
sys.stdout.write("Write works\n")

# Test: socket should be killed
try:
    import socket
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    print("Socket created (should not happen)")
except:
    print("Socket blocked by seccomp")
    sys.exit(1)
EOF

# Try to run (if python-seccomp installed)
python3 /tmp/seccomp_python.py 2>&1 || \
echo "python-seccomp not installed, skipping Python test"
```

### Step 9: Understand seccomp actions

```bash
# Seccomp actions when syscall is blocked:
# - SECCOMP_RET_KILL_PROCESS: Kill entire process
# - SECCOMP_RET_KILL_THREAD: Kill calling thread
# - SECCOMP_RET_TRAP: Send SIGSYS signal
# - SECCOMP_RET_ERRNO: Return errno
# - SECCOMP_RET_TRACE: Notify ptrace
# - SECCOMP_RET_LOG: Log but allow
# - SECCOMP_RET_ALLOW: Allow syscall

# Check available actions
cat /proc/sys/kernel/seccomp/actions_avail
```

### Step 10: Test seccomp with namespaces

```bash
# Combine seccomp with container namespaces
# Create a simple container with seccomp

# First, create the seccomp filter program
cat > /tmp/container_seccomp.c <<'EOF'
#define _GNU_SOURCE
#include <stdio.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <sys/prctl.h>
#include <linux/seccomp.h>
#include <linux/filter.h>

// Allow common syscalls needed for basic operation
struct sock_filter filter[] = {
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS, offsetof(struct seccomp_data, nr)),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_open, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_close, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit_group, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL_PROCESS),
};

struct sock_fprog prog = {
    .len = sizeof(filter) / sizeof(filter[0]),
    .filter = filter,
};

int main(int argc, char *argv[]) {
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) < 0) {
        perror("prctl");
        return 1;
    }
    
    if (prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog) < 0) {
        perror("prctl seccomp");
        return 1;
    }
    
    printf("Seccomp filter active. Executing: %s\n", argv[1]);
    execvp(argv[1], &argv[1]);
    perror("execvp");
    return 1;
}
EOF

# Compile
gcc -o /tmp/container_seccomp /tmp/container_seccomp.c

# Test: Run a command with seccomp
/tmp/container_seccomp /bin/ls
# Should work (ls uses allowed syscalls)

# Test: Try something that needs blocked syscall
/tmp/container_seccomp /bin/ping -c 1 127.0.0.1 2>&1
# Should be killed (ping needs socket syscall)
```

### Step 11: Clean up

```bash
# Remove test programs
rm -f /tmp/seccomp_test.c /tmp/seccomp_test
rm -f /tmp/seccomp_strict.c /tmp/seccomp_strict
rm -f /tmp/seccomp_filter.c /tmp/seccomp_filter
rm -f /tmp/seccomp_python.py
rm -f /tmp/container_seccomp.c /tmp/container_seccomp
```

## Verification

To verify seccomp filters:

1. **Seccomp enabled**: Process runs with seccomp filter
2. **Allowed syscalls work**: Basic operations (read, write) work
3. **Blocked syscalls killed**: Processes killed when using blocked syscalls
4. **Filter active**: Can verify with `/proc/<pid>/status` or seccomp-tools

## Common Pitfalls

1. **PR_SET_NO_NEW_PRIVS required**: Must set before seccomp
   ```c
   prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
   ```

2. **BPF filter complexity**: Writing BPF filters is complex
   - Consider using libraries (libseccomp)
   - Or use Docker's profiles as reference

3. **Syscall numbers**: Different architectures have different syscall numbers
   - Use `__NR_*` constants from `<sys/syscall.h>`

4. **Testing**: Some syscalls are used by libc, not directly
   - Blocking `socket` may break many programs

5. **Too restrictive**: Blocking too many syscalls breaks functionality

## Understanding Seccomp

Seccomp provides:
- **Syscall filtering**: Allow/deny specific system calls
- **Defense in depth**: Additional security layer
- **Attack prevention**: Blocks dangerous syscalls
- **Docker integration**: Docker uses seccomp by default

Use cases:
- **Containers**: Restrict what containers can do
- **Sandboxes**: Isolate untrusted code
- **Minimal environments**: Only allow needed syscalls

## Next Steps

In Step 8.1, we'll learn about Overlay filesystems, which is how Docker implements its layered filesystem. This allows efficient image storage and copy-on-write behavior.
