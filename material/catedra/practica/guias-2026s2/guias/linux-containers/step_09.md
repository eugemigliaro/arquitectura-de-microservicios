# Step 4.2: IPC Namespace

## Learning Objectives

- Understand IPC namespace isolation
- Create an IPC namespace
- Understand shared memory isolation
- Understand semaphore isolation
- Understand message queue isolation
- Verify IPC resource isolation

## Prerequisites

- Completion of Step 4.1 (UTS Namespace)
- Basic understanding of IPC mechanisms (shared memory, semaphores, message queues)
- Root access on a Linux system

## Background

### What is an IPC Namespace?

IPC (Inter-Process Communication) namespace isolates inter-process communication resources:
- **Shared memory segments** (shm): Memory shared between processes
- **Semaphores** (sem): Synchronization primitives
- **Message queues** (msg): Message passing between processes

### How IPC Namespaces Provide Resource Isolation

IPC namespaces create isolated IPC resource spaces:

1. **Resource ID Isolation**: Each namespace has its own IPC ID space:
   - Same ID can refer to different resources in different namespaces
   - ID 12345 in namespace A ≠ ID 12345 in namespace B
   - No conflicts between containers using same IDs

2. **Resource Visibility**: Processes can only see IPC resources in their namespace:
   - Cannot access shared memory from other containers
   - Cannot access semaphores from host
   - Cannot access message queues from other namespaces

3. **Key-based Access**: IPC resources are identified by keys:
   - Keys are namespace-specific
   - Same key in different namespaces creates different resources
   - Provides additional isolation layer

4. **Resource Persistence**: IPC resources persist until explicitly removed:
   - Survive process exit
   - Must be explicitly cleaned up with `ipcrm`
   - Important for proper cleanup

### Why IPC Namespace Matters

IPC namespace is important for:
- **Security**: Prevents containers from accessing each other's IPC resources
- **Isolation**: Allows multiple containers to use same IPC IDs without conflict
- **Application Compatibility**: Applications using IPC work correctly within namespace
- **Resource Management**: Prevents IPC resource leaks between containers

### IPC Namespace Isolation Benefits

IPC namespace provides:
- **Resource Isolation**: Containers can't access each other's IPC
- **ID Reuse**: Same IDs can be used in different namespaces
- **Security**: Prevents IPC-based attacks between containers
- **Compatibility**: Applications using IPC work normally within namespace
- **Clean Resource Management**: IPC resources are properly isolated

This is why Docker isolates IPC resources - it uses IPC namespaces to prevent containers from interfering with each other's shared memory, semaphores, and message queues.

## Key Commands Explained

Before diving into the practical steps, let's understand the key commands we'll be using:

### `ipcs` - List IPC Resources

The `ipcs` command lists inter-process communication resources (shared memory, semaphores, message queues).

**Syntax**: `ipcs [OPTIONS]`

**Key Options**:
- `-m`: Show shared memory segments
- `-s`: Show semaphores
- `-q`: Show message queues
- `-a`: Show all IPC resources
- `-i ID`: Show details for specific resource ID

**How it works**:
- Reads IPC resource information from kernel
- Shows resources in current IPC namespace
- Displays IDs, keys, owners, permissions, sizes

**Example**:
```bash
# List all IPC resources
ipcs -a

# List shared memory
ipcs -m

# List semaphores
ipcs -s

# List message queues
ipcs -q

# Show details for specific shared memory ID
ipcs -m -i 12345
```

**In IPC namespace context**:
- Shows only resources in current namespace
- Host IPC resources are not visible
- Resources from other containers are not visible
- Demonstrates namespace isolation

### `ipcmk` - Create IPC Resources

The `ipcmk` command creates IPC resources (shared memory, semaphores, message queues).

**Syntax**: `ipcmk [OPTIONS]`

**Key Options**:
- `-M SIZE`: Create shared memory segment of SIZE bytes
- `-S NUMSEMS`: Create semaphore set with NUMSEMS semaphores
- `-Q`: Create message queue

**How it works**:
- Creates IPC resource in current IPC namespace
- Returns the resource ID
- Resource persists until explicitly removed

