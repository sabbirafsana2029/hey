# 🌐 Linux Network Namespace Simulation

A project that demonstrates how to create isolated virtual network environments on a single Linux host using **network namespaces**, **virtual ethernet (veth) pairs**, and **Linux bridges** — simulating real inter-network routing without any physical hardware or VMs.

---

## 📌 Objective

Create a network simulation with two separate networks connected via a router using Linux network namespaces and bridges.

---

## 🗺️ Network Topology

```
┌─────────────────────────────────────────────────────────────────────┐
│                         HOST MACHINE                                │
│                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌─────────────┐    ┌──────────┐  │
│  │   ns1    │    │   br0    │    │  router-ns  │    │   br1    │  │
│  │10.0.1.2  │◄──►│ (bridge) │◄──►│  10.0.1.1   │◄──►│ (bridge) │  │
│  │  /24     │    │          │    │  10.0.2.1   │    │          │  │
│  └──────────┘    └──────────┘    └─────────────┘    └──────────┘  │
│                                                             │       │
│                                                    ┌──────────┐    │
│                                                    │   ns2    │    │
│                                                    │10.0.2.2  │    │
│                                                    │  /24     │    │
│                                                    └──────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

**Simple view:**
```
ns1 ── br0 ── router-ns ── br1 ── ns2
```

---

## 🗂️ Project Structure

```
project/
├── README.md       ← Documentation (this file)
├── setup.sh        ← Automated setup script
├── cleanup.sh      ← Teardown script
└── diagram.png     ← Network topology diagram
```

---

## 📋 IP Addressing Scheme

| Component   | Interface  | IP Address | Subnet | Role                  |
|-------------|------------|------------|--------|-----------------------|
| ns1         | veth-ns1   | 10.0.1.2   | /24    | Host on Network 1     |
| router-ns   | veth-r1    | 10.0.1.1   | /24    | Gateway for Network 1 |
| router-ns   | veth-r2    | 10.0.2.1   | /24    | Gateway for Network 2 |
| ns2         | veth-ns2   | 10.0.2.2   | /24    | Host on Network 2     |

- **Network 1:** `10.0.1.0/24`
- **Network 2:** `10.0.2.0/24`
- **Default gateway for ns1:** `10.0.1.1`
- **Default gateway for ns2:** `10.0.2.1`

---

## ⚙️ Components Explained

### 🔷 Bridges (br0, br1)
Linux bridges act like virtual switches. `br0` connects ns1 and the router's first interface. `br1` connects ns2 and the router's second interface.

### 🔷 Namespaces (ns1, ns2, router-ns)
Each namespace is a fully isolated network stack with its own interfaces, routing table, and firewall rules — completely separate from the host and each other.

### 🔷 veth Pairs
A veth pair is a virtual cable with two ends. Packets entering one end come out the other.

| veth Pair              | End 1       | End 2      | Purpose                     |
|------------------------|-------------|------------|-----------------------------|
| veth-ns1 ↔ veth-br0   | ns1         | br0        | Connect ns1 to Network 1    |
| veth-r1  ↔ veth-br0-r | router-ns   | br0        | Connect router to Network 1 |
| veth-ns2 ↔ veth-br1   | ns2         | br1        | Connect ns2 to Network 2    |
| veth-r2  ↔ veth-br1-r | router-ns   | br1        | Connect router to Network 2 |

---

## 🚀 Setup & Usage

### Requirements
- Linux OS (Ubuntu 20.04+ / Debian 11+)
- `iproute2` package (`ip` command)
- Root / sudo privileges

### Run with Script (Recommended)

```bash
# Clone the repository
git clone https://github.com/your-username/your-repo-name.git
cd your-repo-name

# Give execute permission
chmod +x setup.sh cleanup.sh

# Run setup
sudo ./setup.sh

