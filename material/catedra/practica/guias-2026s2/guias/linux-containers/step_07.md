# Step 3.2: Bridge Networking

## Learning Objectives

- Understand Linux bridge networking
- Create a Linux bridge
- Connect multiple network namespaces to a bridge
- Configure NAT for external connectivity
- Test inter-container communication
- Test external connectivity
- Understand Docker's bridge network

## Prerequisites

- Completion of Step 3.1 (Network Namespace Basics)
- Understanding of Linux bridges
- Basic knowledge of iptables and NAT
- Root access on a Linux system

## Background

A Linux bridge is a virtual switch that connects multiple network interfaces:
- Acts like a physical network switch
- Forwards packets between connected interfaces
- Used by Docker to connect containers

Docker's default bridge network:
- Creates a bridge (usually `docker0`)
- Each container gets a veth pair
- One end in container, other end connected to bridge
- Bridge has an IP address (gateway for containers)
- NAT is configured for external connectivity

This setup allows:
- Containers to communicate with each other
- Containers to access the internet
- External access to containers (via port mapping)

## Commands

> **⚠️ Nota:** Si tenés Docker instalado o `br_netfilter` cargado, el tráfico entre containers via bridge pasa por iptables. Si los pings fallan silenciosamente, ejecutá:
> ```bash
> # Desactivar filtrado de bridge por iptables
> echo 0 > /proc/sys/net/bridge/bridge-nf-call-iptables 2>/dev/null || true
> # O cambiar la policy de FORWARD
> iptables -P FORWARD ACCEPT
> ```

### Step 1: Create a Linux bridge

```bash
# Ensure root access
sudo su

# Create a bridge
ip link add name br0 type bridge

# Bring up the bridge
ip link set br0 up

# Assign an IP address to the bridge (this will be the gateway)
ip addr add 172.16.0.1/16 dev br0

# Verify bridge exists
ip link show br0
# Should show: state UP

# Check bridge IP
ip addr show br0
# Should show: 172.16.0.1/16
```

### Step 2: Create network namespaces (containers)

```bash
# Create two network namespaces
ip netns add container1
ip netns add container2

# Verify
ip netns list
# Should show: container1 container2
```

### Step 3: Create veth pairs and connect to bridge

```bash
# Create veth pair for container1
ip link add veth-c1 type veth peer name veth-c1-br

# Create veth pair for container2
ip link add veth-c2 type veth peer name veth-c2-br

# Move container ends to namespaces
ip link set veth-c1 netns container1
ip link set veth-c2 netns container2

# Connect bridge ends to bridge
ip link set veth-c1-br master br0
ip link set veth-c2-br master br0

# Bring up bridge ends
ip link set veth-c1-br up
ip link set veth-c2-br up

# Verify bridge connections
bridge link show
# Should show both veth interfaces connected to br0
```

### Step 4: Configure container network interfaces

```bash
# Configure container1
ip netns exec container1 ip link set lo up
ip netns exec container1 ip link set veth-c1 up
ip netns exec container1 ip addr add 172.16.0.10/16 dev veth-c1

# Configure container2
ip netns exec container2 ip link set lo up
ip netns exec container2 ip link set veth-c2 up
ip netns exec container2 ip addr add 172.16.0.20/16 dev veth-c2

# Verify IPs
ip netns exec container1 ip addr show veth-c1
# Should show: 172.16.0.10/16

ip netns exec container2 ip addr show veth-c2
# Should show: 172.16.0.20/16
```

### Step 5: Configure routing in containers

```bash
# Set default gateway to bridge IP in container1
ip netns exec container1 ip route add default via 172.16.0.1

# Set default gateway to bridge IP in container2
ip netns exec container2 ip route add default via 172.16.0.1

# Verify routes
ip netns exec container1 ip route
# Should show: default via 172.16.0.1 dev veth-c1

ip netns exec container2 ip route
# Should show: default via 172.16.0.1 dev veth-c2
```

