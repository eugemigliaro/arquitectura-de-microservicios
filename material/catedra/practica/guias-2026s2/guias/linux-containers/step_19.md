# Step 8.2: Container Runtime Interface

## Learning Objectives

- Understand OCI runtime specification
- Examine OCI bundle structure
- Create a basic OCI bundle
- Use `runc` to run containers
- Compare with manual namespace creation
- Understand container runtime interface

## Prerequisites

- Completion of Step 8.1 (Overlay Filesystems)
- Understanding of container runtimes
- Root access on a Linux system
- `runc` installed (or instructions to install)

## Background

OCI (Open Container Initiative) Runtime Specification:
- **Standard format** for container runtimes
- **Bundle structure**: rootfs + config.json
- **Runtime interface**: How to create/start/delete containers

`runc` is the reference OCI runtime:
- Used by Docker, Podman, etc.
- Implements OCI spec
- Low-level container runtime

This is what actually creates containers - Docker and other tools use runc under the hood.

## Commands

### Step 1: Install runc

```bash
# Check if runc is installed
which runc
runc --version

# If not installed, install it
# On Debian/Ubuntu:
sudo apt-get update
sudo apt-get install -y runc

# Or build from source (more recent version):
# git clone https://github.com/opencontainers/runc.git
# cd runc
# make
# sudo make install

# Verify installation
runc --version
```

### Step 2: Understand OCI bundle structure

```bash
# OCI bundle consists of:
# 1. rootfs/ - Root filesystem
# 2. config.json - Container configuration

# Create a bundle directory
mkdir -p /tmp/oci-bundle
cd /tmp/oci-bundle

# Create rootfs directory
mkdir rootfs
```

### Step 3: Create a minimal rootfs

```bash
# Create minimal root filesystem
cd /tmp/oci-bundle/rootfs

# Create directory structure
mkdir -p bin lib lib64 etc proc sys dev tmp usr/bin usr/lib root

# On Ubuntu 24.04, /bin is a symlink to /usr/bin. runc uses pivot_root
# internally, so bash resolves commands via PATH to /usr/bin/. We mirror
# that structure with a symlink so binaries are found regardless of PATH.
rm -rf /tmp/oci-bundle/rootfs/usr/bin
ln -s ../../bin /tmp/oci-bundle/rootfs/usr/bin

# Copy essential binaries
cp /bin/bash bin/
cp /bin/ls bin/
cp /bin/cat bin/
cp /bin/sh bin/ 2>/dev/null || ln -s /bin/bash bin/sh

# Copy libraries for bash (and other binaries)
for binary in /bin/bash /bin/ls /bin/cat; do
    ldd "$binary" 2>/dev/null | grep -o '/[^ ]*' | while read lib; do
        mkdir -p "/tmp/oci-bundle/rootfs$(dirname "$lib")"
        cp "$lib" "/tmp/oci-bundle/rootfs$lib" 2>/dev/null || true
    done
done

# Create basic config files
cat > etc/passwd <<EOF
root:x:0:0:root:/root:/bin/bash
EOF

cat > etc/group <<EOF
root:x:0:
EOF

cat > etc/hosts <<EOF
127.0.0.1 localhost
EOF
```

### Step 4: Generate OCI config.json

```bash
# Use runc spec to generate default config.json
cd /tmp/oci-bundle
runc spec

# Verify config.json was created
ls -la config.json
cat config.json | head -50
# Shows OCI configuration in JSON format
```

### Step 5: Examine config.json

```bash
# Look at the generated config
cat config.json | python3 -m json.tool | head -100
# Or use jq if available
# cat config.json | jq .

# Key sections:
# - root: rootfs path
# - process: command to run, env, cwd
# - linux: namespaces, cgroups, capabilities, etc.
# - mounts: filesystem mounts
```

### Step 6: Modify config.json

```bash
# Edit config.json to customize
# Example: Change the command
cat > /tmp/modify_config.py <<'EOF'
import json

with open('/tmp/oci-bundle/config.json', 'r') as f:
    config = json.load(f)

# Change command
config['process']['args'] = ['/bin/bash']

# Set working directory
config['process']['cwd'] = '/root'

# Add environment variable
config['process']['env'].append('CONTAINER=oci-test')

# Save
with open('/tmp/oci-bundle/config.json', 'w') as f:
    json.dump(config, f, indent=2)

print("Config modified")
EOF

python3 /tmp/modify_config.py

# Or edit manually with your preferred editor
# nano /tmp/oci-bundle/config.json
```

