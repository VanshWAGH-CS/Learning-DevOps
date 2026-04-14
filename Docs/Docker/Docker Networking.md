# Docker Networking - Complete Reference Guide

Docker networking enables containers to communicate with each other and with external systems. Docker provides several network types to suit different use cases.

---

## Docker Network Architecture Overview

```mermaid
flowchart TB
    subgraph HOST["Docker Host"]
        subgraph CONTAINERS["Containers"]
            C1["Container A"]
            C2["Container B"]
            C3["Container C"]
        end
        
        subgraph NETWORKS["Networks"]
            N1["Bridge Network"]
            N2["Overlay Network"]
            N3["Macvlan Network"]
        end
        
        DOCKERD["Docker Daemon"]
    end
    
    C1 --> N1
    C2 --> N1
    C3 --> N2
    
    N1 --> DOCKERD
    N2 --> DOCKERD
    N3 --> DOCKERD
    
    DOCKERD --> NIC["Physical NIC"]
    NIC --> INTERNET["Internet"]
```

---

## 1. Default Bridge Network

```mermaid
flowchart TB
    subgraph HOST["Docker Host"]
        subgraph BRIDGE["Default Bridge Network (docker0)"]
            C1["Container A<br/>IP: 172.17.0.2"]
            C2["Container B<br/>IP: 172.17.0.3"]
        end
        
        BRIDGE --> HOST_NIC["Host Network Interface<br/>IP: 192.168.1.10"]
    end
    
    HOST_NIC --> INTERNET["Internet"]
    
    C1 <-.->|"Can communicate via IP only<br/>NOT via container names"| C2
```

**Command:**
```bash
docker run --network=bridge my-container
```

**Key Points:**
- Default network mode
- Private network on host machine
- Containers communicate via IP addresses (NOT container names)
- Automatic IP assignment

---

## 2. Host Network

```mermaid
flowchart TB
    subgraph HOST["Docker Host"]
        subgraph HOST_STACK["Host Network Stack"]
            C1["Container A<br/>Shares Host IP"]
            C2["Container B<br/>Shares Host IP"]
        end
        
        HOST_STACK --> NIC["Network Interface<br/>IP: 192.168.1.10"]
    end
    
    NIC --> INTERNET["Internet"]
    
    C1 -.->|"No network isolation<br/>Direct host access"| C2
```

**Command:**
```bash
docker run --network=host my-container
```

**Key Points:**
- Shares host's network stack
- No port mapping required
- Best for performance-critical apps
- No network isolation

---

## 3. Overlay Network (Multi-Host)

```mermaid
flowchart TB
    subgraph HOST1["Docker Host 1 (192.168.1.10)"]
        subgraph OVERLAY1["Overlay Network"]
            C1["Container A<br/>IP: 10.0.0.2"]
        end
    end
    
    subgraph HOST2["Docker Host 2 (192.168.1.20)"]
        subgraph OVERLAY2["Overlay Network"]
            C2["Container B<br/>IP: 10.0.0.3"]
        end
    end
    
    subgraph SWARM["Docker Swarm Manager"]
        M["Swarm Manager"]
    end
    
    OVERLAY1 <-.->|"VXLAN Tunnel<br/>(Encrypted)"| OVERLAY2
    M --> OVERLAY1
    M --> OVERLAY2
    
    C1 <-.->|"Seamless Communication<br/>Same Network"| C2
```

**Commands:**
```bash
# Initialize Swarm mode first
docker swarm init

# Create overlay network
docker network create --driver=overlay my-overlay-network

# Deploy service with overlay network
docker service create --network=my-overlay-network my-service
```

**Key Points:**
- Connects containers across multiple hosts
- Uses VXLAN encapsulation
- Requires Docker Swarm mode
- Built-in encryption option

---

## 4. Macvlan Network

```mermaid
flowchart TB
    subgraph HOST["Docker Host"]
        subgraph MACVLAN["Macvlan Network"]
            C1["Container A<br/>MAC: aa:bb:cc:dd:ee:ff<br/>IP: 192.168.1.100"]
            C2["Container B<br/>MAC: 11:22:33:44:55:66<br/>IP: 192.168.1.101"]
        end
        
        NIC["Physical NIC<br/>MAC: 00:11:22:33:44:55<br/>IP: 192.168.1.50"]
        
        MACVLAN --> NIC
    end
    
    subgraph PHYSICAL["Physical Network"]
        SWITCH["Network Switch"]
        ROUTER["Router<br/>Gateway: 192.168.1.1"]
        SERVER["Other Server<br/>IP: 192.168.1.200"]
    end
    
    NIC --> SWITCH
    SWITCH --> ROUTER
    SWITCH --> SERVER
    
    C1 -.->|"Appears as Physical Device"| SERVER
```