**Example**:
```bash
# Create 1KB shared memory segment
ipcmk -M 1024
# Output: shmid: 12345

# Create semaphore set (number of semaphores is required)
ipcmk -S 1
# Output: semid: 67890

# Create message queue
ipcmk -Q
# Output: msqid: 11111
```

**In IPC namespace context**:
- Creates resource in current namespace only
- Not visible in other namespaces
- Can use same IDs in different namespaces

### `ipcrm` - Remove IPC Resources

The `ipcrm` command removes IPC resources.

**Syntax**: `ipcrm [OPTIONS] ID`

**Key Options**:
- `-m SHMID`: Remove shared memory segment
- `-s SEMID`: Remove semaphore set
- `-q MSQID`: Remove message queue
- `-a`: Remove all IPC resources (dangerous!)

**How it works**:
- Removes IPC resource by ID
- Resource must exist in current namespace
- Frees associated kernel memory

**Example**:
```bash
# Remove shared memory segment
ipcrm -m 12345

# Remove semaphore
ipcrm -s 67890

# Remove message queue
ipcrm -q 11111

# Remove all IPC resources (be careful!)
ipcrm -a
```

**Important notes**:
- Resources persist after process exit
- Must explicitly remove to free resources
- `-a` removes ALL resources (use with caution)
- Namespace-specific (only removes from current namespace)

## Commands

### Step 1: Check current IPC resources

```bash
# List shared memory segments
ipcs -m
# Shows shared memory segments

# List semaphores
ipcs -s
# Shows semaphores

# List message queues
ipcs -q
# Shows message queues

# List all IPC resources
ipcs -a
# Shows all IPC resources
```

### Step 2: Create an IPC namespace

```bash
# Ensure root access
sudo su

# Create an IPC namespace
unshare --ipc bash

# Verify IPC namespace
readlink /proc/self/ns/ipc
# Note this ID - compare with host namespace
```

### Step 3: Verify IPC isolation

```bash
# Check IPC resources in namespace
ipcs -a
# Should be empty (or show only resources created in this namespace)

# Compare with host (in another terminal)
ipcs -a
# Host should show its IPC resources (different from namespace)
```

### Step 4: Create shared memory segment

```bash
# Create a shared memory segment using ipcmk
ipcmk -M 1024
# Creates a 1KB shared memory segment
# Note the shmid (shared memory ID)

# List shared memory
ipcs -m
# Should show the segment you just created

# Get details
ipcs -m -i <shmid>
# Replace <shmid> with the ID from previous command
```

### Step 5: Test shared memory isolation

In the IPC namespace:

```bash
# Create a shared memory segment
SHMID=$(ipcmk -M 2048 | awk '{print $NF}')
echo "Created shared memory ID: $SHMID"

# Verify it exists
ipcs -m
# Should show the segment
```

In another terminal (host namespace):

```bash
# Check shared memory
ipcs -m
# Should NOT show the segment created in namespace
# This demonstrates isolation
```

### Step 6: Create semaphore

```bash
# In the IPC namespace, create a semaphore set (nsems required on modern ipcmk)
ipcmk -S 1
# Creates a semaphore set with 1 semaphore
# Note the semid

# List semaphores
ipcs -s
# Should show the semaphore you created

# Get details
ipcs -s -i <semid>
# Replace <semid> with the ID
```

### Step 7: Create message queue

```bash
# In the IPC namespace, create a message queue
ipcmk -Q
# Creates a message queue
# Note the msqid

# List message queues
ipcs -q
# Should show the message queue you created

# Get details
ipcs -q -i <msqid>
# Replace <msqid> with the ID
```

### Step 8: Test IPC with a program

```bash
# Create a simple C program to test shared memory
cat > /tmp/test_shm.c <<'EOF'
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdio.h>
#include <string.h>

int main() {
    key_t key = 1234;
    int shmid;
    char *shm;

    // Create shared memory
    shmid = shmget(key, 1024, IPC_CREAT | 0666);
    if (shmid < 0) {
        perror("shmget");
        return 1;
    }

    // Attach to shared memory
    shm = shmat(shmid, NULL, 0);
    if (shm == (char *)-1) {
        perror("shmat");
        return 1;
    }

    // Write to shared memory
    strcpy(shm, "Hello from IPC namespace!");

    printf("Shared memory ID: %d\n", shmid);
    printf("Written: %s\n", shm);

    // Detach
    shmdt(shm);

    return 0;
}
EOF

# Compile
gcc -o /tmp/test_shm /tmp/test_shm.c

# Run in IPC namespace
/tmp/test_shm

# Check shared memory
ipcs -m
# Should show the segment created by the program
```