### Step 7: Run container with runc

```bash
# Ensure root access
sudo su

# Create container (this creates the container but doesn't start it)
cd /tmp/oci-bundle
runc create mycontainer

# List containers
runc list
# Should show: mycontainer (created)

# Start the container
runc start mycontainer
# Container starts and runs the command from config.json

# In another terminal, check it's running
runc list
# Should show: mycontainer (running)

# Or check processes
ps aux | grep mycontainer
```

### Step 8: Interact with running container

```bash
# Execute a command in running container
runc exec mycontainer ls -la /

# Execute interactive shell
runc exec -t mycontainer /bin/bash

# Check container state
runc state mycontainer
# Shows container state in JSON
```

### Step 9: Stop and delete container

```bash
# Stop the container (sends signal, then removes)
runc kill mycontainer

# Or kill with specific signal
# runc kill mycontainer KILL

# Delete the container
runc delete mycontainer

# Verify deletion
runc list
# Should be empty
```

### Step 10: Configure namespaces in config.json

```bash
# Create a new bundle with custom namespaces
mkdir -p /tmp/oci-bundle2
cd /tmp/oci-bundle2

# Create rootfs (simplified - reuse or create new)
mkdir rootfs
# ... (create rootfs as before)

# Generate spec
runc spec

# Modify config to add namespaces
cat > /tmp/setup_namespaces.py <<'EOF'
import json

with open('/tmp/oci-bundle2/config.json', 'r') as f:
    config = json.load(f)

# Ensure namespaces are configured
# Namespaces are usually in linux.namespaces
if 'linux' not in config:
    config['linux'] = {}
if 'namespaces' not in config['linux']:
    config['linux']['namespaces'] = []

# Add namespaces (if not present)
ns_types = ['pid', 'ipc', 'uts', 'mount', 'network', 'user']
existing = [ns['type'] for ns in config['linux'].get('namespaces', [])]

for ns_type in ns_types:
    if ns_type not in existing:
        config['linux']['namespaces'].append({'type': ns_type})

# Save
with open('/tmp/oci-bundle2/config.json', 'w') as f:
    json.dump(config, f, indent=2)

print("Namespaces configured")
EOF

python3 /tmp/setup_namespaces.py
```

### Step 11: Configure cgroups in config.json

```bash
# Add cgroup configuration
cat > /tmp/setup_cgroups.py <<'EOF'
import json

with open('/tmp/oci-bundle2/config.json', 'r') as f:
    config = json.load(f)

# Add cgroup resources
if 'linux' not in config:
    config['linux'] = {}
if 'resources' not in config['linux']:
    config['linux']['resources'] = {}

# Set memory limit (100MB)
config['linux']['resources']['memory'] = {
    'limit': 104857600,  # 100MB in bytes
}

# Set CPU limit (50%)
config['linux']['resources']['cpu'] = {
    'quota': 50000,
    'period': 100000,
}

# Save
with open('/tmp/oci-bundle2/config.json', 'w') as f:
    json.dump(config, f, indent=2)

print("Cgroups configured")
EOF

python3 /tmp/setup_cgroups.py
```

### Step 12: Configure capabilities