### Step 6: Test inter-container communication

```bash
# Ping container2 from container1
ip netns exec container1 ping -c 3 172.16.0.20
# Should work!

# Ping container1 from container2
ip netns exec container2 ping -c 3 172.16.0.10
# Should work!

# Test with a simple server
# Start a server in container1
ip netns exec container1 python3 -m http.server 8000 &
SERVER_PID=$!

# Test connection from container2
ip netns exec container2 curl -s http://172.16.0.10:8000 | head -5
# Should show HTML content

# Stop server
kill $SERVER_PID 2>/dev/null || true
```

### Step 7: Configure NAT for external connectivity

```bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Find host's external interface (usually eth0, ens33, or similar)
HOST_IF=$(ip route | grep default | awk '{print $5}' | head -1)
echo "Host interface: $HOST_IF"

# Configure iptables NAT rules
# Masquerade traffic from bridge network
iptables -t nat -A POSTROUTING -s 172.16.0.0/16 -o $HOST_IF -j MASQUERADE

# Allow forwarding from bridge to external
iptables -A FORWARD -i br0 -o $HOST_IF -j ACCEPT

# Allow forwarding from external to bridge (for return traffic)
iptables -A FORWARD -i $HOST_IF -o br0 -j ACCEPT

# Verify NAT rules
iptables -t nat -L POSTROUTING -n -v
# Should show MASQUERADE rule
```

### Step 8: Test external connectivity

```bash
# Test DNS resolution from container1
ip netns exec container1 ping -c 3 8.8.8.8
# Should work if NAT is configured correctly

# Test DNS (if DNS is configured)
ip netns exec container1 nslookup google.com 8.8.8.8
# Should resolve if DNS works

# Test HTTP access
ip netns exec container1 curl -s http://httpbin.org/ip
# Should return container's public IP (masqueraded as host IP)
```

### Step 9: Configure DNS in containers

```bash
# Create resolv.conf in namespaces
mkdir -p /etc/netns/container1
mkdir -p /etc/netns/container2

# Add DNS servers
echo "nameserver 8.8.8.8" > /etc/netns/container1/resolv.conf
echo "nameserver 8.8.4.4" >> /etc/netns/container1/resolv.conf

echo "nameserver 8.8.8.8" > /etc/netns/container2/resolv.conf
echo "nameserver 8.8.4.4" >> /etc/netns/container2/resolv.conf

# Test DNS
ip netns exec container1 ping -c 3 google.com
# Should resolve and ping
```

### Step 10: Port mapping (DNAT)

> **⚠️ Note:** `PREROUTING` DNAT only applies to packets arriving from outside the host.
> Traffic to `localhost` / `127.0.0.1` is locally originated and needs an **`OUTPUT`** DNAT rule as well (or test via another host / the bridge IP).

```bash
# Map host port 8080 to container1 port 80
# First, start a server in container1
ip netns exec container1 python3 -m http.server 80 &
SERVER_PID=$!

# DNAT for packets arriving from other hosts
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 172.16.0.10:80
# DNAT for locally originated connections (curl localhost / 127.0.0.1)
iptables -t nat -A OUTPUT -p tcp --dport 8080 -j DNAT --to-destination 172.16.0.10:80

# Allow forwarding for this connection
iptables -A FORWARD -p tcp -d 172.16.0.10 --dport 80 -j ACCEPT

# Ensure reply path / hairpin works
iptables -t nat -A POSTROUTING -d 172.16.0.10 -p tcp --dport 80 -j MASQUERADE 2>/dev/null || true

# Test from host (may take a moment)
curl -s --max-time 5 http://127.0.0.1:8080 | head -5
# Should show content from container1
# Fallback if DNAT is awkward in your VM: curl -s http://172.16.0.10:80 | head -5

# Clean up
kill $SERVER_PID 2>/dev/null || true
iptables -t nat -D PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 172.16.0.10:80 2>/dev/null || true
iptables -t nat -D OUTPUT -p tcp --dport 8080 -j DNAT --to-destination 172.16.0.10:80 2>/dev/null || true
iptables -t nat -D POSTROUTING -d 172.16.0.10 -p tcp --dport 80 -j MASQUERADE 2>/dev/null || true
iptables -D FORWARD -p tcp -d 172.16.0.10 --dport 80 -j ACCEPT 2>/dev/null || true
```

