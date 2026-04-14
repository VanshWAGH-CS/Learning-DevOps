# Docker Networking - Complete Reference Guide

Docker networking enables containers to communicate with each other and with external systems. Docker provides several network types to suit different use cases.

---

## Types of Networks in Docker

### 1. Default Bridge Network

The default network in Docker. It creates a private network on the host machine. Containers connected to the bridge network can communicate with each other using container names as hostnames. IP addresses can also be used.

**Command:**
```bash
docker run --network=bridge my-container
```

**Characteristics:**
- Default network mode
- Private network on host machine
- Containers can communicate using container names as hostnames
- Also supports IP-based communication

---

### 2. Host Network

With this network mode, containers share the host's network stack, bypassing network isolation. Containers using the host network can access services running on the host directly without any port mapping.

**Command:**
```bash
docker run --network=host my-container
```

**Characteristics:**
- Shares host's network stack
- No network isolation
- No port mapping required
- Best for performance-critical applications
- Useful when network isolation is not a concern

---

### 3. Overlay Network

Overlay networks are used for communication between multiple Docker daemon hosts. This type of network allows containers running on different hosts to communicate seamlessly as if they were on the same network. Overlay networks use VXLAN (Virtual Extensible LAN) encapsulation to achieve network connectivity across multiple hosts.

**Commands:**
```bash
# Create an overlay network
docker network create --driver=overlay my-overlay-network

# Run a service with overlay network
docker service create --network=my-overlay-network my-service
```

**Characteristics:**
- Multi-host communication
- Uses VXLAN encapsulation
- Containers on different hosts communicate as if on same network
- Essential for Docker Swarm and container orchestration

---

### 4. Macvlan Network

Macvlan networks allow containers to have a MAC address assigned directly to them. This allows containers to appear as physical devices on the network, enabling them to be assigned IP addresses from the physical network's subnet.

**Command:**
```bash
docker network create -d macvlan \
  --subnet=<subnet> \
  --gateway=<gateway> \
  -o parent=<network-interface> \
  my-macvlan-network

docker run --network=my-macvlan-network my-container
```

**Characteristics:**
- Direct MAC address assignment to containers
- Containers appear as physical devices on the network
- IP addresses assigned from physical network subnet
- Direct Layer 2 network access
- Connect containers directly to external networks

---

### 5. Custom Bridge Network

Creates a private network on the host machine, allowing containers to communicate with each other using container names as hostnames. Containers connected to the same bridge network can reach each other via IP addresses.

**Commands:**
```bash
# Create a custom bridge network
docker network create my-bridge-network

# Run container with custom bridge network
docker run --network=my-bridge-network my-container
```

**Characteristics:**
- User-defined private network
- Container name resolution built-in
- Better isolation than default bridge
- Automatic DNS resolution between containers

---

### 6. None Network

The none network mode disables networking for a container. Containers running in this mode have no network interfaces and are completely isolated from the network.

**Command:**
```bash
docker run --network=none my-container
```

**Characteristics:**
- Complete network isolation
- No network interfaces
- Useful for fully isolated environments
- Security-focused use cases

---

## Quick Comparison Table

| Network Type | Use Case | Isolation | Multi-Host |
|--------------|----------|-----------|------------|
| **Default Bridge** | Single host, basic communication | Moderate | No |
| **Host** | Performance-critical, no isolation needed | None | No |
| **Overlay** | Multi-host communication, Swarm services | High | Yes |
| **Macvlan** | Direct L2 access, legacy systems | Low | Yes |
| **Custom Bridge** | User-defined networks, container naming | Moderate | No |
| **None** | Fully isolated containers | Complete | No |

---

## Useful Network Management Commands

```bash
# List all networks
docker network ls

# Inspect a network
docker network inspect <network_name>

# Create a custom network
docker network create <network_name>

# Remove a network
docker network rm <network_name>

# Connect a container to a network
docker network connect <network_name> <container_name>

# Disconnect a container from a network
docker network disconnect <network_name> <container_name>

# Prune unused networks
docker network prune
```

---

## Best Practices

1. **Use Custom Bridge Networks** instead of the default bridge for better isolation and automatic DNS resolution.

2. **Use Overlay Networks** for multi-host communication and production Docker Swarm deployments.

3. **Use Host Network** only when performance is critical and network isolation is not required.

4. **Use Macvlan** when containers need to be directly accessible on the physical network (legacy integration).

5. **Use None Network** for batch jobs or security-sensitive containers that don't need network access.

---

> **Note**: Default bridge network does not support automatic DNS resolution between containers. Use custom bridge networks for container name-based communication.

*Reference: Docker Networking Notes for DevOps Engineers*