```bash
# Drop capabilities (security)
cat > /tmp/setup_capabilities.py <<'EOF'
import json

with open('/tmp/oci-bundle2/config.json', 'r') as f:
    config = json.load(f)

# Add capabilities
if 'process' not in config:
    config['process'] = {}
if 'capabilities' not in config['process']:
    config['process']['capabilities'] = {}

# Drop all capabilities, then add only what's needed
config['process']['capabilities'] = {
    'bounding': [
        'CAP_CHOWN',
        'CAP_DAC_OVERRIDE',
        'CAP_FOWNER',
        'CAP_FSETID',
        'CAP_KILL',
        'CAP_SETGID',
        'CAP_SETUID',
        'CAP_SETPCAP',
        'CAP_NET_BIND_SERVICE',
        'CAP_NET_RAW',
        'CAP_SYS_CHROOT',
        'CAP_MKNOD',
        'CAP_AUDIT_WRITE',
        'CAP_SETFCAP',
    ],
    'effective': [
        'CAP_CHOWN',
        'CAP_DAC_OVERRIDE',
        'CAP_FOWNER',
        'CAP_FSETID',
        'CAP_KILL',
        'CAP_SETGID',
        'CAP_SETUID',
        'CAP_SETPCAP',
        'CAP_NET_BIND_SERVICE',
        'CAP_NET_RAW',
        'CAP_SYS_CHROOT',
        'CAP_MKNOD',
        'CAP_AUDIT_WRITE',
        'CAP_SETFCAP',
    ],
    'inheritable': [
        'CAP_CHOWN',
        'CAP_DAC_OVERRIDE',
        'CAP_FOWNER',
        'CAP_FSETID',
        'CAP_KILL',
        'CAP_SETGID',
        'CAP_SETUID',
        'CAP_SETPCAP',
        'CAP_NET_BIND_SERVICE',
        'CAP_NET_RAW',
        'CAP_SYS_CHROOT',
        'CAP_MKNOD',
        'CAP_AUDIT_WRITE',
        'CAP_SETFCAP',
    ],
    'permitted': [
        'CAP_CHOWN',
        'CAP_DAC_OVERRIDE',
        'CAP_FOWNER',
        'CAP_FSETID',
        'CAP_KILL',
        'CAP_SETGID',
        'CAP_SETUID',
        'CAP_SETPCAP',
        'CAP_NET_BIND_SERVICE',
        'CAP_NET_RAW',
        'CAP_SYS_CHROOT',
        'CAP_MKNOD',
        'CAP_AUDIT_WRITE',
        'CAP_SETFCAP',
    ],
}

# Save
with open('/tmp/oci-bundle2/config.json', 'w') as f:
    json.dump(config, f, indent=2)

print("Capabilities configured")
EOF

python3 /tmp/setup_capabilities.py
```

### Step 13: Run container with full configuration

```bash
# Create and run container with all configurations
cd /tmp/oci-bundle2
runc create fullcontainer
runc start fullcontainer

# Check it's running
runc list

# Check cgroup (should have memory limit)
CONTAINER_PID=$(runc state fullcontainer | python3 -c "import sys, json; print(json.load(sys.stdin)['pid'])")
cat /proc/$CONTAINER_PID/cgroup
# Should show cgroup path

# Check memory limit
CGROUP_PATH=$(cat /proc/$CONTAINER_PID/cgroup | cut -d: -f3)
cat /sys/fs/cgroup$CGROUP_PATH/memory.max
# Should show: 104857600 (100MB)

# Stop and delete
runc kill fullcontainer
runc delete fullcontainer
```

### Step 14: Compare with manual setup

```bash
# What we did manually in previous steps:
# - Created namespaces with unshare
# - Set up filesystem with pivot_root
# - Configured network with ip netns
# - Set cgroup limits manually
# - Dropped capabilities with capsh

# What runc does:
# - All of the above, but automated
# - Uses OCI spec for configuration
# - Handles container lifecycle
# - Manages state

# runc is essentially a wrapper around all the manual steps!
```

### Step 15: Clean up

```bash
# Remove test bundles
rm -rf /tmp/oci-bundle /tmp/oci-bundle2
rm -f /tmp/modify_config.py /tmp/setup_*.py

# List any remaining containers
runc list
# Should be empty
```

## Verification

To verify OCI runtime:

1. **runc installed**: `runc --version` works
2. **Bundle created**: rootfs and config.json exist
3. **Container created**: `runc create` succeeds
4. **Container runs**: `runc start` starts container
5. **Config applied**: Namespaces, cgroups, capabilities work
6. **State managed**: `runc state` shows container state

## Common Pitfalls

1. **runc not installed**: Install runc package
   ```bash
   sudo apt-get install runc
   ```

2. **Invalid config.json**: Must be valid JSON
   ```bash
   python3 -m json.tool config.json  # Validate
   ```

3. **Rootfs missing**: Must have rootfs directory
   ```bash
   mkdir rootfs
   # Populate it
   ```

4. **Permissions**: Need root for some operations
   ```bash
   sudo runc create ...
   ```

5. **Container already exists**: Delete before recreating
   ```bash
   runc delete mycontainer
   ```

## Understanding OCI Runtime

OCI Runtime provides:
- **Standard format**: OCI spec for containers
- **Runtime interface**: create/start/stop/delete
- **Configuration**: JSON-based config
- **Interoperability**: Works with Docker, Podman, etc.

runc implements:
- **All namespaces**: PID, mount, network, UTS, IPC, user
- **Cgroups**: Resource limits
- **Capabilities**: Security
- **Lifecycle**: Container management

## Next Steps

In Step 9.1, we'll put everything together and create a complete container script that combines all the concepts we've learned: namespaces, cgroups, capabilities, and more.
