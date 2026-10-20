# Step 3.1: Network Namespace Basics

## Learning Objectives

- Understand network namespace isolation
- Create a new network namespace
- Verify network isolation (no interfaces)
- Create virtual Ethernet pairs (veth)
- Connect namespaces with virtual interfaces
- Configure IP addresses and routing

## Prerequisites

- Completion of Step 2.2 (Combining PID and Mount Namespaces)
- Basic understanding of Linux networking
- Understanding of IP addresses and routing
- Root access on a Linux system

## Background

### What is a Network Namespace?

A network namespace isolates the network stack, providing complete network isolation between containers and the host. Each network namespace has:
- Its own network interfaces (completely separate from host)
- Its own routing table (independent routing decisions)
- Its own firewall rules (iptables rules are namespace-specific)
- Its own network statistics and configuration

### How Network Namespaces Provide Network Isolation

Network namespaces create isolated network environments:

1. **Interface Isolation**: Each namespace starts with only a loopback interface (`lo`). It cannot see:
   - Host network interfaces (eth0, wlan0, etc.)
   - Interfaces from other containers
   - Network interfaces in other namespaces

2. **Routing Isolation**: Each namespace has its own routing table:
   - Independent default routes
   - Namespace-specific routes
   - No access to host routing table
   - Routing decisions are isolated

3. **Firewall Isolation**: iptables rules are namespace-specific:
   - Rules in one namespace don't affect others
   - Each namespace can have its own firewall configuration
   - Host firewall rules don't apply to container namespaces

4. **Network Statistics**: Each namespace tracks its own:
   - Packet counts
   - Bandwidth usage
   - Network errors
   - Interface statistics

### Virtual Ethernet Pairs (veth)

Virtual Ethernet pairs (veth) are used to connect namespaces:
- `veth` devices always come in pairs (like a virtual network cable)
- One end in one namespace, other end in another namespace
- Data sent on one end appears on the other end
- Used to connect containers to bridges or other namespaces

**How veth works**:
```
Host Namespace          Container Namespace
    veth0 <==========> veth1
  10.0.0.1           10.0.0.2
```

- Packets sent on `veth0` appear on `veth1`
- Packets sent on `veth1` appear on `veth0`
- Like a virtual network cable connecting two namespaces

### Network Namespace Isolation Benefits

Network namespace provides:
- **Complete Network Isolation**: Containers can't see host network
- **Independent IP Addresses**: Each container can have its own IP
- **Isolated Routing**: Containers have independent routing tables
- **Security**: Network attacks in container don't affect host
- **Resource Management**: Can limit bandwidth per namespace

This is how Docker containers get network connectivity - each container has its own network namespace, connected to a bridge via veth pairs. This provides isolation while allowing connectivity.

## Key Commands Explained

Before diving into the practical steps, let's understand the key commands we'll be using:

### `ip` - Network Configuration Tool

The `ip` command is a powerful tool for network configuration. It's part of the `iproute2` package and replaces older tools like `ifconfig` and `route`.

**Syntax**: `ip [OPTIONS] OBJECT COMMAND`

**Key Objects**:
- `link`: Network interfaces
- `addr`: IP addresses
- `route`: Routing table
- `netns`: Network namespaces

**Common Commands**:

**Network Interfaces**:
```bash
# List all interfaces
ip link show

# Bring interface up/down
ip link set eth0 up
ip link set eth0 down

# Show interface details
ip addr show eth0
```

**IP Addresses**:
```bash
# Add IP address
ip addr add 10.0.0.1/24 dev eth0

# Remove IP address
ip addr del 10.0.0.1/24 dev eth0

# Show all addresses
ip addr show
```

**Routing**:
```bash
# Show routing table
ip route show

# Add route
ip route add 192.168.1.0/24 via 10.0.0.1

# Add default route
ip route add default via 10.0.0.1
```

**Network Namespaces**:
```bash
# Create namespace
ip netns add netns1

# Execute command in namespace
ip netns exec netns1 ip addr show

# List namespaces
ip netns list

# Delete namespace
ip netns delete netns1
```

### `ip netns` - Network Namespace Management

The `ip netns` command manages network namespaces. It's the standard way to create and manage network namespaces.

**Syntax**: `ip netns COMMAND [NAME]`