### Step 11: Complete setup script

```bash
# Create a script that sets up bridge networking
cat > /tmp/setup_bridge.sh <<'SCRIPT'
#!/bin/bash
set -e

BRIDGE_NAME="br0"
BRIDGE_IP="172.16.0.1/16"
SUBNET="172.16.0.0/16"

# Create bridge
ip link add name $BRIDGE_NAME type bridge
ip link set $BRIDGE_NAME up
ip addr add $BRIDGE_IP dev $BRIDGE_NAME

# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# Get host interface
HOST_IF=$(ip route | grep default | awk '{print $5}' | head -1)

# Configure NAT
iptables -t nat -A POSTROUTING -s $SUBNET -o $HOST_IF -j MASQUERADE
iptables -A FORWARD -i $BRIDGE_NAME -o $HOST_IF -j ACCEPT
iptables -A FORWARD -i $HOST_IF -o $BRIDGE_NAME -j ACCEPT

echo "Bridge $BRIDGE_NAME created with IP $BRIDGE_IP"
echo "NAT configured for interface $HOST_IF"
SCRIPT

chmod +x /tmp/setup_bridge.sh
```

### Step 12: Clean up

```bash
# Delete network namespaces (removes interfaces)
ip netns delete container1
ip netns delete container2

# Remove iptables rules
HOST_IF=$(ip route | grep default | awk '{print $5}' | head -1)
iptables -t nat -D POSTROUTING -s 172.16.0.0/16 -o $HOST_IF -j MASQUERADE 2>/dev/null || true
iptables -D FORWARD -i br0 -o $HOST_IF -j ACCEPT 2>/dev/null || true
iptables -D FORWARD -i $HOST_IF -o br0 -j ACCEPT 2>/dev/null || true

# Delete bridge
ip link set br0 down
ip link delete br0

# Verify cleanup
ip link show br0
# Should show: Device "br0" does not exist
```

## Verification

To verify bridge networking:

1. **Bridge exists**: `ip link show br0` should show bridge is UP
2. **Interfaces connected**: `bridge link show` should show veth interfaces
3. **Inter-container ping**: Containers can ping each other
4. **External connectivity**: Containers can ping external IPs (8.8.8.8)
5. **NAT working**: External requests appear to come from host IP
6. **Port mapping**: Host port forwarding works to container ports

## Common Pitfalls

1. **Bridge not up**: Bridge must be brought up
   ```bash
   ip link set br0 up
   ```

2. **IP forwarding disabled**: Must enable IP forwarding
   ```bash
   echo 1 > /proc/sys/net/ipv4/ip_forward
   ```

3. **NAT not configured**: Need MASQUERADE rule for external access
   ```bash
   iptables -t nat -A POSTROUTING -s 172.16.0.0/16 -o eth0 -j MASQUERADE
   ```

4. **FORWARD rules missing**: Need to allow forwarding
   ```bash
   iptables -A FORWARD -i br0 -o eth0 -j ACCEPT
   ```

5. **Default gateway not set**: Containers need default route
   ```bash
   ip netns exec container1 ip route add default via 172.16.0.1
   ```

## Understanding Docker's Bridge Network

Docker's default bridge works exactly like this:
- Creates `docker0` bridge (like our `br0`)
- Each container gets a veth pair
- Containers get IPs from bridge subnet
- NAT allows external access
- Port mapping uses DNAT rules

This is essentially what we just built manually!

## Next Steps

In Step 4.1, we'll learn about UTS namespaces, which isolate hostname and domain name. This allows each container to have its own hostname, independent of the host system.