**Command:**
```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my-macvlan-network

docker run --network=my-macvlan-network my-container
```

**Key Points:**
- Each container gets its own MAC address
- Containers appear as physical devices on network
- Direct Layer 2 network access
- Useful for legacy applications

---

## 5. Custom Bridge Network (Recommended)

```mermaid
flowchart TB
    subgraph HOST["Docker Host"]
        subgraph CUSTOM["Custom Bridge Network: my-network<br/>Subnet: 10.10.0.0/24"]
            WEB["Web Container<br/>Name: web-app<br/>IP: 10.10.0.2"]
            DB["Database Container<br/>Name: database<br/>IP: 10.10.0.3"]
            CACHE["Cache Container<br/>Name: redis<br/>IP: 10.10.0.4"]
        end
        
        CUSTOM --> HOST_NIC["Host NIC<br/>IP: 192.168.1.10"]
    end
    
    HOST_NIC --> INTERNET["Internet"]
    
    WEB <-.->|"DNS: 'database' resolves to 10.10.0.3"| DB
    WEB <-.->|"DNS: 'redis' resolves to 10.10.0.4"| CACHE
```

**Commands:**
```bash
# Create custom bridge network
docker network create my-network

# Run containers on the network
docker run -d --name web-app --network my-network nginx
docker run -d --name database --network my-network postgres
docker run -d --name redis --network my-network redis

# Now web-app can ping database by name!
```

**Key Points:**
- User-defined private network
- **Automatic DNS resolution between containers**
- Better isolation than default bridge
- Containers can reach each other by container name

---

## 6. None Network (Fully Isolated)

```mermaid
flowchart TB
    subgraph HOST["Docker Host"]
        subgraph ISOLATED["None Network Mode"]
            C1["Container A"]
            C2["No eth0 interface"]
            C3["No IP Address"]
            C4["No Network Access"]
            C5["Completely Isolated"]
        end
    end
    
    EXTERNAL["External World / Internet"]
    
    ISOLATED -.->|"NO Communication Possible"| EXTERNAL
    
    style ISOLATED fill:#f5f5f5,stroke:#999999,stroke-width:1px
    style C1 fill:#ffffff,stroke:#666666
    style C2 fill:#ffffff,stroke:#666666
    style C3 fill:#ffffff,stroke:#666666
    style C4 fill:#ffffff,stroke:#666666
    style C5 fill:#ffffff,stroke:#666666
    style EXTERNAL fill:#e8f4f8,stroke:#666666
```

**Command:**
```bash
docker run --network=none my-container
```

**Key Points:**
- Complete network isolation
- No network interfaces at all
- No incoming or outgoing traffic
- Maximum security for sensitive workloads
- Perfect for batch jobs or offline processing

---

## Network Selection Guide

```mermaid
flowchart TD
    START["Need Docker Network?"] --> Q1["Multiple Docker Hosts?"]
    
    Q1 -->|"Yes"| Q2["Need Direct Layer 2 Access?"]
    Q1 -->|"No"| Q3["Need Maximum Performance?"]
    
    Q2 -->|"Yes"| MACVLAN["Macvlan Network"]
    Q2 -->|"No"| OVERLAY["Overlay Network"]
    
    Q3 -->|"Yes"| HOST["Host Network"]
    Q3 -->|"No"| Q4["Need Container Name Resolution?"]
    
    Q4 -->|"Yes"| CUSTOM["Custom Bridge Network (Recommended)"]
    Q4 -->|"No"| Q5["Need Isolation Level?"]
    
    Q5 -->|"Complete"| NONE["None Network"]
    Q5 -->|"Moderate"| DEFAULT["Default Bridge Network"]
    
    MACVLAN --> DEPLOY["Deploy Containers"]
    OVERLAY --> DEPLOY
    HOST --> DEPLOY
    CUSTOM --> DEPLOY
    NONE --> DEPLOY
    DEFAULT --> DEPLOY
```

---

## Comparison Table