**Key Commands**:
- `add NAME`: Create a new network namespace
- `delete NAME`: Delete a network namespace
- `list`: List all network namespaces
- `exec NAME COMMAND`: Execute command in namespace

**How it works**:
- Creates namespaces in `/var/run/netns/`
- Allows persistent namespaces (survive process exit)
- Provides easy way to execute commands in namespaces

**Example**:
```bash
# Create namespace
ip netns add container1

# Execute command in namespace
ip netns exec container1 ip addr show

# List namespaces
ip netns list

# Delete namespace
ip netns delete container1
```

**Why `/var/run/netns/` is needed**:
- Network namespaces are normally tied to processes
- When process exits, namespace is deleted
- `ip netns` creates bind mounts to persist namespaces
- Allows namespaces to exist without running processes

### `veth` - Virtual Ethernet Pair

Virtual Ethernet pairs are created using `ip link add type veth`. They always come in pairs and connect namespaces.

**Syntax**: `ip link add NAME1 type veth peer name NAME2`

**How it works**:
- Creates two virtual network interfaces
- They are connected like a network cable
- Data sent on one appears on the other
- One end can be moved to another namespace

**Example**:
```bash
# Create veth pair
ip link add veth0 type veth peer name veth1

# Move one end to namespace
ip link set veth1 netns container1

# Configure IPs
ip addr add 10.0.0.1/24 dev veth0
ip netns exec container1 ip addr add 10.0.0.2/24 dev veth1

# Bring interfaces up
ip link set veth0 up
ip netns exec container1 ip link set veth1 up
```

**Why veth pairs are used**:
- Connect isolated network namespaces
- Allow communication between namespaces
- Used to connect containers to bridges
- Essential for container networking

### `ping` - Test Network Connectivity

The `ping` command tests network connectivity by sending ICMP echo requests.

**Syntax**: `ping [OPTIONS] HOST`

**Key Options**:
- `-c COUNT`: Send COUNT packets and stop
- `-i INTERVAL`: Wait INTERVAL seconds between packets
- `-W TIMEOUT`: Wait TIMEOUT seconds for reply

**Example**:
```bash
# Ping with count
ping -c 3 8.8.8.8

# Ping from namespace
ip netns exec container1 ping -c 3 10.0.0.2
```

**In network namespace context**:
- Tests connectivity within namespace
- Can ping other interfaces in same namespace
- Can ping interfaces in connected namespaces
- Cannot ping host interfaces (isolation)

## Commands

### Step 1: Create a network namespace

```bash
# Ensure root access
sudo su

# Note: ip netns requires /var/run/netns directory to exist
# Create it if it doesn't exist (some systems may need this)
mkdir -p /var/run/netns

# Create a new network namespace
ip netns add netns1

# Verify it was created
ip netns list
# Should show: netns1
```

### Step 2: Verify network isolation

```bash
# Check interfaces in the namespace
ip netns exec netns1 ip addr
# Should show: only loopback (lo) interface, no other interfaces

# Compare with host
ip addr
# Host should show multiple interfaces (eth0, wlan0, etc.)

# Check routing table in namespace
ip netns exec netns1 ip route
# Should be empty or only loopback routes

# Compare with host
ip route
# Host should show default routes
```

### Step 3: Bring up loopback interface

```bash
# Bring up lo interface in the namespace
ip netns exec netns1 ip link set lo up

# Verify
ip netns exec netns1 ip addr show lo
# Should show: state UP
```

### Step 4: Create a virtual Ethernet pair

```bash
# Create a veth pair
# veth0 will be in host namespace
# veth1 will be moved to netns1
ip link add veth0 type veth peer name veth1

# Verify both ends exist in host namespace
ip link show veth0
ip link show veth1
# Both should exist

# Move veth1 to netns1
ip link set veth1 netns netns1

# Verify veth1 is now in netns1
ip netns exec netns1 ip link show veth1
# Should show veth1

# Verify veth0 is still in host
ip link show veth0
# Should show veth0
```

### Step 5: Configure IP addresses

```bash
# Assign IP to veth0 (host side)
ip addr add 10.0.0.1/24 dev veth0
ip link set veth0 up

# Assign IP to veth1 (namespace side)
ip netns exec netns1 ip addr add 10.0.0.2/24 dev veth1
ip netns exec netns1 ip link set veth1 up

# Verify IPs
ip addr show veth0
# Should show: 10.0.0.1/24

ip netns exec netns1 ip addr show veth1
# Should show: 10.0.0.2/24
```