# Run cleanup after testing
sudo ./cleanup.sh
```

---

## 🔧 All Commands (Step by Step)

### Step 0 — Clean Start
```bash
sudo ip netns del ns1 2>/dev/null
sudo ip netns del ns2 2>/dev/null
sudo ip netns del router-ns 2>/dev/null
sudo ip link del br0 2>/dev/null
sudo ip link del br1 2>/dev/null
```

### Step 1 — Create Bridges
```bash
sudo ip link add br0 type bridge
sudo ip link add br1 type bridge
sudo ip link set br0 up
sudo ip link set br1 up
```

### Step 2 — Create Namespaces
```bash
sudo ip netns add ns1
sudo ip netns add ns2
sudo ip netns add router-ns
ip netns
```

### Step 3 — Create veth Pairs
```bash
sudo ip link add veth-ns1 type veth peer name veth-br0
sudo ip link add veth-ns2 type veth peer name veth-br1
sudo ip link add veth-r1 type veth peer name veth-br0-r
sudo ip link add veth-r2 type veth peer name veth-br1-r
```

### Step 4 — Move Interfaces into Namespaces
```bash
sudo ip link set veth-ns1 netns ns1
sudo ip link set veth-ns2 netns ns2
sudo ip link set veth-r1 netns router-ns
sudo ip link set veth-r2 netns router-ns
```

### Step 5 — Attach to Bridges and Bring Up
```bash
sudo ip link set veth-br0 master br0
sudo ip link set veth-br0-r master br0
sudo ip link set veth-br1 master br1
sudo ip link set veth-br1-r master br1
sudo ip link set veth-br0 up
sudo ip link set veth-br0-r up
sudo ip link set veth-br1 up
sudo ip link set veth-br1-r up
```

### Step 6 — Assign IP Addresses

**ns1:**
```bash
sudo ip netns exec ns1 ip addr add 10.0.1.2/24 dev veth-ns1
sudo ip netns exec ns1 ip link set veth-ns1 up
sudo ip netns exec ns1 ip link set lo up
```

**ns2:**
```bash
sudo ip netns exec ns2 ip addr add 10.0.2.2/24 dev veth-ns2
sudo ip netns exec ns2 ip link set veth-ns2 up
sudo ip netns exec ns2 ip link set lo up
```

**router-ns:**
```bash
sudo ip netns exec router-ns ip addr add 10.0.1.1/24 dev veth-r1
sudo ip netns exec router-ns ip addr add 10.0.2.1/24 dev veth-r2
sudo ip netns exec router-ns ip link set veth-r1 up
sudo ip netns exec router-ns ip link set veth-r2 up
sudo ip netns exec router-ns ip link set lo up
```

### Step 7 — Add Default Routes
```bash
sudo ip netns exec ns1 ip route add default via 10.0.1.1
sudo ip netns exec ns2 ip route add default via 10.0.2.1
```

### Step 8 — Enable IP Forwarding in Router
```bash
sudo ip netns exec router-ns sysctl -w net.ipv4.ip_forward=1
```

### Step 9 — Test Connectivity
```bash
sudo ip netns exec ns1 ping 10.0.2.2
sudo ip netns exec ns2 ping 10.0.1.2
```

### Cleanup
```bash
sudo ip netns del ns1
sudo ip netns del ns2
sudo ip netns del router-ns
sudo ip link del br0
sudo ip link del br1
```

---

## 🔀 Routing Configuration Explained

### Why do ns1 and ns2 need default routes?
Without a default route, ns1 only knows its own subnet `10.0.1.0/24`. When it tries to reach `10.0.2.2`, it has no idea where to send the packet. The default route `via 10.0.1.1` tells ns1 to send all unknown traffic to the router.

### Why does the router need IP forwarding?
By default, Linux drops packets destined for a different interface. `net.ipv4.ip_forward=1` tells the kernel to **forward** such packets — this is what makes router-ns act as a real router.

### Does the router need static routes?
No. The router-ns automatically knows both subnets because it has an interface directly in each one. Linux adds these **connected routes** automatically when IP addresses are assigned.

---

## ✅ Testing Results

### Expected ping output (ns1 → ns2)
```
PING 10.0.2.2 (10.0.2.2) 56(84) bytes of data.
64 bytes from 10.0.2.2: icmp_seq=1 ttl=63 time=0.XXX ms
64 bytes from 10.0.2.2: icmp_seq=2 ttl=63 time=0.XXX ms
64 bytes from 10.0.2.2: icmp_seq=3 ttl=63 time=0.XXX ms
```

> `ttl=63` (not 64) confirms the packet passed through router-ns — each hop decrements TTL by 1.

### Verify routing tables
```bash
sudo ip netns exec ns1 ip route
sudo ip netns exec ns2 ip route
sudo ip netns exec router-ns ip route
```

### Verify IP addresses
```bash
sudo ip netns exec ns1 ip addr
sudo ip netns exec ns2 ip addr
sudo ip netns exec router-ns ip addr
```

---

## 🛠️ Technical Requirements

- All commands must be executed with root privileges
- Solution works on a standard Linux system with kernel 3.8+
- `iproute2` must be installed

---

## 👤 Author

> Replace this section with your name, roll number, and institution.

- **Name:** Your Name
- **Roll No:** Your Roll Number
- **Institution:** Your College / University
