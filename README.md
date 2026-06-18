# High-Availability (HA) Multi-Tier Web Infrastructure PoC 
# Refer to /Verification/Full Architecture System.pdf

A production-grade, highly available enterprise infrastructure platform built entirely on VMware Workstation. [cite_start]This Proof of Concept (PoC) eliminates Single Points of Failure (SPOFs) across the application processing, session management, and database layers to achieve **99.9% uptime** resilience under automated fault conditions[cite: 2, 3, 36].

## 🏗️ System Architecture
<img width="1002" height="451" alt="image" src="https://github.com/user-attachments/assets/d62bd9ca-2225-48c6-8228-cd2992ae6716" />


### Infrastructure & Network Matrix
[cite_start]The entire platform operates over static IPv4 network interfaces on the `192.168.192.0/24` subnet[cite: 37, 51]. [cite_start]Node communication parameters are mapped out explicitly below:

| VM Host Name | Installed Service | IP Address | Infrastructure Role |
| :--- | :--- | :--- | :--- |
| **Nginx** | Nginx Reverse Proxy | `192.168.192.100` | [cite_start]Gateway Layer-7 Load Balancer via upstream distribution. |
| **WebApp-1** | Apache + PHP 8.3 | `192.168.192.145` | [cite_start]Stateless processing node (Primary Active)[cite: 7, 36, 37]. |
| **WebApp-2** | Apache + PHP 8.3 | `192.168.192.146` | [cite_start]Stateless processing node (Redundant Active)[cite: 8, 36, 37]. |
| **MySQL-Master**| MySQL 8.0 Engine | `192.168.192.140` | [cite_start]Primary transaction instance for all WRITE operations. |
| **MySQL-Slave** | MySQL 8.0 Engine | `192.168.192.141` | [cite_start]Asynchronous read-only data redundancy replica. |
| **Redis-DB** | Redis Key-Value Core| `192.168.192.142` | [cite_start]Centralized external session store for user persistence[cite: 10, 36, 37]. |
| **Storage-NFS** | NFS Kernel Daemon | `192.168.192.143` | [cite_start]Shared network filesystem for consistent uploads/media[cite: 18, 36, 37]. |

---

## 🛠️ Multi-Tier Architecture & Configurations

### 1. Gateway Load Balancing (Nginx)
[cite_start]The frontdoor Nginx proxy balances the incoming traffic utilizing the `least_conn` strategy to target nodes with fewer active connections[cite: 6, 55, 56]. [cite_start]It handles proxy header injections securely so that underlying backends remain fully aware of the client's original state[cite: 65]:
* [cite_start]Configured upstream proxy headers pass the real client visitor IPs (`X-Real-IP`, `X-Forwarded-For`) backwards to the web nodes[cite: 67, 68].
* Production blueprints are isolated in `configs/nginx/web_lb.conf`.

### 2. Stateless Web Tier (Apache + PHP 8.3)
[cite_start]To achieve seamless round-robin routing without introducing session loss or forcing users to log back in when toggled between nodes, the application layer is configured completely **stateless**[cite: 7, 8, 56]:
* [cite_start]**Decoupled Session Persistence:** Native local disk caching is disabled in PHP configuration profiles[cite: 80]. [cite_start]Instead, session handling parameters target the standalone Redis engine instance directly (`tcp://192.168.192.142:6379`) via TCP sockets[cite: 81].
* [cite_start]**Proxy Header Awareness:** Backends deploy `mod_remoteip` to overwrite the client tracking parameters with the headers forwarded by the load balancer at `.100`[cite: 90, 91].
* [cite_start]**Distributed File Consistency:** Permanent boot-time `/etc/fstab` bindings mount shared asset locations directly from the central storage appliance over an NFSv4 storage mount[cite: 103, 104].

### 3. Database Replication Cluster (MySQL 8.0)
[cite_start]Data resiliency relies on stable asynchronous streaming pipelines across separate nodes[cite: 12]:
* [cite_start]**Master Node (`.140`):** Operates under global `server-id = 1` with binary tracking files actively generated (`mysql-bin.log`)[cite: 40, 42, 43].
* [cite_start]**Slave Node (`.141`):** Synchronizes logs using a specialized relay engine (`mysql-relay-bin.log`) running under matching `server-id = 2`[cite: 45, 47, 48].
* [cite_start]Active telemetry validates successful connection paths, indicating `Seconds_Behind_Master: 0` along with operational states confirming `Replica has read all relay log: waiting for more updates`[cite: 128, 140].

---

## 🧪 Operational Failover Testing & Verification

[cite_start]Failover simulations were executed manually in VMware Workstation to test structural robustness against sudden node deaths[cite: 190, 208, 234]:

### Condition 1: Nominal State Operations (All Nodes Online)
* [cite_start]Nginx balances client actions equally between `WebApp-1` and `WebApp-2`[cite: 55, 57, 58].
* [cite_start]Primary application transactions stream into the active MySQL Master database without disruption[cite: 196, 197].
* [cite_start]Active Redis connections maintain a `Ping Response: 1` tracking user token allocation cleanly[cite: 202, 203].

### Condition 2: Primary Node Failure (Web App 1 Offline)
* [cite_start]**Action Executed:** The `WebApp-1` virtual machine was abruptly suspended/paused inside VMware[cite: 208, 233].
* [cite_start]**Result observed:** Nginx health checks instantly flagged backend timeout boundaries, dropping the unresponsive node and routing 100% of incoming client traffic to `WebApp-2`[cite: 58].
* [cite_start]**Session Integrity:** Users experienced zero dropouts or data degradation because active session parameters remained securely available within the remote Redis caching backend[cite: 222].

### Condition 3: Total Failover Verification
* [cite_start]The platform cleanly served application requests through the remaining running node[cite: 234]. 
* [cite_start]Even with `WebApp-1` completely offline, data integrity remained intact, and database queries executed successfully over the persistent network tier[cite: 255, 256].

---

## 🔮 Production Roadmap Enhancements
To scale this laboratory PoC environment up into an enterprise cloud implementation, the following architectural upgrades would be introduced:
1. **Load Balancer Clustering:** Introduce a secondary active/passive Nginx node managed by **Keepalived (VRRP)** sharing a Virtual IP (VIP) to remove the load balancer as a single point of failure.
2. **Distributed Storage Clustering:** Replace the single NFS storage appliance with a self-healing distributed filesystem such as **GlusterFS** or a replicated **Ceph Storage** cluster.
3. **Automated Database Failover Orchestration:** Integrate **Orchestrator** or **ProxySQL** proxy nodes to automate topology tracking, transforming manual recovery tasks into scripted, zero-touch failovers if the primary database engine suffers an outage.