### Step 6: Test connectivity

```bash
# Ping from host to namespace
ping -c 3 10.0.0.2
# Should work!

# Ping from namespace to host
ip netns exec netns1 ping -c 3 10.0.0.1
# Should work!

# Check connectivity from namespace
ip netns exec netns1 ping -c 3 10.0.0.2
# Should ping itself (via veth1)
```

### Step 7: Create a second namespace and connect them

> **⚠️ Note:** Putting two host-side veths on the **same** `/24` (e.g. both `10.0.0.0/24`) does **not** reliably route between namespaces — the host treats them as the same L2 subnet and will not forward. Use **different subnets** (below) or a **bridge** (Step 3.2).

```bash
# Create second namespace
ip netns add netns2

# Create veth pair for netns2
ip link add veth2 type veth peer name veth3

# Move veth3 to netns2
ip link set veth3 netns netns2

# Configure IPs on a *different* subnet than netns1 (10.0.0.0/24)
ip addr add 10.0.1.1/24 dev veth2
ip link set veth2 up

ip netns exec netns2 ip link set lo up
ip netns exec netns2 ip addr add 10.0.1.2/24 dev veth3
ip netns exec netns2 ip link set veth3 up
```

### Step 8: Set up routing

```bash
# Enable IP forwarding on host
echo 1 > /proc/sys/net/ipv4/ip_forward

# Routes: each namespace reaches the other subnet via its veth → host
ip netns exec netns1 ip route add 10.0.1.0/24 via 10.0.0.1
ip netns exec netns2 ip route add 10.0.0.0/24 via 10.0.1.1

# Test connectivity between namespaces via host routing
ip netns exec netns1 ping -c 3 10.0.1.2
# Should work now
```

### Step 9: Create namespace using unshare

```bash
# Alternative: Create network namespace using unshare
unshare --net bash

# Verify you're in a new network namespace
ip addr
# Should only show loopback

# Check namespace ID
readlink /proc/self/ns/net
# Should be different from host
```

### Step 10: Clean up

```bash
# Delete network namespaces (this removes all interfaces in them)
ip netns delete netns1
ip netns delete netns2

# Or delete all
ip netns list | xargs -I {} ip netns delete {}

# Verify cleanup
ip netns list
# Should be empty
```

## Verification

To verify network namespace isolation:

1. **No interfaces**: New namespace should have no interfaces (except loopback)
2. **Isolated routing**: Routing table should be empty or minimal
3. **veth connectivity**: Can ping between namespaces via veth pairs
4. **Namespace ID**: `readlink /proc/self/ns/net` should be different from host
5. **Isolation**: Processes in namespace can't see host network interfaces

## Common Pitfalls

1. **Forgetting to bring up interfaces**: Interfaces must be up to work
   ```bash
   ip link set veth0 up
   ```

2. **Wrong IP configuration**: Ensure IPs are on correct interfaces
   ```bash
   ip addr add 10.0.0.1/24 dev veth0
   ```

3. **IP forwarding disabled**: Host must forward packets between namespaces
   ```bash
   echo 1 > /proc/sys/net/ipv4/ip_forward
   ```

4. **Routing not configured**: Namespaces need routes to reach each other
   ```bash
   ip netns exec netns1 ip route add 10.0.0.0/24 dev veth1
   ```

5. **veth pair not moved**: Must move one end to target namespace
   ```bash
   ip link set veth1 netns netns1
   ```

## Understanding veth Pairs

veth pairs are like virtual network cables:
- Always created in pairs: `ip link add veth0 type veth peer name veth1`
- One end stays in host, other end moves to namespace
- Data sent on one end appears on the other end
- Used to connect namespaces together

Visual representation:
```
Host Namespace          netns1
    veth0 <==========> veth1
  10.0.0.1           10.0.0.2
```

## Next Steps

In Step 3.2, we'll learn about bridge networking, which is how Docker connects multiple containers. We'll create a Linux bridge, connect multiple network namespaces to it, and configure NAT for external connectivity - this is essentially Docker's bridge network mode.