| Network Type | Use Case | Isolation | Multi-Host | DNS Resolution | Performance |
|--------------|----------|-----------|------------|----------------|-------------|
| **Default Bridge** | Basic single-host | Moderate | No | No | Medium |
| **Host Network** | Performance critical | None | No | N/A | High |
| **Overlay Network** | Multi-host services | High | Yes | Yes | Medium |
| **Macvlan Network** | Legacy/L2 access | Low | Yes | No | High |
| **Custom Bridge** | Production apps | Moderate | No | Yes | Medium |
| **None Network** | Security sensitive | Complete | No | N/A | N/A |

---

## Network Communication Flow Example

```mermaid
sequenceDiagram
    participant Web as Web Container<br/>(web-app)
    participant DNS as Custom Bridge DNS<br/>(my-network)
    participant DB as Database Container<br/>(database)
    
    Note over Web: Code tries to connect to<br/>'database:5432'
    
    Web->>DNS: DNS Query: "database"
    DNS->>DNS: Lookup in network
    DNS->>Web: Returns IP: 10.10.0.3
    
    Web->>DB: TCP Connection to 10.10.0.3:5432
    DB->>Web: Connection Established
    Web->>DB: Send Query
    DB->>Web: Return Results
    
    Note over Web,DB: Communication Successful!
```

---

## Network Management Commands

```bash
# List all networks
docker network ls

# Inspect a network (shows configuration and connected containers)
docker network inspect <network_name>

# Create custom bridge network
docker network create my-network

# Create network with specific subnet
docker network create --subnet=10.10.0.0/24 my-network

# Create overlay network (requires Swarm)
docker network create --driver=overlay my-overlay-network

# Create macvlan network
docker network create -d macvlan --subnet=192.168.1.0/24 --gateway=192.168.1.1 -o parent=eth0 my-macvlan

# Connect a running container to a network
docker network connect <network_name> <container_name>

# Disconnect a container from a network
docker network disconnect <network_name> <container_name>

# Remove a network
docker network rm <network_name>

# Remove all unused networks
docker network prune
```

---

## Best Practices

### Do's ✅
- Use **Custom Bridge Networks** for production applications
- Use **Overlay Networks** for multi-host setups
- Use descriptive network names (e.g., `prod-frontend`, `dev-database`)
- Regularly prune unused networks with `docker network prune`
- Use `docker network inspect` to debug connectivity issues

### Don'ts ❌
- Don't use Default Bridge for production (no DNS resolution)
- Don't use Host Network unless absolutely necessary
- Don't create too many unused networks
- Don't forget to cleanup after testing

---

## Common Scenarios

### Scenario 1: Web Application with Database
```mermaid
flowchart TB
    subgraph SCENARIO["Single Host - Web + Database"]
        WEB["Web Container<br/>(nginx on port 80)"]
        APP["App Container<br/>(Node.js/Python)"]
        DB["Database Container<br/>(PostgreSQL)"]
        NET["Custom Bridge Network<br/>(app-network)"]
        
        WEB --> NET
        APP --> NET
        DB --> NET
    end
```

### Scenario 2: Multi-Host Microservices
```mermaid
flowchart TB
    subgraph HOSTA["Production Host A"]
        SERVICE1["User Service"]
        OVERLAY1["Overlay Network"]
    end
    
    subgraph HOSTB["Production Host B"]
        SERVICE2["Payment Service"]
        OVERLAY2["Overlay Network"]
    end
    
    subgraph HOSTC["Production Host C"]
        SERVICE3["Notification Service"]
        OVERLAY3["Overlay Network"]
    end
    
    OVERLAY1 <--> OVERLAY2
    OVERLAY2 <--> OVERLAY3
    SERVICE1 --> OVERLAY1
    SERVICE2 --> OVERLAY2
    SERVICE3 --> OVERLAY3
```

---

## Troubleshooting Common Issues

| Issue | Solution |
|-------|----------|
| Containers can't ping each other by name | Use Custom Bridge Network instead of Default Bridge |
| Port already in use | Use different host port: `-p 8081:80` |
| Can't communicate between hosts | Use Overlay Network with Docker Swarm |
| Container has no internet | Check if network has `--internal` flag |
| Macvlan not working | Ensure parent interface name is correct (`eth0`, `ens33`, etc.) |

---

> **Important Note**: Default bridge network does NOT support automatic DNS resolution between containers. Always use **Custom Bridge Networks** for container name-based communication in production.

*Reference: Docker Networking Notes for DevOps Engineers*