### Step 9: Clean up IPC resources

```bash
# Remove shared memory segment
ipcrm -m <shmid>
# Replace <shmid> with the ID

# Remove semaphore
ipcrm -s <semid>
# Replace <semid> with the ID

# Remove message queue
ipcrm -q <msqid>
# Replace <msqid> with the ID

# Remove all IPC resources (be careful!)
ipcrm -a
```

### Step 10: Combine IPC with other namespaces

```bash
# Exit current namespace
exit

# Create IPC namespace with other namespaces
sudo su

# Create IPC + PID + mount namespaces
unshare --ipc --pid --mount --fork bash

# Verify IPC namespace
readlink /proc/self/ns/ipc
# Should be different from host

# Remount /proc
umount -l /proc 2>/dev/null || true
mount -t proc proc /proc

# Create IPC resource
ipcmk -M 1024
SHMID=$(ipcs -m | tail -1 | awk '{print $2}')
echo "Created shared memory ID: $SHMID in namespace"

# Verify isolation
ipcs -m
# Should only show resources in this namespace
```

### Step 11: Test IPC between processes in namespace

```bash
# Create a writer program
cat > /tmp/shm_writer.c <<'EOF'
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdio.h>
#include <string.h>

int main() {
    key_t key = 5678;
    int shmid = shmget(key, 1024, IPC_CREAT | 0666);
    char *shm = shmat(shmid, NULL, 0);
    strcpy(shm, "Data from writer process");
    printf("Writer: Written to shmid %d\n", shmid);
    shmdt(shm);
    return 0;
}
EOF

# Create a reader program
cat > /tmp/shm_reader.c <<'EOF'
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdio.h>

int main() {
    key_t key = 5678;
    int shmid = shmget(key, 1024, 0666);
    if (shmid < 0) {
        perror("shmget");
        return 1;
    }
    char *shm = shmat(shmid, NULL, 0);
    printf("Reader: Read from shmid %d: %s\n", shmid, shm);
    shmdt(shm);
    shmctl(shmid, IPC_RMID, NULL);
    return 0;
}
EOF

# Compile
gcc -o /tmp/shm_writer /tmp/shm_writer.c
gcc -o /tmp/shm_reader /tmp/shm_reader.c

# In IPC namespace, test communication
/tmp/shm_writer
/tmp/shm_reader
# Should work - processes can share IPC within namespace
```

## Verification

To verify IPC namespace isolation:

1. **IPC namespace ID**: `readlink /proc/self/ns/ipc` should be different from host
2. **Empty IPC**: New namespace should have no IPC resources (or only ones created in it)
3. **Isolation**: IPC resources created in namespace not visible on host
4. **Same IDs**: Can use same IPC IDs in different namespaces without conflict
5. **Process communication**: Processes in same namespace can share IPC resources

## Common Pitfalls

1. **IPC resources persist**: IPC resources survive process exit until explicitly removed
   ```bash
   # Clean up after testing
   ipcrm -a
   ```

2. **Permission issues**: IPC operations may require proper permissions
   ```bash
   # Use 0666 permissions for testing
   shmget(key, size, IPC_CREAT | 0666)
   ```

3. **Key conflicts**: Same key in different namespaces creates different resources
   ```bash
   # Keys are namespace-specific
   ```

4. **Not checking namespace**: Verify you're in the right namespace
   ```bash
   readlink /proc/self/ns/ipc
   ```

## Understanding IPC Isolation

IPC namespace provides:
- **Isolation**: Containers can't access each other's IPC
- **ID reuse**: Same IDs can be used in different namespaces
- **Security**: Prevents IPC-based attacks between containers
- **Compatibility**: Applications using IPC work normally within namespace

Use cases:
- Database systems using shared memory
- Applications using semaphores for synchronization
- Message-passing architectures
- High-performance computing applications

## Next Steps

In Step 5.1, we'll learn about user namespaces, which allow mapping user IDs without requiring root privileges. This is a crucial security feature that allows containers to run as "root" inside while being non-root on the host.
