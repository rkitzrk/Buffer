# Networking Interview Questions - SDE Interview Prep

---

## TOP 10 MOST IMPORTANT NETWORKING QUESTIONS

---

### Q1: Explain the OSI model, its seven layers, the devices used at each layer, and how it differs from the TCP/IP model.

**Answer:** The OSI (Open Systems Interconnection) model is a 7-layer reference model that standardizes network communication between devices. Each layer performs a specific function, and different networking devices operate at different layers. The TCP/IP model is a 4-layer practical model that is widely used for Internet communication.

**Polished Answer:** The OSI model is a conceptual framework that standardizes network communication into 7 distinct layers:

| Layer | Function | Devices |
|-------|----------|---------|
| **Layer 7 - Application** | Provides network services to end-user applications | Gateway, Firewall |
| **Layer 6 - Presentation** | Translates, encrypts, compresses data | Gateway, Firewall |
| **Layer 5 - Session** | Establishes, manages, terminates sessions | Gateway, Firewall |
| **Layer 4 - Transport** | End-to-end delivery, segmentation, reassembly | Gateway, Firewall |
| **Layer 3 - Network** | Logical addressing, routing, path determination | Router, Layer 3 Switch |
| **Layer 2 - Data Link** | Physical addressing (MAC), framing, error detection | Switch, Bridge |
| **Layer 1 - Physical** | Transmits raw bits over medium | Hub, Repeater, Cables |

**OSI vs TCP/IP:** OSI is a theoretical reference model with 7 layers, while TCP/IP is a practical implementation with 4 layers (Application, Transport, Internet, Network Access). OSI has strict layer boundaries; TCP/IP is more flexible. OSI was developed by ISO; TCP/IP was developed by the US Department of Defense.

**TL;DR:** OSI = 7-layer reference model (theoretical). TCP/IP = 4-layer practical model. Each OSI layer has specific devices and functions.

**Keywords:** OSI layers, TCP/IP model, encapsulation, decapsulation, Layer 1-7, networking devices

---

### Q2: Explain the TCP three-way handshake and why TCP uses three handshakes instead of two.

**Answer:** The TCP three-way handshake is the process used to establish a reliable connection between a client and a server before data transmission begins. It ensures that both devices are ready to communicate and can exchange data reliably.

**Polished Answer:** The TCP three-way handshake establishes a reliable connection in three steps:

**Step 1 (SYN):** Client sends a SYN packet with an initial sequence number (ISN) to request a connection.

**Step 2 (SYN-ACK):** Server responds with a SYN-ACK packet, acknowledging the client's SYN and sending its own sequence number.

**Step 3 (ACK):** Client sends a final ACK confirming receipt of the server's SYN-ACK.

**Why three handshakes, not two?**
- The third ACK confirms the client received the server's response
- It ensures both sides can send AND receive data (bidirectional readiness)
- It prevents duplicate or delayed connection requests from creating invalid/half-open connections
- With only two steps, the server wouldn't know if the client actually received its response

**TL;DR:** SYN → SYN-ACK → ACK. Three steps ensure bidirectional communication readiness and prevent stale connection requests.

**Keywords:** TCP handshake, SYN, SYN-ACK, ACK, sequence numbers, connection establishment, half-open connections

---

### Q3: What is the difference between TCP and UDP?

**Answer:** TCP is a connection-oriented protocol providing reliable communication with error checking, acknowledgments, and retransmission. UDP is a connectionless protocol that sends data without establishing a connection, providing no delivery guarantees.

**Polished Answer:**

| Feature | TCP | UDP |
|---------|-----|-----|
| **Connection** | Connection-oriented (handshake required) | Connectionless (no handshake) |
| **Reliability** | Reliable - guaranteed delivery with ACKs | Unreliable - no delivery guarantee |
| **Ordering** | Maintains packet order | No ordering guarantee |
| **Error Checking** | Full error detection and recovery | Basic checksum only |
| **Speed** | Slower due to overhead | Faster, minimal overhead |
| **Retransmission** | Yes, retransmits lost packets | No retransmission |
| **Header Size** | 20-60 bytes | 8 bytes |
| **Use Cases** | Web browsing (HTTP/HTTPS), Email (SMTP), File Transfer (FTP), SSH | Streaming, VoIP, Online Gaming, DNS queries |

**Why UDP for real-time apps?** Low latency matters more than reliability. Small packet losses are acceptable in streaming/gaming, but delays from TCP retransmission would degrade the experience.

**TL;DR:** TCP = reliable but slower (connection + ACKs). UDP = fast but unreliable (no connection, no guarantees). Choose based on application needs.

**Keywords:** TCP vs UDP, connection-oriented, connectionless, reliability, latency, streaming, retransmission

---

### Q4: What is DNS and how does it work? What happens when you enter a URL in a browser?

**Answer:** DNS (Domain Name System) is a decentralized and hierarchical naming system that translates domain names to IP addresses. When a URL is entered, the browser resolves the domain using DNS, establishes a TCP/TLS connection, sends an HTTP request, and renders the webpage.

**Polished Answer:**

**DNS (Domain Name System):**
- Acts as the "phonebook of the Internet"
- Translates human-readable domain names (e.g., google.com) to IP addresses (e.g., 172.217.166.36)
- Uses port 53 by default
- Hierarchical structure: Root DNS servers → TLD servers (.com, .org) → Authoritative DNS servers

**What happens when you enter a URL:**

1. **DNS Resolution:** Browser checks its cache → OS cache → Router cache → ISP DNS server → Root/TLD/Authoritative servers to resolve the domain to an IP address

2. **TCP Connection:** Browser establishes a TCP connection with the server using a three-way handshake (SYN, SYN-ACK, ACK)

3. **TLS Handshake (if HTTPS):** Client and server exchange certificates, verify identity, and negotiate encryption keys

4. **HTTP Request:** Browser sends an HTTP GET request for the webpage

5. **Server Processing:** Server processes the request and sends back an HTTP response with the webpage content

6. **Rendering:** Browser receives, parses HTML/CSS/JavaScript, and displays the webpage

**TL;DR:** Enter URL → DNS resolves domain to IP → TCP + TLS connection → HTTP request → Server responds → Browser renders page.

**Keywords:** DNS resolution, URL loading, TCP handshake, TLS handshake, HTTP request, DNS hierarchy

---

### Q5: What is NAT (Network Address Translation) and why is it used?

**Answer:** Network Address Translation (NAT) is a technique that converts private IP addresses into public IP addresses, allowing multiple devices to share a single public IP. It helps conserve IPv4 addresses and improves security by hiding internal network details.

**Polished Answer:**

**What is NAT?**
NAT is a router technique that translates private IP addresses to public IP addresses (and vice versa) for Internet communication.

**How it works:**
1. Device with private IP (e.g., 192.168.1.10) sends a request to a website
2. Router replaces the private IP with its own public IP
3. Router maintains a NAT table mapping internal IP:port to external IP:port
4. Response arrives at the router, which forwards it to the correct internal device

**Types of NAT:**
- **Static NAT:** Fixed one-to-one mapping between private and public IP (used for servers)
- **Dynamic NAT:** Uses a pool of public IPs assigned on-demand
- **PAT (Port Address Translation / NAT Overload):** Multiple devices share one public IP using different port numbers (most common)

**Why NAT is used:**
- **IPv4 Address Conservation:** Multiple devices share a single public IP
- **Security:** Internal IPs are hidden from external networks
- **Network Flexibility:** Internal network can use private addressing independently

**Limitations:**
- Breaks end-to-end connectivity (external hosts can't initiate connections without port forwarding)
- Complicates peer-to-peer, VoIP, online gaming
- Adds processing overhead

**TL;DR:** NAT translates private IPs to public IPs so multiple devices can share one public IP. Conserves IPv4 addresses and hides internal network.

**Keywords:** NAT, IP address translation, PAT, private IP, public IP, IPv4 conservation, port forwarding

---

### Q6: What is a VPN (Virtual Private Network) and what are its types?

**Answer:** A VPN creates a secure and encrypted connection over the Internet, allowing users to safely access a private network. It protects data during transmission and enables secure communication between users or networks.

**Polished Answer:**

**What is a VPN?**
A VPN creates an encrypted "tunnel" through the public Internet, allowing secure communication as if devices were on a private network. It provides confidentiality, integrity, and secure remote access.

**Types of VPN:**

**1. Remote Access VPN:**
- Connects individual users to a private network over the Internet
- Commonly used by employees working from home or traveling
- Requires VPN client software on the user's device

**2. Site-to-Site VPN:**
- Connects two or more entire networks over the Internet
- Two subtypes:
  - **Intranet VPN:** Connects branches of the same organization
  - **Extranet VPN:** Connects an organization with partners or customers

**Advantages:**
- Secure, encrypted communication over public networks
- Cost-effective compared to dedicated WAN connections
- Enables secure remote work
- Protects data from interception and eavesdropping
- Disguises online identity and location

**TL;DR:** VPN = encrypted tunnel over the Internet. Types: Remote Access (user to network) and Site-to-Site (network to network).

**Keywords:** VPN, Virtual Private Network, encryption, remote access, site-to-site, intranet VPN, extranet VPN, tunnel

---

### Q7: Explain the TCP Sliding Window mechanism. How does it improve throughput?

**Answer:** The TCP Sliding Window is a flow control mechanism that allows multiple packets to be sent before receiving acknowledgments. As acknowledgments arrive, the window moves forward, allowing continuous data transmission.

**Polished Answer:**

**What is TCP Sliding Window?**
A flow control mechanism that allows the sender to transmit multiple segments before requiring an acknowledgment, rather than waiting for each ACK individually.

**How it works:**
1. Sender maintains a "window" of unacknowledged packets it can send
2. Window size is determined by min(congestion window, receive window)
3. As ACKs arrive, the window "slides" forward, allowing new packets to be sent
4. This creates a continuous pipeline of data rather than stop-and-wait operation

**Example:**
- If window size is 5, sender transmits 5 packets without waiting
- When ACK for packet 1 arrives, window slides, and packet 6 can be sent
- This continuous flow maximizes bandwidth utilization

**Benefits:**
- **Improved Throughput:** Multiple packets in flight simultaneously
- **Better Bandwidth Utilization:** Reduced idle time waiting for ACKs
- **Flow Control:** Receiver's buffer capacity respected (receive window)

**Related concepts:**
- **Bandwidth-Delay Product (BDP):** Amount of data that can be in transit = Bandwidth × RTT
- **Pipelining:** Sending multiple packets without waiting for ACKs

**TL;DR:** Sliding window = send multiple packets before waiting for ACKs. Window slides as ACKs arrive. Maximizes throughput.

**Keywords:** TCP sliding window, flow control, pipelining, bandwidth utilization, throughput, ACK, congestion window

---

### Q8: What is the difference between a hub, switch, and router?

**Answer:** A hub broadcasts data to all connected devices (Physical layer). A switch forwards data only to the destination device using MAC addresses (Data Link layer). A router connects different networks and forwards packets using IP addresses (Network layer).

**Polished Answer:**

| Device | OSI Layer | Addressing | Function | Collision Domain |
|--------|-----------|------------|----------|------------------|
| **Hub** | Layer 1 (Physical) | None (broadcasts to all) | Broadcasts all traffic to all ports | Single collision domain |
| **Switch** | Layer 2 (Data Link) | MAC addresses | Forwards frames only to destination port | Separate collision domain per port |
| **Router** | Layer 3 (Network) | IP addresses | Routes packets between different networks | Separates broadcast domains |

**Key differences:**

**Hub:**
- Dumb device - broadcasts everything
- All connected devices share bandwidth
- Creates one large collision domain
- Prone to collisions and congestion

**Switch:**
- Maintains MAC address table (learns from source MACs)
- Forwards frames only to intended destination port
- Creates separate collision domain per port
- Supports VLANs, full-duplex communication

**Router:**
- Connects different networks (e.g., home network to Internet)
- Uses routing tables and IP addresses
- Separates broadcast domains
- Performs NAT, firewall functions, path selection

**Modern note:** Layer 3 switches combine switching (hardware speed) with routing capabilities.

**TL;DR:** Hub = broadcast to all (L1). Switch = forward by MAC (L2). Router = route by IP (L3).

**Keywords:** Hub, switch, router, collision domain, broadcast domain, MAC address, IP address, OSI layers

---

### Q9: What is a firewall and how does it work?

**Answer:** A firewall is a security system that monitors and controls incoming and outgoing network traffic based on predefined rules. It blocks unauthorized access and allows legitimate traffic.

**Polished Answer:**

**What is a Firewall?**
A firewall is a network security device (hardware or software) that acts as a barrier between trusted and untrusted networks, filtering traffic based on security rules.

**How it works:**
1. **Traffic Inspection:** Examines all incoming and outgoing packets
2. **Rule Evaluation:** Compares traffic against configured security policies
3. **Action:** Allows, blocks, or logs traffic based on rule match

**Types of Firewalls:**

**1. Packet Filtering Firewall:**
- Examines packet headers (source/destination IP, port, protocol)
- Fast but limited - doesn't inspect content
- Operates at Network/Transport layer

**2. Stateful Inspection Firewall:**
- Tracks active connections in a state table
- Makes decisions based on connection state
- More secure than packet filtering
- Operates at Network/Transport layer

**3. Application Layer Firewall (Proxy Firewall):**
- Inspects application-level data
- Can block specific content, URLs, malware
- Deepest inspection but slowest
- Operates at Application layer

**4. Next-Generation Firewall (NGFW):**
- Combines stateful inspection with application awareness
- Includes IPS, deep packet inspection, threat intelligence

**TL;DR:** Firewall = security barrier filtering traffic based on rules. Types: Packet filtering, Stateful, Application/Proxy, NGFW.

**Keywords:** Firewall, security, packet filtering, stateful inspection, access control, network security

---

### Q10: What is a subnet mask and what is CIDR notation?

**Answer:** Subnetting divides a network into smaller parts. The subnet mask tells which part of an IP address is the network and which part is for hosts. CIDR notation is a shorter way to represent the subnet mask.

**Polished Answer:**

**Subnet Mask:**
A 32-bit value that separates an IP address into network and host portions. It uses 1s for network bits and 0s for host bits.

**Common Subnet Masks:**

| CIDR | Subnet Mask | Total Addresses | Usable Hosts |
|------|-------------|-----------------|--------------|
| /8 | 255.0.0.0 | 16,777,216 | 16,777,214 |
| /16 | 255.255.0.0 | 65,536 | 65,534 |
| /24 | 255.255.255.0 | 256 | 254 |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /32 | 255.255.255.255 | 1 | 1 (single host) |

**Example: 192.168.1.0/24**
- Network portion: 192.168.1 (first 24 bits)
- Host portion: Last 8 bits (0-255)
- Network address: 192.168.1.0
- Broadcast address: 192.168.1.255
- Usable IPs: 192.168.1.1 to 192.168.1.254 (254 hosts)

**Why subnet?**
- Reduces broadcast traffic
- Improves security through network isolation
- Efficient IP address usage
- Logical organization of network

**CIDR (Classless Inter-Domain Routing):**
- Replaces classful addressing (Class A, B, C)
- Allows variable-length subnet masks (VLSM)
- Enables route aggregation/summarization

**TL;DR:** Subnet mask divides IP into network + host. CIDR (/24) is shorthand notation. Subnetting improves efficiency and security.

**Keywords:** Subnet mask, CIDR, network address, broadcast address, IP addressing, VLSM, subnetting

---

## TOP 11-25 MOST IMPORTANT NETWORKING QUESTIONS

---

### Q11: What is the TCP connection termination process and the TIME_WAIT state?

**Answer:** TCP connection termination uses a four-way handshake because each direction of communication is closed independently. The client and server exchange FIN and ACK packets to close the connection gracefully.

**Polished Answer:**

**TCP Connection Termination (Four-Way Handshake):**

**Step 1 (FIN):** Client sends FIN to server, indicating it has no more data to send.

**Step 2 (ACK):** Server acknowledges the FIN, but may still have data to send.

**Step 3 (FIN):** Server sends its own FIN when it's done sending data.

**Step 4 (ACK):** Client sends final ACK and enters TIME_WAIT state.

**TIME_WAIT State:**
- The client (initiator of closure) waits for 2×MSL (Maximum Segment Lifetime, typically 30-60 seconds)
- Allows delayed packets from the connection to expire
- Provides time for the final ACK to be retransmitted if lost

**What if final ACK is lost?**
- Server retransmits FIN
- Client, still in TIME_WAIT, sends ACK again
- Both sides then close cleanly

**Why four steps, not three?**
- TCP is full-duplex; each direction closes independently
- Server may have pending data after receiving client's FIN

**TL;DR:** FIN → ACK → FIN → ACK. TIME_WAIT prevents stale packets and handles lost ACKs.

**Keywords:** TCP termination, four-way handshake, TIME_WAIT, FIN, ACK, MSL, graceful shutdown

---

### Q12: What is ARP (Address Resolution Protocol) and how does it work?

**Answer:** ARP is used to map an IP address to a MAC address within a local network. A device sends an ARP broadcast asking "Who has this IP address?" and the destination device replies with its MAC address.

**Polished Answer:**

**What is ARP?**
ARP resolves IP addresses to MAC addresses on a local network segment. It operates at the boundary between Layer 2 (Data Link) and Layer 3 (Network).

**How ARP works:**

1. **Check ARP Cache:** Device first checks if it already has the IP-to-MAC mapping

2. **ARP Request (Broadcast):** If no mapping exists, device sends a broadcast frame: "Who has IP 192.168.1.5? Tell 192.168.1.10"

3. **ARP Reply (Unicast):** The device with that IP responds directly with its MAC address

4. **Cache Update:** Sender stores the mapping in its ARP cache for future communication

**Example:**
- PC A (192.168.1.10) wants to send data to PC B (192.168.1.5)
- PC A broadcasts ARP request to all devices
- PC B responds with its MAC address
- PC A now has the MAC address to build the Ethernet frame

**ARP Cache:**
- Temporary storage of IP-to-MAC mappings
- Entries expire after a timeout (typically 2-20 minutes)
- Reduces unnecessary ARP broadcasts

**ARP in IPv6:** Replaced by Neighbor Discovery Protocol (NDP) using ICMPv6.

**TL;DR:** ARP maps IP → MAC on local networks. Broadcast request → unicast reply → cache mapping.

**Keywords:** ARP, MAC address, IP address, ARP cache, broadcast, Address Resolution Protocol, Layer 2

---

### Q13: What is DHCP and what is the DORA process?

**Answer:** DHCP (Dynamic Host Configuration Protocol) automatically assigns IP addresses and other network settings to devices. The address assignment process follows DORA: Discover, Offer, Request, Acknowledge.

**Polished Answer:**

**What is DHCP?**
DHCP automatically configures devices with IP addresses, subnet masks, default gateways, DNS servers, and other network parameters without manual configuration.

**DORA Process:**

**D - Discover:**
- Client broadcasts a DHCP Discover message to find DHCP servers
- Sent to 255.255.255.255 (broadcast)

**O - Offer:**
- DHCP server responds with a DHCP Offer containing an available IP address
- Includes proposed lease duration and configuration parameters

**R - Request:**
- Client broadcasts a DHCP Request for the offered IP address
- Broadcast (not unicast) because multiple servers may have responded

**A - Acknowledge:**
- Server confirms with DHCP Acknowledgment (ACK)
- Client now has a valid IP address configuration

**DHCP Lease:**
- IP addresses are leased for a fixed period (typically 24 hours)
- Clients renew leases at 50% of lease time
- Releases IP if not renewed

**APIPA (Automatic Private IP Addressing):**
- If DHCP server is unreachable, device assigns itself 169.254.x.x
- Enables local network communication only (no Internet)
- Indicates DHCP configuration issue

**TL;DR:** DHCP = automatic IP assignment. DORA = Discover, Offer, Request, Acknowledge.

**Keywords:** DHCP, DORA, IP address assignment, lease, APIPA, 169.254.x.x, automatic configuration

---

### Q14: What is the difference between MAC address and IP address?

**Answer:** A MAC address is a hardware address assigned to a network interface card, while an IP address is a logical address used for communication across networks.

**Polished Answer:**

| Feature | MAC Address | IP Address |
|---------|-------------|------------|
| **Full Form** | Media Access Control | Internet Protocol |
| **OSI Layer** | Layer 2 (Data Link) | Layer 3 (Network) |
| **Type** | Physical/Hardware address | Logical address |
| **Assigned By** | Manufacturer (burned into NIC) | Network administrator or DHCP |
| **Format** | 48-bit hexadecimal (e.g., 00:1A:2B:3C:4D:5E) | 32-bit dotted decimal IPv4 (e.g., 192.168.1.1) |
| **Permanence** | Usually fixed (can be spoofed) | Can change (static or dynamic) |
| **Scope** | Local network only | Global routing possible |
| **Function** | Identifies device on local network | Identifies device location in network |
| **Address Space** | Flat (no hierarchy) | Hierarchical (network + host) |

**Analogy:**
- MAC address = Physical mailing address of a building (fixed)
- IP address = Current location/room number (can change)

**Key points:**
- MAC addresses operate within a local network segment
- IP addresses enable routing across networks
- Routers use IP addresses; switches use MAC addresses
- ARP bridges the gap between them

**TL;DR:** MAC = physical/hardware address (L2, local). IP = logical address (L3, routable).

**Keywords:** MAC address, IP address, physical address, logical address, OSI layers, network interface

---

### Q15: What are the different types of computer networks?

**Answer:** Computer networks are classified based on the area they cover: PAN, LAN, MAN, WAN, and GAN.

**Polished Answer:**

| Network Type | Coverage Area | Examples | Characteristics |
|--------------|---------------|----------|-----------------|
| **PAN** | Personal space (few meters) | Bluetooth, USB connections | Personal device connectivity |
| **LAN** | Single building/campus | Office network, home Wi-Fi | High speed, private ownership |
| **MAN** | City-wide | Cable TV network, city Wi-Fi | Connects multiple LANs |
| **WAN** | Country/continent | Internet, corporate WAN | Long distance, leased lines |
| **GAN** | Global | The Internet | Connects WANs worldwide |

**Details:**

**PAN (Personal Area Network):**
- Range: ~10 meters
- Examples: Bluetooth headphones, smartwatch to phone

**LAN (Local Area Network):**
- Range: Building/campus
- Examples: Office Ethernet, home Wi-Fi
- Technologies: Ethernet (IEEE 802.3), Wi-Fi (IEEE 802.11)

**MAN (Metropolitan Area Network):**
- Range: City
- Examples: ISP infrastructure, cable TV

**WAN (Wide Area Network):**
- Range: Cross-country/continental
- Technologies: MPLS, leased lines, satellite

**GAN (Global Area Network):**
- The Internet itself
- Connects networks globally via satellites and undersea cables

**TL;DR:** Networks classified by coverage: PAN (personal) → LAN (local) → MAN (city) → WAN (country) → GAN (global).

**Keywords:** PAN, LAN, MAN, WAN, GAN, network types, coverage area, network classification

---

### Q16: What is network topology and what are the different types?

**Answer:** Network topology is the physical or logical arrangement of devices in a computer network. It defines how devices are connected and how data is transmitted between them.

**Polished Answer:**

**What is Network Topology?**
The layout or arrangement of nodes (devices) and links (connections) in a network, defining the communication pattern.

**Types of Topologies:**

**1. Bus Topology:**
- All devices connected to a single shared cable (backbone)
- Simple and cheap for small networks
- Single point of failure (main cable)
- Examples: Old Ethernet (10BASE5, 10BASE2)

**2. Star Topology:**
- All devices connected to a central hub/switch
- Most common in modern LANs
- Failure of central device affects entire network
- Easy to troubleshoot and add/remove devices

**3. Ring Topology:**
- Devices connected in a closed circular loop
- Data travels in one direction from device to device
- Single node failure breaks the entire ring
- Examples: Token Ring networks (rarely used today)

**4. Mesh Topology:**
- Every device connected to one or more other devices
- Multiple communication paths (redundancy)
- Highly robust but expensive (many cables/ports)
- Used in critical infrastructure and WAN core networks

**5. Tree Topology:**
- Hierarchical combination of star networks on a bus backbone
- Scalable and organized
- Core failure affects entire subtree

**6. Hybrid Topology:**
- Combines multiple topology types
- Leverages strengths while mitigating weaknesses
- Common in large enterprise networks

**TL;DR:** Topology = network layout. Types: Bus, Star, Ring, Mesh, Tree, Hybrid. Star is most common in LANs.

**Keywords:** Network topology, bus topology, star topology, ring topology, mesh topology, tree topology, network layout

---

### Q17: What is a VLAN (Virtual LAN) and why is it used?

**Answer:** A VLAN is a way to divide a single physical network into multiple logical networks using a switch. It improves security, reduces broadcast traffic, and makes network management more flexible.

**Polished Answer:**

**What is a VLAN?**
A VLAN logically segments a physical network into isolated broadcast domains, regardless of physical location. Devices in the same VLAN communicate as if connected to the same switch.

**How VLANs work:**
- VLAN tagging (IEEE 802.1Q) adds a VLAN ID (1-4094) to Ethernet frames
- Switches forward traffic only within the same VLAN
- Broadcast traffic stays contained within a VLAN
- Trunk ports carry traffic for multiple VLANs between switches

**Why use VLANs?**

**1. Security:**
- Isolates sensitive departments (HR, Finance) from others
- Limits broadcast domain and potential attack surface

**2. Reduced Broadcast Traffic:**
- Broadcast storms contained within a VLAN
- Better network performance

**3. Logical Organization:**
- Group devices by function, not physical location
- Users can move but remain in the same VLAN

**4. Flexibility:**
- Easier network changes without physical rewiring

**Inter-VLAN Routing:**
- Devices in different VLANs cannot communicate directly
- Requires a router or Layer 3 switch

**Example:**
- VLAN 10: Engineering (ports 1-8)
- VLAN 20: HR (ports 9-16)
- VLAN 30: Finance (ports 17-24)
- All on the same physical switch but isolated from each other

**TL;DR:** VLAN = logical network segmentation on physical infrastructure. Improves security, reduces broadcasts, enables flexible organization.

**Keywords:** VLAN, IEEE 802.1Q, VLAN tagging, broadcast domain, inter-VLAN routing, logical segmentation

---

### Q18: What is HTTP and HTTPS? What is the difference?

**Answer:** HTTP is the HyperText Transfer Protocol for web communication. HTTPS is the secure version using TLS encryption. HTTP uses port 80; HTTPS uses port 443.

**Polished Answer:**

| Feature | HTTP | HTTPS |
|---------|------|-------|
| **Full Form** | HyperText Transfer Protocol | HyperText Transfer Protocol Secure |
| **Port** | 80 | 443 |
| **Encryption** | None (plain text) | TLS/SSL encryption |
| **Certificate** | Not required | Requires SSL/TLS certificate |
| **Data Security** | Vulnerable to interception | Encrypted end-to-end |
| **Speed** | Slightly faster (no encryption overhead) | Slightly slower (encryption) |
| **SEO** | Lower ranking | Better ranking (Google preference) |
| **URL Prefix** | http:// | https:// (padlock icon) |

**What is HTTP?**
- Stateless application layer protocol
- Request-response model between client and server
- Methods: GET, POST, PUT, DELETE, HEAD, OPTIONS
- Each request is independent (stateless)

**What is HTTPS?**
- HTTP over TLS (Transport Layer Security)
- Encrypts all communication between client and server
- Verifies server identity via digital certificates
- Prevents man-in-the-middle attacks, eavesdropping, and tampering

**TLS Handshake (simplified):**
1. Client sends supported TLS versions and ciphers
2. Server responds with chosen cipher and digital certificate
3. Client verifies certificate with Certificate Authority
4. Both agree on session key for symmetric encryption

**TL;DR:** HTTP = plain text (port 80). HTTPS = encrypted via TLS (port 443). Always use HTTPS.

**Keywords:** HTTP, HTTPS, TLS, SSL, encryption, port 80, port 443, web security, digital certificate

---

### Q19: Explain TCP congestion control mechanisms - Slow Start, Congestion Avoidance, and AIMD.

**Answer:** TCP congestion control prevents network congestion by adjusting the amount of data sent. It uses Slow Start, Congestion Avoidance, and AIMD (Additive Increase Multiplicative Decrease) to respond to congestion.

**Polished Answer:**

**TCP Congestion Control:**
Mechanisms that adjust the sending rate based on network conditions to prevent congestion collapse while maximizing throughput.

**1. Slow Start:**
- Starts with small congestion window (cwnd = 1 MSS)
- Exponentially increases cwnd (doubles each RTT)
- Continues until reaching slow start threshold (ssthresh)
- Goal: Quickly probe available bandwidth

**2. Congestion Avoidance:**
- Activated when cwnd ≥ ssthresh
- Linearly increases cwnd (adds 1 MSS per RTT)
- Conservative growth to avoid overshooting capacity

**3. AIMD (Additive Increase Multiplicative Decrease):**
- **Additive Increase:** Slowly add to cwnd during normal operation
- **Multiplicative Decrease:** Cut cwnd significantly on packet loss (typically halve it)
- Creates sawtooth pattern of cwnd over time

**Congestion Detection:**
- **Timeout (RTO):** Severe congestion → reset to Slow Start
- **Three Duplicate ACKs:** Moderate congestion → Fast Retransmit + Fast Recovery

**Fast Retransmit:**
- Three duplicate ACKs indicate packet loss
- Retransmit lost packet immediately without waiting for timeout

**Fast Recovery:**
- After fast retransmit, don't reset to Slow Start
- Reduce cwnd to half and continue in Congestion Avoidance

**TL;DR:** Slow Start (exponential) → Congestion Avoidance (linear) → Packet loss → reduce cwnd → repeat. AIMD = add slowly, cut sharply.

**Keywords:** TCP congestion control, Slow Start, Congestion Avoidance, AIMD, cwnd, ssthresh, Fast Retransmit

---

### Q20: Compare TCP variants - Tahoe, Reno, New Reno, and Cubic.

**Answer:** TCP Tahoe, Reno, New Reno, and Cubic are congestion control algorithms that differ in how they detect packet loss and recover from congestion.

**Polished Answer:**

| Algorithm | Packet Loss Detection | Recovery | Key Features |
|-----------|----------------------|----------|--------------|
| **Tahoe** | Timeout or 3 duplicate ACKs | Always restarts from Slow Start | Simple, conservative, slow recovery |
| **Reno** | Timeout or 3 duplicate ACKs | Fast Recovery on duplicate ACKs | Faster than Tahoe, handles single loss |
| **New Reno** | Timeout or 3 duplicate ACKs | Improved Fast Recovery | Handles multiple losses in one window |
| **Cubic** | Timeout or 3 duplicate ACKs | Cubic function growth | Optimized for high-bandwidth, long-distance |

**Tahoe:**
- Earliest TCP variant with congestion control
- Uses Slow Start and AIMD
- On packet loss (timeout or 3 dup ACKs): resets cwnd to 1 and restarts Slow Start
- Simple but inefficient for single packet losses

**Reno:**
- Introduces Fast Retransmit and Fast Recovery
- On 3 duplicate ACKs: retransmit packet, reduce cwnd by half, enter Fast Recovery
- On timeout: resets to Slow Start
- Better than Tahoe but struggles with multiple losses

**New Reno:**
- Improvement on Reno for multiple packet losses
- Stays in Fast Recovery until all outstanding packets are acknowledged
- Handles partial ACKs correctly

**Cubic:**
- Default in Linux
- Uses cubic function for window growth
- Growth independent of RTT (more fair in heterogeneous networks)
- Optimized for high-bandwidth-delay-product networks
- Quickly recovers to previous throughput after congestion

**TL;DR:** Tahoe = always slow start. Reno = fast recovery. New Reno = better multi-loss. Cubic = modern, high-speed optimized.

**Keywords:** TCP Tahoe, TCP Reno, TCP New Reno, Cubic TCP, congestion control algorithms, Fast Recovery

---

### Q21: What is the difference between symmetric and asymmetric encryption?

**Answer:** Symmetric encryption uses the same key for encryption and decryption, while asymmetric encryption uses two different keys: a public key and a private key.

**Polished Answer:**

| Feature | Symmetric Encryption | Asymmetric Encryption |
|---------|---------------------|----------------------|
| **Keys** | Single shared key | Key pair (public + private) |
| **Speed** | Fast | Slow (100-1000x slower) |
| **Security** | Key distribution risk | More secure key exchange |
| **Examples** | AES, DES, 3DES, Blowfish | RSA, ECC, Diffie-Hellman |
| **Use Case** | Bulk data encryption | Key exchange, digital signatures |

**Symmetric Encryption:**
- Same key encrypts and decrypts
- Both parties must share the secret key beforehand
- Problem: How to securely share the key?
- Very fast and efficient for bulk data
- Example: AES (Advanced Encryption Standard) - 128/256 bit

**Asymmetric Encryption (Public Key Cryptography):**
- Public key encrypts; private key decrypts (or vice versa)
- Public key can be freely shared; private key kept secret
- Solves key distribution problem
- Slower due to complex mathematical operations
- Example: RSA (2048/4096 bit), ECC

**How TLS uses both:**
1. Asymmetric encryption for handshake (securely exchange session key)
2. Symmetric encryption for data transfer (speed and efficiency)

**TL;DR:** Symmetric = one key, fast, key distribution problem. Asymmetric = two keys, slow, secure key exchange.

**Keywords:** Symmetric encryption, asymmetric encryption, public key, private key, AES, RSA, TLS

---

### Q22: What is DNS cache poisoning and how can it be prevented?

**Answer:** DNS cache poisoning occurs when an attacker inserts a fake DNS record into a DNS cache, causing users to be redirected to malicious websites.

**Polished Answer:**

**What is DNS Cache Poisoning?**
A security attack where an attacker injects false DNS records into a DNS resolver's cache, causing users to be redirected to malicious IP addresses instead of legitimate websites.

**How it works:**
1. Attacker sends forged DNS responses to a DNS resolver
2. Resolver caches the fake record without verification
3. Users querying the resolver get incorrect IP addresses
4. Users are redirected to phishing or malware sites

**Example:**
- User tries to visit bank.com
- DNS cache has been poisoned to return attacker's IP instead of legitimate bank IP
- User sees what looks like bank.com but is actually a phishing site

**Prevention methods:**

**1. DNSSEC (DNS Security Extensions):**
- Digitally signs DNS records
- Resolvers verify signatures before accepting records
- Most robust solution but complex deployment

**2. Use Trusted DNS Servers:**
- Configure reliable DNS servers
- Limit recursive queries to authorized users

**3. Regular Cache Maintenance:**
- Clear suspicious cache entries
- Set appropriate TTL values

**4. DNSSEC Validation:**
- Resolvers should validate DNSSEC signatures
- Reject unsigned or improperly signed records

**5. Source Port Randomization:**
- Randomize source ports for DNS queries
- Makes it harder for attackers to guess transaction IDs

**TL;DR:** DNS poisoning = fake records in cache. Prevent with DNSSEC, trusted servers, port randomization.

**Keywords:** DNS cache poisoning, DNSSEC, DNS security, cache poisoning, DNS spoofing, phishing

---

### Q23: What is the role of port numbers in transport-layer communication?

**Answer:** Port numbers identify specific applications or services running on a device. They allow multiple applications to use the network at the same time.

**Polished Answer:**

**What are Port Numbers?**
16-bit identifiers (0-65535) that identify specific processes or services running on a host, enabling multiple applications to share a single network connection.

**Common Well-Known Ports:**

| Port | Protocol | Service |
|------|----------|---------|
| 20/21 | TCP | FTP (Data/Control) |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 67/68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 |
| 143 | TCP | IMAP |
| 161 | UDP | SNMP |
| 443 | TCP | HTTPS |

**Port Ranges:**

| Range | Type | Purpose |
|-------|------|---------|
| 0-1023 | Well-known ports | System-level services |
| 1024-49151 | Registered ports | Application services |
| 49152-65535 | Ephemeral ports | Temporary client connections |

**How Ports Enable Multiplexing:**
- IP address identifies the host
- Port number identifies the specific application on that host
- Combination (IP + Port) = Socket, uniquely identifying a communication endpoint

**Ephemeral Ports:**
- Temporary ports assigned by OS for outgoing connections
- Client uses ephemeral port to connect to server's well-known port
- Example: Browser connecting to 443 on server from ephemeral port 54321

**TL;DR:** Ports identify services on a host. IP + Port = socket. Well-known (0-1023), Registered (1024-49151), Ephemeral (49152-65535).

**Keywords:** Port numbers, sockets, well-known ports, ephemeral ports, TCP ports, UDP ports

---

### Q24: What is a Collision Domain and Broadcast Domain?

**Answer:** A collision domain is a network segment where devices compete for transmission, causing collisions. A broadcast domain is a set of devices that receive the same broadcast frame.

**Polished Answer:**

**Collision Domain:**
- A network segment where two or more devices share the same transmission medium
- Simultaneous transmissions cause collisions
- All devices in the same collision domain compete for bandwidth
- **Devices that separate collision domains:** Switches (each port = separate collision domain), Routers, Bridges
- **Devices that extend collision domain:** Hubs, Repeaters

**Broadcast Domain:**
- A logical network segment where all devices receive broadcast frames
- Broadcast traffic (e.g., ARP requests) reaches all devices in the domain
- Larger broadcast domains = more broadcast overhead
- **Devices that separate broadcast domains:** Routers, Layer 3 switches, VLANs
- **Devices that extend broadcast domain:** Switches, Hubs (without VLANs)

**Example:**
- 24-port switch = 24 separate collision domains, but 1 broadcast domain
- Adding VLANs to the switch = multiple broadcast domains
- Router connecting to the switch = separates broadcast domains

**Why it matters:**
- Too many collision domains: Not an issue in modern switched networks (full-duplex)
- Too large broadcast domains: Broadcast storms, performance degradation
- Proper segmentation improves security and performance

**TL;DR:** Collision domain = devices competing for medium (separated by switches). Broadcast domain = devices receiving broadcasts (separated by routers/VLANs).

**Keywords:** Collision domain, broadcast domain, switch, router, VLAN, network segmentation

---

### Q25: Compare Distance Vector and Link-State Routing Protocols.

**Answer:** Distance Vector protocols share routing information with neighboring routers, while Link-State protocols maintain a complete network map. Link-State converges faster and scales better.

**Polished Answer:**

| Feature | Distance Vector | Link-State |
|---------|----------------|------------|
| **Knowledge** | Only knows neighbors' routes | Complete network topology map |
| **Updates** | Periodic to neighbors only | Triggered updates to all routers |
| **Convergence** | Slow (count-to-infinity problem) | Fast (triggered updates) |
| **Bandwidth** | Low overhead | Higher initial overhead |
| **Scalability** | Limited (15 hops for RIP) | Excellent (large networks) |
| **Examples** | RIP, EIGRP (hybrid) | OSPF, IS-IS |
| **Algorithm** | Bellman-Ford | Dijkstra (Shortest Path First) |

**Distance Vector (RIP example):**
- Routers exchange routing tables with direct neighbors
- Each router computes best path based on hop count
- "Routing by rumor" - relies on neighbors' information
- Problems: Slow convergence, count-to-infinity, routing loops

**Link-State (OSPF example):**
- Each router builds a complete map of the network
- Routers flood Link-State Advertisements (LSAs)
- Each router independently computes shortest path using Dijkstra's algorithm
- All routers have identical topology database

**Why OSPF converges faster than RIP:**
- Triggered updates (not periodic)
- Complete network knowledge (not just neighbor info)
- Dijkstra algorithm efficiently computes shortest paths

**TL;DR:** Distance Vector = shares routes with neighbors, slow convergence. Link-State = complete topology map, fast convergence.

**Keywords:** Distance Vector, Link-State, RIP, OSPF, routing protocols, convergence, Dijkstra algorithm

---

## TOP 26-50 MOST IMPORTANT NETWORKING QUESTIONS

---

### Q26: What is the Bandwidth-Delay Product (BDP)?

**Answer:** The Bandwidth-Delay Product (BDP) is the amount of data that can be in transit before an acknowledgment is received. It is calculated as Bandwidth × Round-Trip Time (RTT).

**Polished Answer:**

**What is BDP?**
BDP represents the maximum amount of unacknowledged data that can be in transit in the network at any given time.

**Formula:** BDP = Bandwidth × RTT (Round-Trip Time)

**Example:**
- Bandwidth = 100 Mbps (100,000,000 bits/second)
- RTT = 50 ms (0.05 seconds)
- BDP = 100,000,000 × 0.05 = 5,000,000 bits = 625 KB

**Significance:**
- Determines optimal TCP window size for maximum throughput
- If window < BDP: Underutilized bandwidth
- If window > BDP: May cause congestion
- Critical for high-bandwidth, high-latency networks (e.g., satellite, long-distance links)

**Pipelining and BDP:**
- Pipelining allows multiple packets in transit
- Higher BDP enables more data in flight
- Improves bandwidth utilization
- Reduces idle time waiting for ACKs

**TL;DR:** BDP = Bandwidth × RTT. Determines optimal window size for maximum throughput.

**Keywords:** BDP, Bandwidth-Delay Product, RTT, pipelining, TCP window, throughput optimization

---

### Q27: What is Route Flapping and how do routing protocols minimize its impact?

**Answer:** Route flapping occurs when a route repeatedly changes between available and unavailable states. Routing protocols use mechanisms like hold-down timers to suppress unstable routes.

**Polished Answer:**

**What is Route Flapping?**
A condition where a network route repeatedly transitions between up and down states in rapid succession, causing instability in the routing system.

**Effects of Route Flapping:**
- Excessive routing updates flooding the network
- Increased CPU utilization on routers
- Frequent reconvergence
- Network instability
- Potential routing loops

**How routing protocols minimize impact:**

**1. BGP Route Dampening:**
- Tracks flapping history
- Temporarily suppresses (dampens) routes that flap frequently
- Route is restored after flap rate decreases

**2. Hold-Down Timers:**
- Wait before accepting frequent route changes
- Prevents premature route acceptance
- Gives network time to stabilize

**3. OSPF SPF Throttling:**
- Reduces how often OSPF recalculates routes
- Delays SPF calculations after rapid changes
- Prevents excessive CPU usage

**4. Link-State Update Throttling:**
- Limits frequency of LSA flooding
- Prevents network from being overwhelmed

**TL;DR:** Route flapping = rapid up/down route changes. Suppressed with dampening, hold-down timers, throttling.

**Keywords:** Route flapping, BGP dampening, hold-down timer, SPF throttling, routing instability

---

### Q28: What is BGP and why is it called a Path-Vector Protocol?

**Answer:** BGP (Border Gateway Protocol) is used to exchange routing information between different Autonomous Systems (AS) on the Internet. It advertises the complete AS path for each route.

**Polished Answer:**

**What is BGP?**
Border Gateway Protocol is the routing protocol of the Internet, responsible for routing between different Autonomous Systems (ASes - networks under single administrative control).

**Why Path-Vector?**
- BGP advertises the complete AS path for each route
- Each route includes the sequence of ASes through which traffic must travel
- Example AS Path: 64500 → 64501 → 64502 (traffic passes through three ASes)

**Key features:**
- **Inter-domain routing:** Routes between different organizations/ISPs
- **Policy-based:** Administrators control traffic flow through routing policies
- **Loop prevention:** Rejects routes containing own AS number in path
- **Scalability:** Designed for Internet-scale routing (900K+ routes)
- **Reliability:** Uses TCP (port 179) for reliable communication

**BGP vs IGP:**
- BGP = Exterior Gateway Protocol (between ASes)
- IGP (OSPF, RIP) = Interior Gateway Protocol (within an AS)
- BGP focuses on policy; IGP focuses on path optimization

**TL;DR:** BGP routes between Autonomous Systems. Path-vector = advertises complete AS path, prevents loops.

**Keywords:** BGP, Border Gateway Protocol, Autonomous System, path-vector, inter-domain routing, AS path

---

### Q29: What are CIDR and Equal-Cost Multi-Path (ECMP)?

**Answer:** CIDR combines multiple network prefixes into a single route to reduce routing table size. ECMP distributes traffic across multiple paths with equal routing cost.

**Polished Answer:**

**CIDR (Classless Inter-Domain Routing):**
- Replaces classful addressing (Class A, B, C)
- Allows variable-length subnet masks (VLSM)
- Enables route aggregation/supernetting

**Route Aggregation example:**
- 192.168.0.0/24, 192.168.1.0/24, 192.168.2.0/24, 192.168.3.0/24
- All can be aggregated into 192.168.0.0/22
- Reduces 4 routing entries to 1

**Benefits:**
- Smaller routing tables (Internet has ~900K routes vs millions without CIDR)
- Efficient IP address allocation
- Reduced routing update traffic

**ECMP (Equal-Cost Multi-Path):**
- When multiple paths have equal cost (metric) to the same destination
- Router distributes traffic across all equal-cost paths
- Increases aggregate bandwidth
- Provides load balancing and redundancy

**How ECMP works:**
- Router identifies multiple next-hops with equal cost
- Traffic is hashed (source/destination IP, ports) to select path
- All paths utilized simultaneously
- Automatic failover if one path fails

**Benefits:**
- Improved bandwidth utilization
- Load balancing
- Increased reliability
- Better network performance

**TL;DR:** CIDR = route aggregation (smaller tables). ECMP = load balancing across equal-cost paths.

**Keywords:** CIDR, route aggregation, ECMP, Equal-Cost Multi-Path, routing table, load balancing

---

### Q30: Differentiate between IGP and EGP.

**Answer:** IGP (Interior Gateway Protocol) routes within a single Autonomous System. EGP (Exterior Gateway Protocol) routes between different Autonomous Systems.

**Polished Answer:**

| Feature | IGP | EGP |
|---------|-----|-----|
| **Scope** | Within a single AS | Between different ASes |
| **Purpose** | Internal routing | Internet/external routing |
| **Convergence** | Fast (internal network) | Slower (Internet scale) |
| **Focus** | Path optimization | Routing policy |
| **Examples** | RIP, OSPF, EIGRP, IS-IS | BGP |
| **Route Count** | Hundreds-thousands | Hundreds of thousands |

**IGP details:**
- Used within an organization's network
- Optimizes for shortest/fastest path
- Routers trust each other (same admin control)
- Examples: OSPF (link-state), RIP (distance vector), EIGRP (Cisco hybrid)

**EGP details:**
- Used between ISPs and large networks
- Focus on policy compliance, not just shortest path
- Routers don't implicitly trust each other
- BGP is the only EGP used on the Internet today

**Why both are needed:**
- IGP handles internal routing efficiently
- EGP handles inter-domain routing with policy control
- Organizations need both for complete connectivity

**TL;DR:** IGP = routing within one organization (OSPF, RIP). EGP = routing between organizations (BGP).

**Keywords:** IGP, EGP, Autonomous System, OSPF, RIP, BGP, routing protocols, interior/exterior

---

### Q31: What is route redistribution?

**Answer:** Route redistribution is the process of sharing routes learned by one routing protocol with another routing protocol.

**Polished Answer:**

**What is Route Redistribution?**
The process of taking routes from one routing protocol and advertising them into another routing protocol, enabling communication between networks using different protocols.

**Why use route redistribution?**
- Multiple routing protocols may exist in the same network
- Different parts of the network may use different protocols
- Redistribution enables connectivity between these parts

**Common scenarios:**
- OSPF on campus network, EIGRP in data center
- RIP on legacy equipment, OSPF on newer equipment
- IGP (internal) redistributed into BGP (external)

**Considerations:**

**1. Metric Translation:**
- Different protocols use different metrics (hop count, cost, bandwidth)
- Must set appropriate metric when redistributing
- Example: RIP hop count → OSPF cost mapping

**2. Route Filtering:**
- Control which routes are redistributed
- Prevent feedback loops (route learned from OSPF should not be redistributed back)

**3. Loop Prevention:**
- Incorrect redistribution can create routing loops
- Use route tags and filters to prevent loops

**4. Administrative Distance:**
- Router may prefer redistributed routes over native routes
- Must configure carefully to maintain proper path selection

**TL;DR:** Route redistribution = sharing routes between routing protocols. Requires careful metric and loop management.

**Keywords:** Route redistribution, OSPF, EIGRP, RIP, metric translation, route filtering, routing loops

---

### Q32: Why are private IP addresses not routable on the Internet?

**Answer:** Private IP addresses are reserved for local networks and are not globally unique. Internet routers do not forward these addresses. NAT converts them to public IPs for Internet communication.

**Polished Answer:**

**Private IP Address Ranges (RFC 1918):**

| Class | Range | CIDR | Usable Hosts |
|-------|-------|------|--------------|
| A | 10.0.0.0 - 10.255.255.255 | 10.0.0.0/8 | ~16.7 million |
| B | 172.16.0.0 - 172.31.255.255 | 172.16.0.0/12 | ~1 million |
| C | 192.168.0.0 - 192.168.255.255 | 192.168.0.0/16 | ~65,536 |

**Why not routable on Internet?**
- Private IPs are not globally unique (any organization can use them)
- Internet routers have no routing information for private ranges
- If routed, packets couldn't be delivered correctly (ambiguous addresses)
- Multiple organizations using same private range would conflict

**How Internet access works:**
- Private IPs used internally
- NAT translates to public IPs for Internet communication
- Router maintains NAT table for connection tracking

**Special/Loopback addresses:**
- 127.0.0.0/8: Loopback addresses for local testing
- Example: 127.0.0.1 always refers to the local machine

**TL;DR:** Private IPs (10.x, 172.16-31.x, 192.168.x) are non-unique → NAT translates to public IPs for Internet access.

**Keywords:** Private IP, RFC 1918, NAT, loopback, 127.0.0.1, non-routable, public IP

---

### Q33: What is a Default Gateway?

**Answer:** A default gateway is the router that forwards packets from a local network to other networks or the Internet.

**Polished Answer:**

**What is a Default Gateway?**
The router (or Layer 3 device) on a local network that serves as the exit point for traffic destined outside the local subnet.

**How it works:**
1. Device wants to communicate with a remote network
2. Device checks if destination is on local subnet (using subnet mask)
3. If not local, device sends packet to default gateway (router)
4. Router determines the next hop and forwards the packet

**Example:**
- PC: 192.168.1.10/24 (subnet mask 255.255.255.0)
- Default Gateway: 192.168.1.1 (router's local interface)
- Traffic to 192.168.1.x → sent directly (same subnet)
- Traffic to anything else → sent to 192.168.1.1

**Key points:**
- Each subnet must have a default gateway for external communication
- Gateway IP must be in the same subnet as the device
- Without default gateway: no Internet access (local only)
- Multiple gateways possible for redundancy (but one active at a time)

**TL;DR:** Default gateway = router that forwards traffic to external networks. Essential for Internet access.

**Keywords:** Default gateway, router, subnet, next hop, network routing, Internet access

---

### Q34: Why does IPv6 eliminate the need for NAT?

**Answer:** IPv6 uses 128-bit addresses, providing a vast number of unique IP addresses. This allows every device to have its own public IP address, eliminating the need for NAT.

**Polished Answer:**

**IPv6 Address Space:**
- 128-bit addresses (vs 32-bit IPv4)
- Total addresses: 340 undecillion (3.4 × 10³⁸)
- Enough for every person and device with unique addresses

**Why NAT isn't needed:**

**1. Abundant Addresses:**
- Every device can have a globally unique address
- No need to share addresses through NAT

**2. End-to-End Connectivity:**
- No address translation = direct communication
- Better for peer-to-peer, VoIP, IoT

**3. Simpler Network Design:**
- No NAT tables to maintain
- No port forwarding requirements
- Reduced router processing

**IPv6 features beyond NAT elimination:**
- Auto-configuration (SLAAC) without DHCP
- Built-in IPSec support
- Simplified header for efficient routing
- Better multicast support

**Comparison:**

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Address Size | 32-bit | 128-bit |
| NAT Required | Yes (address shortage) | No (abundant addresses) |
| Header | Complex | Simplified |
| Auto-config | DHCP or manual | SLAAC or DHCPv6 |
| Broadcast | Yes | No (multicast only) |

**TL;DR:** IPv6 has 128-bit addresses = enough for every device to have unique public IP. NAT unnecessary.

**Keywords:** IPv6, NAT, 128-bit address, unique IP, end-to-end connectivity, SLAAC, address exhaustion

---

### Q35: How does a switch learn MAC addresses?

**Answer:** A switch learns MAC addresses by reading the source MAC address of incoming frames and storing it in its MAC address table. Unknown destination MACs are flooded to all ports.

**Polished Answer:**

**MAC Address Learning Process:**

1. **Initial State:** Switch has an empty MAC address table

2. **Frame Arrival:** When a frame arrives on a port, the switch reads the source MAC address

3. **Table Update:** Switch records the source MAC address and the port where it arrived

4. **Forwarding Decision:**
   - If destination MAC is known: Forward frame only to the corresponding port
   - If destination MAC is unknown: Flood the frame to all ports except the incoming port

5. **Learning Continues:** Switch constantly updates its table as frames arrive

**Example:**
- PC A (MAC: AA:AA:AA) connected to Port 1
- PC B (MAC: BB:BB:BB) connected to Port 2
- PC A sends a frame to PC B:
  - Switch records: Port 1 → AA:AA:AA
  - PC B's MAC is unknown → floods to all ports
  - PC B responds
  - Switch records: Port 2 → BB:BB:BB
  - Subsequent frames forwarded directly Port 1 ↔ Port 2

**MAC Address Table:**
- Port-to-MAC mappings
- Entries age out after inactivity (typically 5 minutes)
- Limited table size (depends on switch model)

**TL;DR:** Switch learns source MACs from incoming frames. Unknown destinations flood to all ports. Known destinations unicast to specific port.

**Keywords:** MAC address table, switch learning, frame forwarding, flooding, MAC aging, source MAC

---

### Q36: What is Store-and-Forward vs Cut-Through switching?

**Answer:** Store-and-forward receives the entire frame, checks CRC, then forwards. Cut-through starts forwarding as soon as destination MAC is read.

**Polished Answer:**

| Feature | Store-and-Forward | Cut-Through |
|---------|-------------------|-------------|
| **Process** | Receives entire frame, checks CRC, forwards | Forwards as soon as destination MAC is read |
| **Latency** | Higher (waits for full frame) | Low (minimal delay) |
| **Error Detection** | Detects corrupted frames | May forward corrupted frames |
| **Reliability** | More reliable | Less reliable |
| **Use Case** | Enterprise networks, quality-critical | Low-latency environments |

**Store-and-Forward:**
- Receives the complete frame
- Validates frame using CRC (Cyclic Redundancy Check)
- Discards corrupted frames
- Forwards only valid frames
- Adds latency proportional to frame size

**Cut-Through:**
- Reads only the destination MAC address (first 6 bytes after preamble)
- Immediately begins forwarding
- Minimal latency (few microseconds)
- May forward corrupted frames (doesn't check CRC)
- Faster but less reliable

**Variants of Cut-Through:**
- **Fragment-Free:** Reads first 64 bytes (minimum frame size), reduces forwarding of collision fragments
- **Fast-Forward:** Pure cut-through, forwards after reading destination MAC

**TL;DR:** Store-and-forward = verify then forward (reliable, slower). Cut-through = forward immediately (fast, less reliable).

**Keywords:** Store-and-forward, cut-through switching, CRC check, latency, frame forwarding, error detection

---

### Q37: Explain Spanning Tree Protocol (STP) and the Root Bridge.

**Answer:** STP prevents switching loops by creating a loop-free topology. It elects a Root Bridge as the reference point for path selection.

**Polished Answer:**

**What is STP?**
Spanning Tree Protocol (IEEE 802.1D) prevents Layer 2 switching loops by creating a loop-free logical topology while maintaining redundant physical links for failover.

**Why STP is needed:**
- Redundant links prevent single points of failure
- But redundant links create switching loops
- Loops cause broadcast storms, MAC table instability
- STP blocks redundant paths to break loops

**Root Bridge Election:**
1. All switches exchange Bridge Protocol Data Units (BPDUs)
2. Switch with the lowest Bridge ID becomes Root Bridge
3. Bridge ID = Priority (default 32768) + MAC address
4. Lower priority wins; if tie, lower MAC address wins

**Port Roles:**
- **Root Port:** Best path to Root Bridge (one per non-root switch)
- **Designated Port:** Best path on each segment
- **Blocked Port:** Redundant ports blocked to prevent loops

**How it works:**
- Non-root switches identify best path to root
- Redundant ports are placed in blocking state
- If active path fails, blocked port transitions to forwarding

**STP Port States:**
- Blocking → Listening → Learning → Forwarding
- Transition takes ~30-50 seconds (rapid STP reduces this)

**TL;DR:** STP prevents loops by blocking redundant paths. Root Bridge = lowest Bridge ID. Blocked ports activate on failure.

**Keywords:** STP, Spanning Tree, Root Bridge, BPDU, switching loops, redundant links, port states

---

### Q38: What is VLAN Tagging (IEEE 802.1Q)?

**Answer:** VLAN tagging adds a VLAN identifier to Ethernet frames, allowing multiple VLANs to share the same physical network.

**Polished Answer:**

**What is 802.1Q?**
The IEEE standard for VLAN tagging that inserts a 4-byte tag into the Ethernet frame header, identifying which VLAN the frame belongs to.

**How it works:**
- Tag added between Source MAC and EtherType fields
- Tag contains VLAN ID (12 bits, values 1-4094)
- VLAN 0 = priority only; VLAN 1 = default VLAN
- Tagged frames travel over trunk links between switches

**Why VLAN tagging is used:**

**1. Multi-VLAN Transport:**
- Single physical link carries traffic for multiple VLANs
- Trunk ports use tagging; access ports are untagged

**2. Network Segmentation:**
- Logical separation of traffic types
- Each VLAN isolated from others

**3. Scalability:**
- Extend VLANs across multiple switches
- No physical rewiring needed for VLAN changes

**4. Security:**
- Traffic isolation between VLANs
- Prevents unauthorized access between VLANs

**Trunk vs Access Ports:**
- **Access Port:** Single VLAN, untagged traffic (end devices)
- **Trunk Port:** Multiple VLANs, tagged traffic (switch-to-switch)

**TL;DR:** 802.1Q adds VLAN ID to frames. Enables multiple VLANs over one link. Trunk ports carry tagged traffic.

**Keywords:** 802.1Q, VLAN tagging, trunk port, access port, VLAN ID, Ethernet frame

---

### Q39: What is PoE (Power over Ethernet)?

**Answer:** Power over Ethernet allows a single Ethernet cable to carry both data and electrical power, eliminating separate power cables.

**Polished Answer:**

**What is PoE?**
Technology that delivers electrical power over standard Ethernet cabling (Cat5e/6), providing both network connectivity and power through a single cable.

**How it works:**
- Power Sourcing Equipment (PSE) injects power into Ethernet cable
- Powered Devices (PDs) receive power and data through the same cable
- Uses unused wire pairs or phantom power over data pairs

**PoE Standards:**

| Standard | Max Power | Applications |
|----------|-----------|--------------|
| PoE (802.3af) | 15.4W | IP phones, basic cameras |
| PoE+ (802.3at) | 30W | PTZ cameras, advanced APs |
| PoE++ (802.3bt) | 60-100W | LED lighting, laptops |

**Benefits:**

**1. Simplified Installation:**
- No need for electrical outlets at device location
- Single cable for power and data

**2. Cost Reduction:**
- Eliminates separate power cabling
- Reduces electrician costs

**3. Flexibility:**
- Devices can be placed anywhere (ceilings, outdoors)
- Easy to reposition without rewiring

**4. Reliability:**
- Centralized power management (UPS backup)
- Remote power monitoring and control

**Common PoE devices:**
- IP phones
- Wireless access points
- IP cameras
- IoT sensors
- Network switches

**TL;DR:** PoE = power + data over one Ethernet cable. Standards: PoE, PoE+, PoE++. Used for IP phones, cameras, APs.

**Keywords:** PoE, Power over Ethernet, 802.3af, 802.3at, 802.3bt, PSE, PD, IP camera

---

### Q40: Why is DNS hierarchical instead of centralized?

**Answer:** DNS uses a hierarchical structure to improve scalability, reliability, and performance. It distributes information among Root, TLD, and Authoritative servers.

**Polished Answer:**

**DNS Hierarchy:**

```
Root DNS Servers (.)
├── .com TLD
│   ├── google.com (Authoritative)
│   └── facebook.com (Authoritative)
├── .org TLD
│   └── wikipedia.org (Authoritative)
└── .net TLD
    └── example.net (Authoritative)
```

**Why hierarchical, not centralized:**

**1. Scalability:**
- Single server can't handle billions of queries
- Distribution spreads load across millions of servers
- Each level handles only its portion of queries

**2. Reliability:**
- No single point of failure
- Redundancy at every level
- Multiple root servers (13 logical, hundreds physical)

**3. Performance:**
- Queries resolved closer to the source
- Caching at multiple levels reduces latency
- Distributed architecture enables parallel resolution

**4. Management:**
- Each domain owner manages their own records
- No central authority needed for all changes
- Delegation of authority

**5. Fault Isolation:**
- Failure in one domain doesn't affect others
- Problems are contained within their zone

**TL;DR:** DNS hierarchy (Root → TLD → Authoritative) distributes load, improves reliability, enables scalability.

**Keywords:** DNS hierarchy, Root servers, TLD, authoritative servers, scalability, domain resolution

---

### Q41: Compare HTTP/1.1, HTTP/2, and HTTP/3.

**Answer:** HTTP/1.1 is sequential TCP-based. HTTP/2 supports multiplexing over TCP. HTTP/3 uses QUIC over UDP for fastest performance.

**Polished Answer:**

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| **Transport** | TCP | TCP | QUIC over UDP |
| **Multiplexing** | No (sequential) | Yes (streams) | Yes (independent streams) |
| **Head-of-Line Blocking** | Yes (severe) | Partial (TCP level) | No (stream level) |
| **Header Compression** | No | HPACK | QPACK |
| **Connection Setup** | Separate per request | One TCP connection | Faster (0-RTT) |
| **Speed** | Slowest | Faster | Fastest |
| **Year** | 1997 | 2015 | 2022 |

**HTTP/1.1:**
- Sequential request processing
- Multiple TCP connections for parallelism (6-8 per domain)
- Head-of-line blocking on each connection
- No header compression

**HTTP/2:**
- Multiplexing: Multiple requests/responses over one TCP connection
- Header compression with HPACK
- Server push capability
- Still uses TCP → TCP-level head-of-line blocking persists

**HTTP/3:**
- Uses QUIC (Quick UDP Internet Connections)
- Independent streams eliminate head-of-line blocking
- 0-RTT connection establishment (faster than TCP handshake)
- Better performance on unstable/mobile networks
- Built-in TLS 1.3

**TL;DR:** HTTP/1.1 = sequential TCP. HTTP/2 = multiplexed TCP. HTTP/3 = multiplexed QUIC over UDP, fastest.

**Keywords:** HTTP/1.1, HTTP/2, HTTP/3, QUIC, multiplexing, head-of-line blocking, transport protocol

---

### Q42: Why does ICMP not use TCP?

**Answer:** ICMP operates at the Network Layer for error reporting and diagnostics. It doesn't use TCP because it sends small control messages, and TCP would add unnecessary connection overhead.

**Polished Answer:**

**What is ICMP?**
Internet Control Message Protocol - a Network Layer protocol used for error reporting, diagnostics, and network troubleshooting.

**Why not TCP?**

**1. Different Purpose:**
- ICMP: Control and diagnostic messages (not application data)
- TCP: Reliable application data delivery

**2. Layer Difference:**
- ICMP operates at Network Layer (Layer 3)
- TCP operates at Transport Layer (Layer 4)
- ICMP is encapsulated directly in IP packets

**3. Overhead Avoidance:**
- TCP requires connection establishment (3-way handshake)
- TCP adds sequence numbers, ACKs, flow control
- Unnecessary overhead for simple control messages

**4. Self-Containment:**
- ICMP messages are small and self-descriptive
- Don't need reliability features of TCP
- Some ICMP messages are sent when TCP itself fails

**Common ICMP uses:**
- Ping (Echo Request/Reply)
- Traceroute (Time Exceeded messages)
- Destination unreachable errors
- Redirect messages
- Path MTU discovery

**TL;DR:** ICMP is Layer 3 diagnostic protocol. TCP overhead unnecessary for control messages. ICMP encapsulated directly in IP.

**Keywords:** ICMP, TCP, Network Layer, diagnostics, ping, traceroute, error reporting

---

### Q43: Compare Packet Switching and Circuit Switching.

**Answer:** Circuit switching establishes a dedicated path before data transfer. Packet switching divides data into packets that travel independently.

**Polished Answer:**

| Feature | Circuit Switching | Packet Switching |
|---------|-------------------|------------------|
| **Path** | Dedicated path established | No dedicated path |
| **Bandwidth** | Reserved for connection | Shared dynamically |
| **Setup** | Required before transfer | Not required |
| **Efficiency** | Resources idle when no data | Efficient resource use |
| **Reliability** | Consistent performance | Variable (congestion) |
| **Examples** | Traditional telephone | Internet, modern networks |
| **Cost** | Higher (dedicated resources) | Lower (shared) |

**Circuit Switching:**
- Dedicated physical path reserved
- Fixed bandwidth for entire connection
- Predictable latency
- Resources wasted when idle
- Example: Traditional telephone calls

**Packet Switching:**
- Data divided into packets
- Packets travel independently
- Packets may take different routes
- Network bandwidth shared
- More efficient resource use
- Handles varying traffic loads
- Example: Internet traffic

**Why Internet uses packet switching:**
- Efficient resource utilization
- Better for bursty traffic patterns
- Fault tolerance (packets reroute)
- Cost-effective for shared infrastructure

**TL;DR:** Circuit = dedicated path (telephone). Packet = independent routing (Internet). Packet switching more efficient.

**Keywords:** Packet switching, circuit switching, bandwidth, dedicated path, routing, efficiency

---

### Q44: What are MTU and Path MTU Discovery?

**Answer:** MTU is the largest packet size that can be transmitted without fragmentation. Path MTU Discovery identifies the smallest MTU along the communication path.

**Polished Answer:**

**MTU (Maximum Transmission Unit):**
- Largest packet/frame size a network interface can transmit without fragmentation
- Default MTU: 1500 bytes for Ethernet
- Larger packets require fragmentation
- Fragmentation reduces efficiency and increases overhead

**Path MTU Discovery (PMTUD):**
- Determines the smallest MTU along the entire path from source to destination
- Identifies the maximum packet size that can transit without fragmentation

**How PMTUD works:**
1. Sender sends packets with DF (Don't Fragment) flag set
2. If a router's MTU is smaller, it drops the packet
3. Router sends ICMP "Fragmentation Needed" message with its MTU
4. Sender reduces packet size and retries
5. Process repeats until packets reach destination

**Why it matters:**
- Fragmentation adds overhead and reduces performance
- Prevents packet loss due to oversized packets
- Critical for VPNs (reduced MTU due to encapsulation)
- Important for IPv6 (no router fragmentation)

**Common MTU sizes:**
- Ethernet: 1500 bytes
- PPPoE: 1492 bytes
- Jumbo frames: 9000 bytes
- IPv6 minimum: 1280 bytes

**TL;DR:** MTU = max packet size. PMTUD finds smallest MTU along path. Prevents fragmentation.

**Keywords:** MTU, Path MTU Discovery, PMTUD, fragmentation, packet size, DF flag, ICMP

---

### Q45: What is RTT (Round Trip Time)?

**Answer:** Round Trip Time is the total time taken for a packet to travel from sender to receiver and for the response to return.

**Polished Answer:**

**What is RTT?**
The time measured from when a packet is sent until the acknowledgment or response is received back.

**Formula:** RTT = Time for packet to reach destination + Time for response to return

**How to measure:**
- `ping` command displays RTT
- Each ping reply shows time in milliseconds
- Multiple pings give average RTT

**Factors affecting RTT:**
- Physical distance (propagation delay)
- Transmission medium (fiber vs copper)
- Number of router hops
- Network congestion (queuing delay)
- Processing time at routers

**Significance:**
- Determines TCP timeout values
- Affects TCP window sizing (BDP = bandwidth × RTT)
- Critical for real-time applications
- Lower RTT = better network performance

**Typical RTT values:**
- Local network: <1 ms
- Same city: 5-20 ms
- Cross-country: 30-80 ms
- International: 100-300 ms
- Satellite: 500-600 ms

**TL;DR:** RTT = time for packet + response. Measures network latency. Affects TCP performance.

**Keywords:** RTT, Round Trip Time, latency, ping, network delay, propagation delay

---

### Q46: What are propagation delay and transmission delay?

**Answer:** Propagation delay is the time for a signal to travel through the medium. Transmission delay is the time to place all bits of a packet onto the link.

**Polished Answer:**

**Propagation Delay:**
- Time for a signal to physically travel from sender to receiver
- Depends on distance and medium speed
- Formula: Propagation Delay = Distance / Signal Speed
- Signal speed in fiber: ~200,000 km/s (2/3 of light speed)
- Example: 1000 km fiber link → 1000/200,000 = 5 ms

**Transmission Delay:**
- Time to push all bits of a packet onto the wire
- Depends on packet size and link bandwidth
- Formula: Transmission Delay = Packet Size / Bandwidth
- Example: 1500 bytes (12,000 bits) at 100 Mbps = 0.12 ms

**Total Network Latency = Propagation + Transmission + Processing + Queuing**

| Delay Type | Depends On | Example |
|------------|------------|---------|
| Propagation | Distance, medium | Long-distance links |
| Transmission | Packet size, bandwidth | Low-bandwidth links |
| Processing | Router CPU | Packet inspection |
| Queuing | Network congestion | Busy routers |

**Which dominates?**
- Long-distance: Propagation delay dominates
- Low-bandwidth: Transmission delay dominates
- Congested network: Queuing delay dominates

**TL;DR:** Propagation = distance/speed. Transmission = size/bandwidth. Both contribute to total latency.

**Keywords:** Propagation delay, transmission delay, latency, bandwidth, network delay, queuing delay

---

### Q47: Why is UDP preferred for streaming and online gaming?

**Answer:** UDP is preferred for real-time applications because it offers low latency and minimal overhead. Small packet losses are acceptable.

**Polished Answer:**

**Why UDP for real-time applications:**

**1. Low Latency:**
- No connection establishment (no handshake delay)
- No acknowledgment waiting
- No retransmission delays
- Packets sent immediately

**2. Minimal Overhead:**
- 8-byte header (vs 20-60 bytes for TCP)
- No connection state management
- No flow control processing

**3. Acceptable Packet Loss:**
- Streaming: A lost video frame is barely noticeable
- Gaming: A missed position update is quickly superseded
- Retransmitting old data is useless in real-time contexts

**4. No Head-of-Line Blocking:**
- Packet loss doesn't stall subsequent packets
- Data keeps flowing even with errors
- Critical for maintaining real-time experience

**Applications using UDP:**
- Video streaming (YouTube Live, Netflix)
- Voice over IP (VoIP)
- Online gaming (FPS, MOBA games)
- DNS queries
- Video conferencing
- Real-time protocols (RTP)

**Trade-off:**
- TCP retransmission would cause stuttering/freezing
- Late-arriving data is worthless in real-time apps
- UDP's speed > TCP's reliability for these use cases

**TL;DR:** UDP = low latency + minimal overhead. Small losses acceptable. Speed beats reliability for real-time.

**Keywords:** UDP, streaming, gaming, latency, real-time, VoIP, packet loss, overhead

---

### Q48: What is Nagle's Algorithm?

**Answer:** Nagle's algorithm reduces network congestion by combining many small TCP packets into fewer larger packets.

**Polished Answer:**

**What is Nagle's Algorithm?**
A TCP optimization that reduces the number of small packets sent by buffering data until it accumulates to a full segment or an ACK is received.

**How it works:**
1. If there is unacknowledged data in flight, buffer new data
2. Send buffered data when full segment size (MSS) is reached
3. Send buffered data when all previous data is acknowledged
4. Reduces packet count for small, frequent sends

**Advantages:**
- Reduces network congestion
- Reduces packet overhead
- Improves bandwidth efficiency
- Particularly useful for telnet, SSH (character-by-character input)

**Disadvantages:**
- Adds delay waiting for ACKs
- Harmful for interactive applications
- Can reduce performance in online gaming, VoIP

**When to disable Nagle's Algorithm:**
- Online gaming (needs immediate response)
- VoIP applications
- Real-time communication
- When using TCP_NODELAY socket option

**Example:**
- Without Nagle: Sending 10 characters generates 10 small packets
- With Nagle: First character sent immediately, others buffered until ACK or full segment

**TL;DR:** Nagle = buffer small packets into larger ones. Improves efficiency but adds delay. Disable for real-time.

**Keywords:** Nagle's algorithm, TCP optimization, small packet coalescing, latency, TCP_NODELAY, congestion reduction

---

### Q49: Differentiate between Flow Control and Congestion Control.

**Answer:** Flow control protects the receiver from overload. Congestion control protects the network from overload.

**Polished Answer:**

| Feature | Flow Control | Congestion Control |
|---------|-------------|-------------------|
| **Protects** | Receiver's buffer | Network capacity |
| **Goal** | Prevent receiver overflow | Prevent network congestion |
| **Mechanism** | Receive Window (rwnd) | Congestion Window (cwnd) |
| **Controlled By** | Receiver advertisement | Sender (based on network signals) |
| **Scope** | End-to-end | Network-wide |

**Flow Control:**
- Ensures sender doesn't overwhelm receiver's buffer
- Receiver advertises available buffer space (rwnd)
- Sender limits unacknowledged data to rwnd
- Example: Slow receiver with limited memory
- Mechanism: TCP Sliding Window

**Congestion Control:**
- Prevents network from becoming overloaded
- Sender adjusts rate based on congestion signals
- Signals: Packet loss (timeout, duplicate ACKs), ECN marks
- Example: Multiple senders saturating a link
- Mechanism: Slow Start, AIMD, Fast Recovery

**Key difference:**
- Flow control: Sender ↔ Receiver negotiation
- Congestion control: Sender detects network conditions
- Sender transmits min(cwnd, rwnd) bytes

**Both are essential for:**
- Reliable communication
- Network stability
- Fair bandwidth sharing

**TL;DR:** Flow control = receiver protection (rwnd). Congestion control = network protection (cwnd).

**Keywords:** Flow control, congestion control, rwnd, cwnd, TCP window, network protection, receiver buffer

---

### Q50: What is a Proxy Server? Forward Proxy vs Reverse Proxy?

**Answer:** A proxy server acts as an intermediary between client and server. Forward proxy sits in front of clients; reverse proxy sits in front of servers.

**Polished Answer:**

**What is a Proxy?**
An intermediary server that sits between clients and servers, forwarding requests and responses, potentially modifying or filtering them.

**Forward Proxy:**
- Sits in front of clients
- Clients send requests to proxy; proxy forwards to destination servers
- Hides client's IP from destination servers
- Used for: Content filtering, user anonymity, caching, corporate control

**Forward Proxy flow:**
```
Client → Forward Proxy → Internet → Server
```

**Reverse Proxy:**
- Sits in front of servers
- Clients send requests to proxy; proxy forwards to backend servers
- Hides server infrastructure from clients
- Used for: Load balancing, SSL termination, caching, security

**Reverse Proxy flow:**
```
Client → Internet → Reverse Proxy → Backend Server
```

| Feature | Forward Proxy | Reverse Proxy |
|---------|--------------|---------------|
| **Position** | Client side | Server side |
| **Hides** | Client identity | Server infrastructure |
| **Use Cases** | Corporate filtering, anonymity | Load balancing, security |
| **Examples** | Squid, corporate firewalls | Nginx, Cloudflare, HAProxy |

**Benefits:**
- Security (hide internal network)
- Load balancing (distribute traffic)
- Caching (reduce server load)
- SSL termination (offload encryption)
- Access control

**TL;DR:** Forward proxy = hides clients. Reverse proxy = hides servers. Both mediate traffic.

**Keywords:** Proxy server, forward proxy, reverse proxy, load balancing, caching, intermediary, Nginx

---

## TOP 51-100 MOST IMPORTANT NETWORKING QUESTIONS

---

### Q51: What are the different types of network delays?

**Answer:** The main types of network delays are propagation delay, transmission delay, processing delay, and queuing delay.

**Polished Answer:**

**Four types of network delays:**

**1. Propagation Delay:**
- Time for signal to physically travel from sender to receiver
- Depends on: Distance and transmission medium
- Formula: Propagation Delay = Distance / Signal Speed
- Example: Signal through fiber travels at ~200,000 km/s

**2. Transmission Delay:**
- Time to push all packet bits onto the wire
- Depends on: Packet size and link bandwidth
- Formula: Transmission Delay = Packet Size / Bandwidth
- Example: 1500-byte packet at 100 Mbps = 0.12 ms

**3. Processing Delay:**
- Time for router to process packet header
- Includes: Error checking, route lookup
- Depends on: Router CPU speed
- Usually very small (microseconds)

**4. Queuing Delay:**
- Time packet waits in router buffer
- Depends on: Network congestion
- Most variable component
- Can be significant during peak traffic

**Total Delay = Propagation + Transmission + Processing + Queuing**

| Delay Type | Dominant Factor | Variability |
|------------|----------------|-------------|
| Propagation | Distance | Fixed |
| Transmission | Bandwidth | Fixed |
| Processing | CPU speed | Low |
| Queuing | Congestion | High |

**TL;DR:** Total delay = propagation (distance) + transmission (bandwidth) + processing (CPU) + queuing (congestion).

**Keywords:** Network delays, propagation delay, transmission delay, processing delay, queuing delay, latency

---

### Q52: What is a ping command and what is TTL?

**Answer:** Ping checks if a system is reachable over a network. TTL (Time To Live) prevents packets from circulating forever in routing loops.

**Polished Answer:**

**Ping Command:**
- Tests network connectivity to a remote host
- Sends ICMP Echo Request packets
- Receives ICMP Echo Reply if host is reachable
- Measures Round Trip Time (RTT)
- Also reports packet loss if any

**Example output:**
```
PING google.com (172.217.166.36): 56 data bytes
64 bytes from 172.217.166.36: icmp_seq=0 ttl=115 time=14.2 ms
```

**TTL (Time To Live):**
- Counter in IP packet header
- Decremented by 1 at each router hop
- Packet discarded when TTL reaches 0
- Router sends ICMP "Time Exceeded" message

**Why TTL is needed:**
- Prevents infinite routing loops
- Ensures packets eventually expire
- Enables traceroute functionality

**Default TTL values:**
- Windows: 128
- Linux/macOS: 64
- Cisco routers: 255

**Traceroute and TTL:**
- Traceroute sends packets with TTL = 1, 2, 3...
- Each router drops packet and reveals its IP
- Maps the entire path to destination

**If ping works but HTTP doesn't:**
- Network connectivity is fine
- Issue is at higher layer (blocked port, service down, application error)

**TL;DR:** Ping = reachability test. TTL = hop counter preventing loops. Traceroute uses TTL to map paths.

**Keywords:** Ping, TTL, ICMP, Echo Request, traceroute, reachability, Time To Live

---

### Q53: How does SSL/TLS work? What happens during a TLS handshake?

**Answer:** TLS encrypts communication between client and server. During the handshake, certificates are verified and encryption keys are generated.

**Polished Answer:**

**TLS (Transport Layer Security):**
- Cryptographic protocol securing communication
- Provides: Encryption, Authentication, Integrity
- Sits between Application (HTTP) and Transport (TCP) layers

**TLS Handshake (TLS 1.2):**

**Step 1 - Client Hello:**
- Client sends supported TLS versions
- Lists supported cipher suites
- Sends random number for key generation

**Step 2 - Server Hello:**
- Server selects TLS version and cipher suite
- Sends its digital certificate (contains public key)
- Sends its random number

**Step 3 - Certificate Verification:**
- Client verifies certificate with Certificate Authority (CA)
- Checks certificate validity, domain match, revocation status

**Step 4 - Key Exchange:**
- Client generates pre-master secret
- Encrypts it with server's public key
- Server decrypts with private key
- Both derive session keys from exchanged values

**Step 5 - Finished:**
- Both send "Finished" message encrypted with session key
- Secure communication begins

**Encryption types used:**
- Asymmetric (RSA/ECDHE): During handshake for key exchange
- Symmetric (AES): For data transfer (faster)

**TLS 1.3 improvements:**
- Handshake reduced from 2 RTT to 1 RTT
- Removes obsolete cipher suites
- Better security defaults

**TL;DR:** TLS handshake = negotiate cipher, verify certificate, exchange keys. Asymmetric for handshake, symmetric for data.

**Keywords:** TLS, SSL, handshake, encryption, certificate, public key, private key, cipher suite

---

### Q54: What are the different types of computer networks?

**Answer:** Computer networks are classified based on coverage area: PAN, LAN, MAN, WAN, and GAN.

**Polished Answer:**

| Network Type | Coverage | Distance | Examples |
|--------------|----------|----------|----------|
| **PAN** | Personal space | ~10 meters | Bluetooth, USB |
| **LAN** | Building/campus | Up to km | Office, home network |
| **MAN** | City | 5-50 km | Cable TV, city Wi-Fi |
| **WAN** | Country/continent | 100s-1000s km | Internet, corporate WAN |
| **GAN** | Global | Worldwide | The Internet |

**PAN (Personal Area Network):**
- Connects personal devices
- Very short range
- Examples: Bluetooth headphones, smartphone to smartwatch

**LAN (Local Area Network):**
- Connects devices in limited area
- High speed (100 Mbps - 10 Gbps)
- Privately owned
- Technologies: Ethernet, Wi-Fi

**MAN (Metropolitan Area Network):**
- Connects LANs across a city
- Higher capacity than LAN
- Examples: ISP city infrastructure, cable TV

**WAN (Wide Area Network):**
- Connects networks across large distances
- Uses leased lines, MPLS, satellite
- Internet is the largest WAN

**GAN (Global Area Network):**
- Connects networks worldwide
- The Internet itself
- Uses undersea cables, satellites

**TL;DR:** Networks by size: PAN (personal) → LAN (building) → MAN (city) → WAN (country) → GAN (global).

**Keywords:** PAN, LAN, MAN, WAN, GAN, network types, network classification, coverage area

---

### Q55: What is IPv4 addressing and what are its classes?

**Answer:** IPv4 addresses are 32-bit addresses divided into 4 octets. Classes A, B, C, D, E define address ranges and usage.

**Polished Answer:**

**IPv4 Address Structure:**
- 32-bit address (4 octets of 8 bits each)
- Written in dotted decimal: 192.168.1.10
- Each octet: 0-255
- Total address space: ~4.3 billion

**IPv4 Classes:**

| Class | First Octet Range | Network/Host Split | Use Case |
|-------|-------------------|-------------------|----------|
| **A** | 1-126 | N.H.H.H (8/24) | Large networks |
| **B** | 128-191 | N.N.H.H (16/16) | Medium networks |
| **C** | 192-223 | N.N.N.H (24/8) | Small networks |
| **D** | 224-239 | N/A (multicast) | Multicasting |
| **E** | 240-255 | N/A (reserved) | Research |

**Class details:**

**Class A:**
- Range: 1.0.0.0 to 126.0.0.0
- Default mask: 255.0.0.0 (/8)
- Max hosts: ~16.7 million
- Example: 10.0.0.0/8 (private)

**Class B:**
- Range: 128.0.0.0 to 191.255.0.0
- Default mask: 255.255.0.0 (/16)
- Max hosts: ~65,536
- Example: 172.16.0.0/16 (private)

**Class C:**
- Range: 192.0.0.0 to 223.255.255.0
- Default mask: 255.255.255.0 (/24)
- Max hosts: 254
- Example: 192.168.0.0/24 (private)

**Class D:** Reserved for multicast (224.0.0.0 - 239.255.255.255)
**Class E:** Reserved for research (240.0.0.0 - 255.255.255.255)

**Note:** Classful addressing replaced by CIDR for efficiency.

**TL;DR:** IPv4 = 32-bit. Classes: A (large), B (medium), C (small), D (multicast), E (reserved). CIDR replaces classful.

**Keywords:** IPv4, address classes, Class A, Class B, Class C, multicast, network address

---

### Q56: What are network devices and their types?

**Answer:** Network devices are hardware components that connect computers and other devices, enabling data transmission and managing network communication.

**Polished Answer:**

**Network Devices by OSI Layer:**

| Device | OSI Layer | Function |
|--------|-----------|----------|
| **Hub** | Layer 1 (Physical) | Broadcasts to all ports |
| **Repeater** | Layer 1 (Physical) | Amplifies signals |
| **Bridge** | Layer 2 (Data Link) | Connects segments, filters by MAC |
| **Switch** | Layer 2 (Data Link) | Forwards by MAC address |
| **Router** | Layer 3 (Network) | Routes by IP address |
| **Gateway** | Layer 4-7 | Connects dissimilar networks |
| **Firewall** | Layer 3-7 | Security filtering |

**Key devices explained:**

**1. Hub:**
- Dumb broadcast device
- All traffic sent to all ports
- Creates collision domains
- Legacy device

**2. Switch:**
- Intelligent forwarding by MAC
- Separate collision domain per port
- Maintains MAC address table
- Core of modern LANs

**3. Router:**
- Connects different networks
- Routes packets by IP address
- Separates broadcast domains
- Provides NAT, firewall capabilities

**4. Gateway:**
- Connects networks using different protocols
- Translates between protocols
- Example: IoT gateway (ZigBee to IP)

**5. Bridge:**
- Connects two network segments
- Filters traffic by MAC
- Predecessor to switches

**6. Repeater:**
- Amplifies/regenerates weak signals
- Extends network distance
- Operates at physical layer

**TL;DR:** Hub (L1) → Switch (L2) → Router (L3) → Gateway (L4-7). Each device operates at different layers.

**Keywords:** Network devices, hub, switch, router, gateway, bridge, repeater, OSI layers

---

### Q57: What is the difference between synchronous and asynchronous transmission?

**Answer:** Synchronous transmission sends data as a continuous stream using a common clock. Asynchronous transmission sends data character by character using start and stop bits.

**Polished Answer:**

| Feature | Synchronous | Asynchronous |
|---------|-------------|--------------|
| **Timing** | Shared clock | Start/stop bits |
| **Data Format** | Continuous stream | Character-by-character |
| **Efficiency** | High (no extra bits) | Lower (start/stop overhead) |
| **Complexity** | Complex hardware | Simple hardware |
| **Use Cases** | Large data transfer, Ethernet | Keyboard, serial ports |

**Synchronous Transmission:**
- Sender and receiver share a common clock
- Data sent as continuous block
- No start/stop bits needed
- More efficient for large transfers
- Example: Ethernet frames, fiber optic links

**Asynchronous Transmission:**
- No shared clock
- Each character framed with start and stop bits
- Start bit (0) signals beginning
- Stop bit (1) signals end
- Extra overhead reduces efficiency
- Example: Keyboard input, RS-232 serial

**Comparison:**
- Synchronous: Faster, more efficient, complex
- Asynchronous: Slower, simpler, flexible

**TL;DR:** Synchronous = shared clock, continuous stream (efficient). Asynchronous = start/stop bits, character-based (simple).

**Keywords:** Synchronous transmission, asynchronous transmission, clock synchronization, start bits, stop bits

---

### Q58: What is the difference between Bluetooth, Wi-Fi, and ZigBee?

**Answer:** Bluetooth is short-range personal device connectivity. Wi-Fi is high-speed local networking. ZigBee is low-power IoT networking.

**Polished Answer:**

| Feature | Bluetooth | Wi-Fi | ZigBee |
|---------|-----------|-------|--------|
| **Range** | 10-100 m | 30-100 m | 10-100 m |
| **Data Rate** | 1-3 Mbps | 54 Mbps - 10 Gbps | 250 Kbps |
| **Power** | Low | High | Very Low |
| **Topology** | Piconet (star) | Star/Mesh | Mesh |
| **Use Cases** | Audio, peripherals | Internet access | IoT, sensors |
| **Frequency** | 2.4 GHz | 2.4/5/6 GHz | 2.4 GHz |
| **Standard** | IEEE 802.15.1 | IEEE 802.11 | IEEE 802.15.4 |

**Bluetooth:**
- Personal device connectivity
- Low power consumption
- Pairs devices (headphones, keyboards)
- Short-range communication
- Bluetooth 5.0: range up to 240m

**Wi-Fi:**
- High-speed local networking
- Internet access
- Infrastructure or ad-hoc mode
- Higher power consumption
- Wi-Fi 6/6E: multi-gigabit speeds

**ZigBee:**
- Low-power IoT networks
- Very low data rates
- Mesh topology (self-healing)
- Battery-powered sensors (years of life)
- Home automation, industrial IoT

**TL;DR:** Bluetooth = personal devices (short range). Wi-Fi = high-speed networking. ZigBee = low-power IoT mesh.

**Keywords:** Bluetooth, Wi-Fi, ZigBee, IEEE 802.11, IEEE 802.15, IoT, wireless networking

---

### Q59: What is a firewall and what are its types?

**Answer:** A firewall is a security system that monitors and controls network traffic based on predefined rules.

**Polished Answer:**

**Firewall Types:**

**1. Packet Filtering Firewall:**
- Examines packet headers only
- Filters by IP, port, protocol
- Fast but limited inspection
- Operates at Network/Transport layer

**2. Stateful Inspection Firewall:**
- Tracks connection state
- Maintains state table
- Understands TCP sessions
- More intelligent filtering

**3. Application Layer Firewall:**
- Deep packet inspection
- Understands application protocols
- Can block specific content/URLs
- Slowest but most secure

**4. Next-Generation Firewall (NGFW):**
- Combines multiple features
- Application awareness
- Intrusion prevention (IPS)
- Threat intelligence
- Deep packet inspection

**Firewall Deployment:**

**Hardware Firewall:**
- Dedicated appliance
- Protects entire network
- Placed at network perimeter
- Examples: Cisco ASA, Palo Alto

**Software Firewall:**
- Installed on host
- Protects individual devices
- Windows Firewall, iptables

**TL;DR:** Firewall = traffic filter. Types: Packet filtering, Stateful, Application, NGFW. Hardware or software.

**Keywords:** Firewall, packet filtering, stateful inspection, application firewall, NGFW, network security

---

### Q60: What are the advantages of fiber optics?

**Answer:** Fiber optic cables transmit data using light, providing faster, more reliable communication with minimal signal loss.

**Polished Answer:**

**Advantages of Fiber Optics:**

**1. High Bandwidth:**
- Supports up to terabits per second
- Much higher than copper (coax/twisted pair)
- Future-proof with DWDM technology

**2. Long Distance:**
- Low signal attenuation
- Signals travel kilometers without repeaters
- Single-mode fiber: 40-100 km

**3. Electromagnetic Immunity:**
- Immune to EMI (Electromagnetic Interference)
- No crosstalk between fibers
- Ideal for industrial environments

**4. Security:**
- Difficult to tap without detection
- No electromagnetic leakage
- Secure for sensitive data

**5. Reliability:**
- Not affected by temperature extremes
- Corrosion resistant
- Longer lifespan

**6. Size and Weight:**
- Thinner and lighter than copper
- Easier to install in conduits
- Saves space

**Disadvantages:**
- Higher initial cost
- Requires specialized installation
- More fragile than copper
- Harder to terminate

**TL;DR:** Fiber = higher bandwidth, longer distance, immune to EMI, more secure than copper.

**Keywords:** Fiber optics, bandwidth, EMI immunity, signal attenuation, optical cable, light transmission

---

### Q61: Can IP Multicast be load-balanced?

**Answer:** IP multicast is not load-balanced because traffic from a single source follows only one network path.

**Polished Answer:**

**Why multicast isn't load-balanced:**

**1. Single Path Delivery:**
- Multicast traffic from one source follows one distribution tree
- No path diversity for the same source-group pair

**2. Deterministic Behavior:**
- Load balancing would create multiple copies
- Receivers might receive duplicate packets
- Order and consistency matters

**3. Distribution Trees:**
- Source Tree (SPT): One path from source to each receiver
- Shared Tree (RPT): One path via Rendezvous Point

**What multicast does instead:**
- Uses efficient distribution trees
- Replicates packets at branch points
- Optimizes bandwidth (one copy per link)

**ECMP and Multicast:**
- ECMP works for unicast traffic
- Multicast doesn't use ECMP
- Multiple sources to same group may take different paths

**TL;DR:** Multicast uses single distribution tree per source. No load balancing to avoid duplicates.

**Keywords:** IP multicast, load balancing, distribution tree, ECMP, source tree, shared tree

---

### Q62: What is CGMP (Cisco Group Management Protocol)?

**Answer:** CGMP helps switches manage multicast traffic. Routers send CGMP messages to switches, reducing unnecessary multicast flooding.

**Polished Answer:**

**What is CGMP?**
A Cisco proprietary protocol that enables switches to learn which ports need multicast traffic, preventing unnecessary flooding.

**How CGMP works:**
1. Router receives IGMP join from host
2. Router sends CGMP message to switch
3. Switch learns which port needs multicast traffic
4. Switch forwards multicast only to that port

**Why CGMP was needed:**
- Switches normally flood multicast to all ports
- This wastes bandwidth
- CGMP restricts multicast to interested receivers

**CGMP vs IGMP Snooping:**
- CGMP: Router-to-switch protocol (Cisco proprietary)
- IGMP Snooping: Switch directly inspects IGMP messages
- Both accomplish same goal (multicast optimization)
- IGMP Snooping is modern standard

**TL;DR:** CGMP = Cisco protocol for multicast port learning. Reduces multicast flooding. Replaced by IGMP snooping.

**Keywords:** CGMP, multicast, IGMP, Cisco, switch, multicast traffic, port management

---

### Q63: Why do we need POP3 for email?

**Answer:** POP3 downloads emails from a mail server to a user's device, allowing offline access.

**Polished Answer:**

**POP3 (Post Office Protocol version 3):**
- Downloads emails from mail server to local device
- Emails typically deleted from server after download
- Allows offline reading
- Port 110 (or 995 with SSL/TLS)

**Why POP3 is needed:**
- Access email without Internet (after download)
- Simple protocol, widely supported
- Ideal for single-device email access
- Reduces server storage (downloads and deletes)

**POP3 vs IMAP:**

| Feature | POP3 | IMAP |
|---------|------|------|
| **Email Storage** | Local device | Server |
| **Offline Access** | Yes | Limited |
| **Multi-device** | No | Yes |
| **Server Storage** | Minimal | Required |
| **Port** | 110 | 143 |

**When to use POP3:**
- Single device access
- Limited server storage
- Need for offline access
- Simple email needs

**TL;DR:** POP3 downloads emails for offline access. Deletes from server. Good for single-device use.

**Keywords:** POP3, IMAP, email protocol, offline access, port 110, mail server, email download

---

### Q64: Define Jitter.

**Answer:** Jitter is the variation in the delay of data packets during transmission, affecting real-time applications.

**Polished Answer:**

**What is Jitter?**
The variation in packet arrival times (delay) during data transmission. It measures how consistently packets arrive.

**Understanding Jitter:**
- If all packets have 20ms delay: Zero jitter (consistent)
- If delays vary: 15ms, 25ms, 30ms, 18ms: High jitter (inconsistent)

**Why jitter matters:**
- Real-time applications need consistent timing
- Voice calls: Jitter causes choppy audio
- Video streaming: Jitter causes buffering
- Gaming: Jitter causes lag spikes

**How to reduce jitter:**
- Jitter buffers (buffer packets before playback)
- QoS (Quality of Service) prioritization
- Dedicated bandwidth for real-time traffic

**Measurement:**
- Measured in milliseconds (ms)
- Standard deviation of latency values
- Tools: ping tests, iperf, network analyzers

**Acceptable jitter levels:**
- <30 ms: Excellent for VoIP/gaming
- 30-50 ms: Acceptable
- >50 ms: Noticeable degradation

**TL;DR:** Jitter = variation in packet delay. Affects real-time apps. Reduced with buffers and QoS.

**Keywords:** Jitter, packet delay variation, latency consistency, VoIP quality, jitter buffer, QoS

---

### Q65: What is a Default Gateway and why is it important?

**Answer:** A default gateway is the router that forwards packets from a local network to other networks or the Internet.

**Polished Answer:**

**What is a Default Gateway?**
The router on a local network that serves as the exit point for traffic destined to external networks.

**How it works:**
1. Device checks if destination IP is on local subnet
2. If local: Sends directly via MAC address (ARP)
3. If external: Sends to default gateway's MAC address
4. Gateway routes packet toward destination

**Example configuration:**
- IP Address: 192.168.1.10
- Subnet Mask: 255.255.255.0 (/24)
- Default Gateway: 192.168.1.1

**Routing logic:**
- Traffic to 192.168.1.x → Direct (same subnet)
- Traffic to anything else → Default gateway
- Gateway forwards to ISP, which routes to destination

**Key points:**
- Gateway must be in same subnet as device
- Without gateway: No Internet access (local only)
- Gateway is the first hop in routing
- Multiple gateways possible for redundancy

**TL;DR:** Default gateway = router for external traffic. Essential for Internet access. First hop in routing.

**Keywords:** Default gateway, router, subnet, next hop, routing, Internet access, local network

---

### Q66: What are APIPA and its importance?

**Answer:** APIPA automatically assigns an IP in 169.254.0.0/16 range when a device cannot reach a DHCP server.

**Polished Answer:**

**APIPA (Automatic Private IP Addressing):**
- Windows feature for automatic IP assignment
- Triggered when DHCP server is unavailable
- Assigns address in 169.254.0.0/16 range
- Enables local network communication only

**How APIPA works:**
1. Device requests IP from DHCP server
2. No response received (timeout)
3. Device assigns itself a random 169.254.x.x address
4. Checks for address conflicts (ARP)
5. Uses this address for local communication

**What APIPA enables:**
- Local network communication (same subnet)
- Printers and file sharing on LAN
- Troubleshooting DHCP issues

**What APIPA doesn't provide:**
- Internet access
- Cross-subnet communication
- DNS resolution (no gateway/DNS configured)

**Identifying APIPA:**
- IP starts with 169.254.x.x
- Indicates DHCP configuration problem
- Check: DHCP server availability, cable, switch port

**TL;DR:** APIPA = self-assigned 169.254.x.x when DHCP fails. Local communication only. Signals DHCP problem.

**Keywords:** APIPA, 169.254.x.x, DHCP failure, automatic IP assignment, link-local address, troubleshooting

---

### Q67: What is the TCP Sliding Window and how does it improve throughput?

**Answer:** The TCP Sliding Window allows multiple packets to be sent before receiving acknowledgments, improving throughput.

**Polished Answer:**

**TCP Sliding Window:**
A flow control mechanism that enables the sender to transmit multiple segments before requiring acknowledgment.

**How it works:**
- Sender maintains a window of unacknowledged packets
- Window size = min(cwnd, rwnd)
- As ACKs arrive, window slides forward
- New packets can be sent as window advances

**Example:**
- Window size: 5 packets
- Sender transmits packets 1-5
- Receives ACK for packet 1
- Window slides: Can now send packet 6
- Continuous flow: Always 5 packets in transit

**Benefits:**
- Better bandwidth utilization
- Reduced idle time (not waiting for each ACK)
- Flow control (respects receiver's buffer)
- Increased throughput

**Window size considerations:**
- Too small: Underutilized bandwidth
- Too large: May cause congestion
- Optimal: Bandwidth-Delay Product (BDP)

**TL;DR:** Sliding window = send multiple packets before ACK. Window slides as ACKs arrive. Improves throughput.

**Keywords:** TCP sliding window, flow control, throughput, pipelining, window size, ACK

---

### Q68: What is Round Trip Time (RTT) and why is it important?

**Answer:** RTT is the total time for a packet to travel to destination and back. It measures network latency.

**Polished Answer:**

**What is RTT?**
Round Trip Time - the time from sending a packet until receiving its acknowledgment or response.

**Formula:** RTT = Outbound time + Return time

**How to measure:**
- `ping` command displays RTT
- `traceroute` shows per-hop RTT
- Network monitoring tools

**Importance:**
- Determines TCP timeout values
- Affects TCP window sizing (BDP = bandwidth × RTT)
- Critical for real-time applications
- Indicator of network health

**Typical RTT ranges:**
- Local network: 1-5 ms
- Same city: 5-20 ms
- Cross-country: 30-80 ms
- International: 100-300 ms
- Satellite: 500+ ms

**RTT vs Latency:**
- Latency: One-way delay
- RTT: Two-way (round trip) delay
- RTT = 2 × Latency (approximately)

**TL;DR:** RTT = time for packet + response. Measures latency. Affects TCP performance and real-time apps.

**Keywords:** RTT, Round Trip Time, latency, ping, TCP timeout, network performance

---

### Q69: What is Manchester encoding and why is it called self-clocking?

**Answer:** Manchester encoding has a signal transition in the middle of every bit, allowing the receiver to recover clock timing from the signal.

**Polished Answer:**

**Manchester Encoding:**
- Digital encoding where each bit has a mid-bit transition
- Transition direction determines bit value
- Used in early Ethernet (10BASE-T)

**How it works:**
- Bit 1: Transition from high to low
- Bit 0: Transition from low to high
- Every bit has a guaranteed transition

**Why self-clocking:**
- Mid-bit transition provides timing information
- Receiver synchronizes clock using transitions
- No separate clock signal needed
- Works even with long runs of same bit

**Manchester vs NRZ:**

| Feature | Manchester | NRZ |
|---------|-----------|-----|
| **Self-clocking** | Yes | No (sync issues) |
| **Bandwidth** | Higher (2x signal rate) | Lower (efficient) |
| **Synchronization** | Excellent | Poor with long runs |
| **Use Case** | Early Ethernet | Modern high-speed links |

**Advantages:**
- Guaranteed transitions for sync
- Error detection capability
- DC balanced (no DC component)

**Disadvantages:**
- Requires double bandwidth
- Inefficient for high speeds

**TL;DR:** Manchester = mid-bit transition every bit. Self-clocking = receiver extracts clock from signal.

**Keywords:** Manchester encoding, self-clocking, NRZ, bit transition, clock recovery, Ethernet

---

### Q70: What are TCP Sequence Numbers and Acknowledgment Numbers?

**Answer:** Sequence numbers identify byte positions in transmitted data. Acknowledgment numbers indicate the next expected byte.

**Polished Answer:**

**Sequence Numbers:**
- Identify the position of each byte in the data stream
- Initial Sequence Number (ISN) randomly chosen
- Incremented by bytes sent
- Enables ordering and reassembly

**Acknowledgment Numbers:**
- Indicate the next byte the receiver expects
- Cumulative: Acknowledges all bytes up to that point
- Sent in ACK packets

**How they work:**
1. Sender sends 1000 bytes starting at sequence 1000
2. Receiver acknowledges with ACK 2001 (1000 + 1000 + 1)
3. Sender knows all bytes up to 2000 were received
4. Next data sent starts at sequence 2001

**How they prevent duplicates:**
- Receiver tracks received sequence numbers
- Duplicate packets have same sequence numbers
- TCP discards packets with already-received sequences

**Importance:**
- Reliable ordered delivery
- Data integrity
- Retransmission detection
- Duplicate detection

**TL;DR:** Sequence = byte position in stream. ACK = next expected byte. Together ensure reliable, ordered delivery.

**Keywords:** TCP sequence numbers, acknowledgment numbers, ISN, byte ordering, reliable delivery, duplicate detection

---

### Q71: Explain TCP Retransmission and RTO.

**Answer:** TCP retransmission resends lost packets. RTO is the timeout before retransmission occurs.

**Polished Answer:**

**TCP Retransmission:**
Process of resending packets that were lost, corrupted, or unacknowledged within the timeout period.

**RTO (Retransmission Timeout):**
- Time sender waits for ACK before retransmitting
- Dynamically calculated based on RTT measurements
- Initial RTO: Typically 1 second
- Doubles on each retransmission (exponential backoff)

**How RTO is calculated:**
1. Measure RTT samples
2. Calculate smoothed RTT (SRTT)
3. Calculate RTT variance (RTTVAR)
4. RTO = SRTT + 4 × RTTVAR

**Fast Retransmit:**
- Three duplicate ACKs trigger immediate retransmission
- No waiting for RTO
- Indicates packet loss (later packets arriving)

**Fast Recovery:**
- After fast retransmit, don't reset to Slow Start
- Reduce cwnd to half
- Continue in Congestion Avoidance

**TL;DR:** RTO = timeout before retransmission. Fast Retransmit = retransmit on 3 duplicate ACKs. Exponential backoff on timeouts.

**Keywords:** TCP retransmission, RTO, Fast Retransmit, Fast Recovery, duplicate ACKs, timeout

---

### Q72: Compare cwnd and rwnd.

**Answer:** cwnd (congestion window) is controlled by sender based on network congestion. rwnd (receive window) is advertised by receiver based on buffer space.

**Polished Answer:**

| Feature | cwnd | rwnd |
|---------|------|------|
| **Full Form** | Congestion Window | Receive Window |
| **Controlled By** | Sender | Receiver |
| **Based On** | Network congestion | Buffer capacity |
| **Purpose** | Prevent network overload | Prevent receiver overload |
| **Adjustment** | Dynamic (TCP algorithms) | Static (buffer size) |
| **Range** | 1 MSS to large values | Fixed maximum |

**cwnd (Congestion Window):**
- Sender-side control
- Adjusts based on network signals
- Increases: Slow Start, Congestion Avoidance
- Decreases: On packet loss
- Goal: Avoid network congestion

**rwnd (Receive Window):**
- Receiver-advertised value
- Indicates available buffer space
- Sent in TCP header (Window field)
- Prevents sender from overwhelming receiver
- Changes as buffer fills/empties

**Sending Limit:** min(cwnd, rwnd)

**Example:**
- cwnd = 100 KB (network allows)
- rwnd = 64 KB (receiver buffer limit)
- Effective sending window = 64 KB (limited by receiver)

**TL;DR:** cwnd = network protection (sender). rwnd = receiver protection (buffer). Send limit = min of both.

**Keywords:** cwnd, rwnd, congestion window, receive window, TCP flow control, congestion control

---

### Q73: What is ECN (Explicit Congestion Notification)?

**Answer:** ECN allows routers to mark packets instead of dropping them when congestion is detected, improving network performance.

**Polished Answer:**

**What is ECN?**
A TCP congestion control mechanism where routers mark packets experiencing congestion instead of dropping them, allowing the sender to reduce rate before packet loss occurs.

**How ECN works:**
1. Router detects congestion (buffer building up)
2. Router marks packet with ECN flag instead of dropping
3. Receiver receives marked packet
4. Receiver notifies sender (via ACK with ECE flag)
5. Sender reduces transmission rate

**Benefits:**
- Reduces unnecessary packet drops
- Faster congestion detection (before loss)
- Improves throughput
- Reduces retransmissions
- Lower latency

**ECN fields:**
- Two bits in IP header (ECT, CE)
- Two flags in TCP header (ECE, CWR)

**Requirements:**
- All routers in path must support ECN
- Both endpoints must support ECN
- Negotiated during TCP handshake

**TL;DR:** ECN = mark instead of drop. Faster congestion detection. Improves throughput and reduces retransmissions.

**Keywords:** ECN, Explicit Congestion Notification, congestion detection, packet marking, TCP optimization

---

### Q74: Compare Stop-and-Wait, Go-Back-N, and Selective Repeat.

**Answer:** These are ARQ protocols for reliable data transfer with different efficiency levels.

**Polished Answer:**

| Protocol | Efficiency | Window Size | Retransmission |
|----------|-----------|-------------|----------------|
| **Stop-and-Wait** | Low | 1 packet | Single packet |
| **Go-Back-N** | Medium | N packets | All from lost packet |
| **Selective Repeat** | High | N packets | Only lost packets |

**Stop-and-Wait:**
- Send one packet, wait for ACK
- Very simple but inefficient
- Network idle during waiting
- Throughput limited by RTT

**Go-Back-N:**
- Multiple packets in flight (window)
- On loss: Retransmit all packets from lost one
- Wastes bandwidth with unnecessary retransmissions
- Cumulative ACKs only

**Selective Repeat:**
- Multiple packets in flight (window)
- On loss: Retransmit only lost packets
- Individual ACKs for each packet
- Most efficient use of bandwidth

**Window size constraints:**
- Stop-and-Wait: Always 1
- Go-Back-N: Up to 2^n - 1
- Selective Repeat: Up to 2^(n-1)

**TL;DR:** Stop-and-Wait = 1 packet, inefficient. Go-Back-N = retransmit all from loss. Selective Repeat = retransmit only lost.

**Keywords:** ARQ, Stop-and-Wait, Go-Back-N, Selective Repeat, reliable data transfer, window size

---

### Q75: Why must Selective Repeat window size be at most half the sequence number space?

**Answer:** Window size must be ≤ half sequence space to prevent ambiguity between new packets and old delayed packets.

**Polished Answer:**

**The Problem:**
- Sequence numbers are reused after wrapping around
- Old delayed packets might have same sequence as new packets
- Receiver can't distinguish them if window too large

**Example with sequence space = 8 (0-7):**

**Window size = 4 (valid):**
- Sender uses sequences 0-3
- Old delayed packet from sequence 0
- Receiver knows 0-3 are in current window
- No ambiguity

**Window size = 7 (invalid):**
- Sender uses sequences 0-6
- Old delayed packet with sequence 0
- Receiver might think it's new (sequence wrapped)
- Ambiguity: Is this old or new packet?

**Why half works:**
- With window ≤ 2^(n-1), receiver can distinguish
- New packets have sequence in current window
- Old packets have sequence outside current window
- No wrap-around ambiguity

**TL;DR:** Window ≤ half sequence space prevents old/new packet confusion with sequence number reuse.

**Keywords:** Selective Repeat, window size, sequence numbers, packet ambiguity, ARQ protocol

---

### Q76: What is the Bandwidth-Delay Product and how does pipelining improve performance?

**Answer:** BDP = Bandwidth × RTT. Pipelining allows multiple packets in transit, improving bandwidth utilization.

**Polished Answer:**

**Bandwidth-Delay Product (BDP):**
- Amount of data that can be in transit before ACK is received
- Formula: BDP = Bandwidth × RTT
- Determines optimal window size

**Example:**
- 100 Mbps bandwidth, 50 ms RTT
- BDP = 100,000,000 × 0.05 = 5,000,000 bits = 625 KB
- Optimal window size: 625 KB for full utilization

**Pipelining:**
- Sending multiple packets without waiting for ACKs
- More packets in transit = better utilization
- Reduces idle time

**How pipelining helps:**
- Without pipelining: Send → Wait for ACK → Send (idle time)
- With pipelining: Continuous transmission
- Bandwidth utilized during RTT

**Performance:**
- Low BDP: Little data in transit (LAN)
- High BDP: Much data in transit (long-distance)
- Window should match BDP for maximum throughput

**TL;DR:** BDP = data in transit capacity. Pipelining keeps bandwidth utilized by sending multiple packets.

**Keywords:** BDP, Bandwidth-Delay Product, pipelining, window size, throughput, bandwidth utilization

---

### Q77: Why does RIP suffer from count-to-infinity problem?

**Answer:** RIP routers share distance vectors, and when a route fails, routers can create routing loops with continuously increasing hop counts.

**Polished Answer:**

**Count-to-Infinity Problem:**
When a route becomes unreachable, RIP routers may take a long time to converge, with hop counts incrementing until reaching infinity (16 for RIP).

**How it happens:**
1. Router A loses connection to network X
2. Router A tells Router B: "X is unreachable" (hop count 16)
3. But before B updates, B tells A: "I can reach X in 2 hops"
4. A updates: "X is 3 hops away" (incorrect)
5. They exchange updates, incrementing hop count
6. Loop continues until hop count reaches 16

**Solutions:**

**1. Split Horizon:**
- Don't advertise route back on the interface it was learned from
- Prevents basic loops
- Example: B learned X from A, so B won't advertise X to A

**2. Route Poisoning:**
- When route fails, immediately advertise it with metric 16
- Neighbors quickly learn route is unreachable
- Prevents slow convergence

**3. Poison Reverse:**
- Combination of split horizon and route poisoning
- Advertise failed route with metric 16 back on learning interface

**4. Hold-Down Timers:**
- Wait before accepting new route information
- Gives network time to stabilize
- Suppresses flapping routes

**TL;DR:** Count-to-infinity = routing loop with incrementing hops. Solved by Split Horizon, Route Poisoning, Hold-down timers.

**Keywords:** RIP, count-to-infinity, Split Horizon, Route Poisoning, routing loops, hold-down timer

---

### Q78: Why does OSPF converge faster than RIP?

**Answer:** OSPF uses Link-State routing with triggered updates and Dijkstra's algorithm, converging faster than RIP's periodic distance vector updates.

**Polished Answer:**

**Why OSPF converges faster:**

**1. Triggered Updates:**
- OSPF: Immediate updates on topology change
- RIP: Periodic updates (every 30 seconds) + triggered

**2. Complete Network Knowledge:**
- OSPF: Each router has full topology map
- RIP: Only knows neighbors' routes (routing by rumor)

**3. Dijkstra Algorithm:**
- OSPF: Computes shortest paths directly
- RIP: Bellman-Ford algorithm with slower propagation

**4. No Count-to-Infinity:**
- OSPF: Link-state avoids loops inherently
- RIP: Suffers from count-to-infinity problem

**5. Hierarchical Design:**
- OSPF: Areas reduce convergence scope
- RIP: Flat topology, full network convergence needed

**Dijkstra Algorithm in OSPF:**
1. Each router has Link-State Database (LSDB)
2. Runs SPF (Shortest Path First) algorithm
3. Calculates shortest path to all destinations
4. Builds routing table from results

**TL;DR:** OSPF = triggered updates + full topology + Dijkstra = fast convergence. RIP = periodic + partial knowledge = slow.

**Keywords:** OSPF, RIP, convergence, Dijkstra algorithm, Link-State, triggered updates, SPF

---

### Q79: What is Power over Ethernet (PoE)?

**Answer:** PoE delivers both data and electrical power over a single Ethernet cable, simplifying installation.

**Polished Answer:**

**What is PoE?**
Technology enabling Ethernet cables (Cat5e/6) to carry electrical power along with data, powering devices without separate power supplies.

**PoE Standards:**

| Standard | IEEE | Max Power | Applications |
|----------|------|-----------|--------------|
| PoE | 802.3af | 15.4W | IP phones, basic cameras |
| PoE+ | 802.3at | 30W | PTZ cameras, advanced APs |
| PoE++ | 802.3bt | 60-100W | LED lighting, laptops |

**Components:**
- **PSE (Power Sourcing Equipment):** Switch or injector providing power
- **PD (Powered Device):** Device receiving power (IP phone, camera)

**Benefits:**
- Single cable for power and data
- No electrical outlet needed at device
- Centralized power management
- Easier installation and relocation
- Cost savings on cabling

**Common PoE devices:**
- IP phones
- Wireless access points
- IP cameras
- IoT sensors
- Network switches

**TL;DR:** PoE = power + data over Ethernet. Standards: PoE, PoE+, PoE++. Powers IP phones, cameras, APs.

**Keywords:** PoE, Power over Ethernet, 802.3af, 802.3at, PSE, PD, Ethernet power

---

### Q80: What is DNS and how does DNS resolution work?

**Answer:** DNS translates domain names to IP addresses through a hierarchical system of servers.

**Polished Answer:**

**What is DNS?**
Domain Name System - a distributed database that maps human-readable domain names to IP addresses.

**DNS Hierarchy:**
```
Root Servers (.)
├── .com (TLD)
│   ├── google.com (Authoritative)
│   └── facebook.com (Authoritative)
├── .org (TLD)
└── .net (TLD)
```

**DNS Resolution Process:**
1. Browser checks local cache
2. Checks OS cache
3. Query to ISP DNS resolver
4. Resolver queries Root servers → TLD servers → Authoritative servers
5. IP address returned and cached

**DNS Record Types:**
- **A Record:** Domain to IPv4 address
- **AAAA Record:** Domain to IPv6 address
- **CNAME:** Alias to another domain
- **MX:** Mail server
- **TXT:** Text information (SPF, verification)

**Port:** 53 (TCP and UDP)

**TL;DR:** DNS = phonebook of Internet. Resolves domains → IPs. Hierarchical (Root → TLD → Authoritative).

**Keywords:** DNS, Domain Name System, DNS resolution, authoritative server, TLD, root server, A record

---

### Q81: What is SMTP and how does email work?

**Answer:** SMTP is the protocol for sending email between servers. It operates on port 25.

**Polished Answer:**

**SMTP (Simple Mail Transfer Protocol):**
- Protocol for sending email from client to server and between servers
- Port 25 (or 587 for submission, 465 for SMTPS)
- Text-based protocol

**Email flow:**

1. **Sending:**
   - Email client (MUA) sends to SMTP server (MSA)
   - Uses SMTP submission (port 587)

2. **Server to Server:**
   - Sender's SMTP server forwards to recipient's SMTP server
   - Uses DNS MX records to find recipient's server

3. **Receiving:**
   - Recipient's server stores email
   - Recipient retrieves via POP3 or IMAP

**SMTP commands:**
- HELO/EHLO: Identify client
- MAIL FROM: Sender address
- RCPT TO: Recipient address
- DATA: Message content
- QUIT: End session

**Related protocols:**
- POP3 (port 110): Download emails
- IMAP (port 143): Access emails on server
- MIME: Email formatting

**TL;DR:** SMTP = email sending protocol. Port 25/587. Works with POP3/IMAP for full email functionality.

**Keywords:** SMTP, email protocol, port 25, POP3, IMAP, email sending, MTA

---

### Q82: What is the difference between flow control and congestion control?

**Answer:** Flow control protects the receiver from overload. Congestion control protects the network from overload.

**Polished Answer:**

**Flow Control:**
- **Purpose:** Prevent sender from overwhelming receiver
- **Mechanism:** Receive Window (rwnd)
- **Control:** Receiver advertises available buffer
- **Scope:** Point-to-point (end-to-end)
- **Example:** Fast sender, slow receiver with small buffer

**Congestion Control:**
- **Purpose:** Prevent network from becoming congested
- **Mechanism:** Congestion Window (cwnd)
- **Control:** Sender detects network conditions
- **Scope:** Network-wide
- **Example:** Multiple senders saturating a bottleneck link

**Key differences:**

| Aspect | Flow Control | Congestion Control |
|--------|-------------|-------------------|
| **What it protects** | Receiver buffer | Network capacity |
| **Window** | rwnd (receiver set) | cwnd (sender calculated) |
| **Signals** | Window advertisement | Packet loss, ECN |
| **Response** | Adjust sending rate | Reduce cwnd |

**Sending window = min(cwnd, rwnd)**

**TL;DR:** Flow control = receiver protection (rwnd). Congestion control = network protection (cwnd). Both limit sending.

**Keywords:** Flow control, congestion control, rwnd, cwnd, TCP, network protection, receiver buffer

---

### Q83: What is IPv6 Neighbor Discovery and how does it replace ARP?

**Answer:** NDP is IPv6's mechanism for discovering neighbors, resolving addresses, and finding routers, replacing ARP from IPv4.

**Polished Answer:**

**Neighbor Discovery Protocol (NDP):**
- IPv6 protocol using ICMPv6
- Replaces ARP, RARP, and Router Discovery from IPv4
- Operates at Link Layer

**NDP Functions:**

**1. Address Resolution:**
- Maps IPv6 addresses to MAC addresses
- Replaces ARP
- Uses Neighbor Solicitation/Advertisement messages

**2. Router Discovery:**
- Hosts discover routers on the network
- Router Advertisement messages
- Replaces ICMP Router Discovery

**3. Prefix Discovery:**
- Hosts learn network prefix
- Enables Stateless Address Auto-configuration (SLAAC)

**4. Duplicate Address Detection (DAD):**
- Ensures IPv6 address uniqueness
- Before using an address, check if already in use

**5. Neighbor Unreachability Detection:**
- Determines if neighbors are still reachable
- Detects failed connections

**NDP vs ARP:**
- NDP: Uses ICMPv6, multicast
- ARP: Uses Ethernet broadcasts
- NDP: More features (router discovery, DAD, SLAAC)

**TL;DR:** NDP = IPv6 neighbor discovery. Replaces ARP. Uses ICMPv6. Enables SLAAC and DAD.

**Keywords:** NDP, IPv6, Neighbor Discovery, ARP replacement, ICMPv6, SLAAC, address resolution

---

### Q84: Compare CIDR and VLSM.

**Answer:** CIDR combines networks for route aggregation. VLSM divides networks into different-sized subnets for efficient addressing.

**Polished Answer:**

| Feature | CIDR | VLSM |
|---------|------|------|
| **Purpose** | Route aggregation | Efficient subnetting |
| **Direction** | Combine networks | Divide network |
| **Use Case** | Internet routing | Internal network design |
| **Result** | Fewer routing entries | Better IP utilization |

**CIDR (Classless Inter-Domain Routing):**
- Combines multiple network prefixes into one
- Reduces routing table size
- Used on the Internet
- Example: 192.168.0.0/24 + 192.168.1.0/24 = 192.168.0.0/23

**VLSM (Variable Length Subnet Mask):**
- Divides network into subnets of different sizes
- Matches subnet size to requirement
- Efficient IP usage
- Example: /26 for 60 hosts, /28 for 10 hosts

**How they work together:**
- VLSM: Subnet internally based on needs
- CIDR: Aggregate routes for external advertisement
- Both use classless addressing concepts

**TL;DR:** CIDR = aggregate routes (Internet). VLSM = variable subnets (internal). Both improve efficiency.

**Keywords:** CIDR, VLSM, route aggregation, subnetting, IP addressing, network design, routing table

---

### Q85: What is a collision domain and how do modern networks eliminate collisions?

**Answer:** A collision domain is a network segment where devices compete for transmission, causing collisions. Modern switched networks eliminate collisions with full-duplex.

**Polished Answer:**

**Collision Domain:**
- Network segment where devices share transmission medium
- Simultaneous transmissions cause signal collision
- Collision corrupts data, requiring retransmission

**How collisions happen:**
- Half-duplex communication
- Two devices transmit at same time
- Signals interfere on shared medium

**Modern solutions:**

**1. Switches:**
- Each port = separate collision domain
- Dedicated bandwidth per port
- No sharing of medium

**2. Full-Duplex:**
- Simultaneous send and receive
- Separate channels for each direction
- No possibility of collision

**3. CSMA/CD (Legacy):**
- Carrier Sense Multiple Access with Collision Detection
- Used in shared Ethernet (hubs)
- Detect collision, wait random time, retry
- Obsolete in modern networks

**Devices and collision domains:**
- Hub: One collision domain (all ports)
- Switch: Separate domain per port
- Router: Separate domain per interface

**TL;DR:** Collision = devices competing for medium. Switches + full-duplex eliminate collisions in modern networks.

**Keywords:** Collision domain, CSMA/CD, full-duplex, switch, hub, Ethernet, network collision

---

### Q86: What is STP and what is the role of the Root Bridge?

**Answer:** STP prevents switching loops. The Root Bridge serves as the reference point for path selection.

**Polished Answer:**

**Spanning Tree Protocol (STP):**
- IEEE 802.1D protocol
- Prevents Layer 2 switching loops
- Creates loop-free topology
- Maintains redundant links for failover

**Root Bridge:**
- Central reference point in STP topology
- All path calculations originate from Root Bridge
- Elected based on lowest Bridge ID

**Bridge ID:**
- Priority (2 bytes) + MAC Address (6 bytes)
- Default priority: 32768
- Lower Bridge ID wins election

**Port Roles:**
- **Root Port:** Best path to Root Bridge (on non-root switches)
- **Designated Port:** Forwarding port on each segment
- **Blocked Port:** Redundant ports prevented from forwarding

**How STP works:**
1. Elect Root Bridge (lowest Bridge ID)
2. Each switch determines best path to Root
3. Block redundant paths
4. Unblock if active path fails

**STP Port States:**
- Blocking (20 sec) → Listening (15 sec) → Learning (15 sec) → Forwarding

**TL;DR:** STP prevents loops. Root Bridge = reference point (lowest Bridge ID). Blocked ports = redundancy.

**Keywords:** STP, Root Bridge, Bridge ID, BPDU, switching loop, port states, IEEE 802.1D

---

### Q87: What is a Layer 3 switch and how does it differ from a router?

**Answer:** A Layer 3 switch performs routing in hardware for high-speed LAN routing. A router offers advanced features for WAN connectivity.

**Polished Answer:**

| Feature | Layer 3 Switch | Router |
|---------|---------------|--------|
| **Routing** | Hardware (ASIC) | Software (CPU) |
| **Speed** | Very fast | Slower |
| **WAN Support** | Limited | Full (serial, MPLS) |
| **Protocols** | Basic (OSPF, RIP) | Full (BGP, MPLS, VPN) |
| **NAT** | Limited | Full support |
| **Use Case** | LAN routing, VLAN routing | WAN, Internet edge |

**Layer 3 Switch:**
- Combines switching and routing
- Routes at wire speed (hardware)
- Ideal for inter-VLAN routing
- Limited WAN features
- Used in campus/data center networks

**Router:**
- Software-based routing
- Advanced features: BGP, MPLS, VPN, NAT
- WAN connectivity (serial, fiber, LTE)
- More flexible but slower
- Used at network edge

**When to choose:**
- **L3 Switch:** High-speed internal routing, inter-VLAN
- **Router:** Internet connectivity, WAN, advanced policies

**TL;DR:** L3 switch = hardware routing for LAN. Router = software routing for WAN/Internet. Choose based on use case.

**Keywords:** Layer 3 switch, router, hardware routing, software routing, inter-VLAN, WAN, ASIC

---

### Q88: What is DHCP and how does the DORA process work?

**Answer:** DHCP automatically assigns IP addresses. DORA stands for Discover, Offer, Request, Acknowledge.

**Polished Answer:**

**DHCP (Dynamic Host Configuration Protocol):**
- Automates IP address assignment
- Provides: IP, subnet mask, gateway, DNS
- Eliminates manual configuration
- Port: 67 (server), 68 (client)

**DORA Process:**

**D - Discover:**
- Client broadcasts DHCP Discover
- Seeks available DHCP servers
- Sent to 255.255.255.255

**O - Offer:**
- Server responds with DHCP Offer
- Contains proposed IP and configuration
- Client may receive multiple offers

**R - Request:**
- Client broadcasts DHCP Request
- Selects one offer, requests that IP
- Broadcast (informs all servers)

**A - Acknowledge:**
- Server confirms with DHCP ACK
- IP officially assigned
- Client configures network settings

**Lease and Renewal:**
- IP assigned for lease duration
- Renew at 50% lease time
- Rebinds at 87.5%
- Release at 100% (if not renewed)

**TL;DR:** DHCP = automatic IP assignment. DORA = Discover, Offer, Request, Acknowledge.

**Keywords:** DHCP, DORA, IP assignment, lease, Discover, Offer, Request, Acknowledge

---

### Q89: What is DNS cache poisoning?

**Answer:** DNS cache poisoning injects fake DNS records into cache, redirecting users to malicious websites.

**Polished Answer:**

**What is DNS Cache Poisoning?**
Attack where fake DNS records are inserted into a DNS resolver's cache, causing users to be directed to malicious IPs.

**How it works:**
1. Attacker sends forged DNS responses to resolver
2. Resolver caches fake records without verification
3. Users querying resolver get incorrect IPs
4. Users redirected to phishing/malware sites

**Example:**
- User visits bank.com
- Cache has been poisoned
- Returns attacker's IP instead of bank's IP
- User sees fake bank site (credential theft)

**Prevention:**

**1. DNSSEC:**
- Digitally signs DNS records
- Resolvers verify signatures
- Rejects unsigned/fake records

**2. Transaction ID Randomization:**
- Randomize query IDs
- Harder for attackers to match responses

**3. Source Port Randomization:**
- Random source ports for queries
- Increases attacker difficulty

**4. DNSSEC Validation:**
- Resolvers validate signatures
- Reject tampered records

**TL;DR:** DNS poisoning = fake records in cache. Prevent with DNSSEC, ID/port randomization, validation.

**Keywords:** DNS cache poisoning, DNSSEC, DNS spoofing, cache attack, transaction ID, source port

---

### Q90: What is Explicit Congestion Notification (ECN)?

**Answer:** ECN marks packets instead of dropping them during congestion, enabling faster congestion detection and reduced retransmissions.

**Polished Answer:**

**What is ECN?**
A TCP/IP extension where routers mark packets experiencing congestion rather than dropping them, allowing senders to reduce rate before packet loss.

**How ECN works:**
1. Router detects early congestion (buffer threshold)
2. Marks packet with ECN flag (CE bit)
3. Receiver gets marked packet
4. Receiver notifies sender via ACK (ECE flag)
5. Sender reduces transmission rate
6. Sender confirms with CWR flag

**Benefits:**
- **Reduced Packet Loss:** Mark instead of drop
- **Faster Response:** Detect before actual loss
- **Improved Throughput:** Fewer retransmissions
- **Lower Latency:** No timeout retransmissions

**ECN Flags:**
- **ECT (ECN Capable Transport):** Indicates support
- **CE (Congestion Experienced):** Router mark
- **ECE (ECN Echo):** Receiver notification
- **CWR (Congestion Window Reduced):** Sender confirmation

**Requirements:**
- All routers support ECN
- Both endpoints support ECN
- Negotiated in TCP handshake

**TL;DR:** ECN = mark packets for congestion instead of dropping. Faster response, fewer retransmissions, better throughput.

**Keywords:** ECN, Explicit Congestion Notification, congestion marking, ECE, CWR, TCP congestion control

---

### Q91: What is route flapping?

**Answer:** Route flapping occurs when a route repeatedly changes between available and unavailable states, causing network instability.

**Polished Answer:**

**What is Route Flapping?**
A condition where a network route alternates between up and down states rapidly, causing routing instability and excessive updates.

**Causes:**
- Unstable links (flapping connections)
- Hardware failures
- Misconfiguration
- Interference (wireless)
- Power fluctuations

**Effects:**
- Excessive routing updates
- Increased CPU utilization
- Network instability
- Potential routing loops
- Degraded performance

**Mitigation techniques:**

**1. BGP Route Dampening:**
- Tracks flapping frequency
- Suppresses routes that flap repeatedly
- Restores after stability period

**2. Hold-Down Timers:**
- Wait before accepting route changes
- Prevents premature updates
- Allows network stabilization

**3. OSPF SPF Throttling:**
- Limits route calculation frequency
- Prevents excessive CPU usage
- Delays SPF after rapid changes

**4. Update Throttling:**
- Limits LSA flooding frequency
- Prevents network overload

**TL;DR:** Route flapping = unstable route. Caused by link/hardware issues. Mitigated with dampening and throttling.

**Keywords:** Route flapping, BGP dampening, hold-down timer, SPF throttling, routing instability, network stability

---

### Q92: What is a Default Route?

**Answer:** A default route is the route used when no specific route matches the destination in the routing table.

**Polished Answer:**

**What is a Default Route?**
A routing table entry (0.0.0.0/0 in IPv4, ::/0 in IPv6) that specifies where to send packets when no more specific route matches.

**Purpose:**
- Catch-all route for unmatched destinations
- Usually points to upstream router/ISP
- Essential for Internet connectivity
- Reduces routing table size

**Example:**
- Router has routes for: 192.168.1.0/24, 10.0.0.0/8
- Default route: 0.0.0.0/0 via ISP router
- Traffic to 172.217.x.x (Google) uses default route

**How it works:**
1. Router receives packet for destination X
2. Checks routing table for match
3. Longest prefix match wins
4. If no match: Uses default route
5. If no default: Drops packet (destination unreachable)

**Configuration (Cisco):**
```
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

**TL;DR:** Default route = catch-all for unmatched destinations. Routes to ISP for Internet access. Essential for WAN.

**Keywords:** Default route, 0.0.0.0/0, routing table, longest prefix match, ISP, gateway, catch-all route

---

### Q93: What is the difference between Store-and-Forward and Cut-Through switching?

**Answer:** Store-and-Forward receives entire frame and checks CRC before forwarding. Cut-Through starts forwarding after reading destination MAC.

**Polished Answer:**

**Store-and-Forward:**
- Receives complete frame
- Validates via CRC (Cyclic Redundancy Check)
- Discards corrupted frames
- Forwards only valid frames
- Higher latency (waits for full frame)
- More reliable (error detection)

**Cut-Through:**
- Reads only destination MAC address
- Begins forwarding immediately
- Low latency (no full-frame wait)
- May forward corrupted frames
- Doesn't check CRC
- Faster but less reliable

**Comparison:**

| Feature | Store-and-Forward | Cut-Through |
|---------|-------------------|-------------|
| **Latency** | Higher | Lower |
| **Error Detection** | Yes (CRC) | No |
| **Reliability** | High | Lower |
| **Speed** | Slower | Faster |
| **Use Case** | Enterprise, quality-critical | Low-latency, speed-critical |

**Fragment-Free (variant):**
- Reads first 64 bytes (min frame size)
- Filters collision fragments
- Middle ground

**TL;DR:** Store-and-Forward = check then forward (reliable). Cut-Through = forward immediately (fast).

**Keywords:** Store-and-Forward, Cut-Through, switching, CRC, latency, frame forwarding, error detection

---

### Q94: What is a Broadcast Domain?

**Answer:** A broadcast domain is a set of devices that receive the same broadcast frame.

**Polished Answer:**

**What is a Broadcast Domain?**
A logical network segment where broadcast traffic is forwarded to all devices. All devices in the domain receive broadcast frames.

**Devices that separate broadcast domains:**
- Routers
- Layer 3 switches
- VLANs (on switches)
- Firewalls

**Devices that extend broadcast domain:**
- Switches (without VLANs)
- Hubs
- Bridges

**Example:**
- 24-port switch (no VLANs) = 1 broadcast domain
- All 24 ports receive broadcast traffic
- Adding VLANs creates multiple broadcast domains
- Router connecting to switch separates domains

**Why broadcast domains matter:**
- Large domains = more broadcast overhead
- Broadcast storms can degrade performance
- Proper segmentation improves performance
- Security isolation between domains

**Broadcast vs Collision domain:**
- Broadcast: Router/VLAN separated
- Collision: Switch separated
- Switch: Many collision domains, one broadcast domain (no VLANs)

**TL;DR:** Broadcast domain = devices receiving broadcasts. Separated by routers/VLANs. Extended by switches.

**Keywords:** Broadcast domain, VLAN, router, switch, broadcast traffic, network segmentation

---

### Q95: What is Path MTU Discovery?

**Answer:** Path MTU Discovery finds the smallest MTU along a network path to prevent fragmentation.

**Polished Answer:**

**What is PMTUD?**
Path MTU Discovery - a technique to determine the maximum packet size that can transit a path without fragmentation.

**How it works:**
1. Sender sends packets with DF (Don't Fragment) flag
2. If router's MTU smaller than packet, router drops
3. Router sends ICMP "Fragmentation Needed" message
4. ICMP message includes router's MTU
5. Sender reduces packet size
6. Process repeats until packets reach destination

**Example:**
- Sender MTU: 1500 bytes
- Path includes link with 1400 MTU
- 1500-byte packet dropped
- ICMP informs sender to use 1400
- Sender adjusts and retransmits

**Why PMTUD matters:**
- Prevents fragmentation overhead
- Avoids performance degradation
- Critical for VPNs (encapsulation reduces MTU)
- Essential for IPv6 (no router fragmentation)

**Common MTU values:**
- Ethernet: 1500
- PPPoE: 1492
- VPN (IPsec): ~1400
- IPv6 minimum: 1280
- Jumbo frames: 9000

**TL;DR:** PMTUD finds smallest MTU along path. Uses ICMP to adjust. Prevents fragmentation.

**Keywords:** PMTUD, Path MTU Discovery, MTU, fragmentation, DF flag, ICMP, packet size

---

### Q96: What is a Gateway and how does it differ from a router?

**Answer:** A gateway connects dissimilar networks and translates protocols. A router routes packets between similar networks using IP.

**Polished Answer:**

| Feature | Gateway | Router |
|---------|---------|--------|
| **Function** | Protocol translation | Packet routing |
| **Networks** | Dissimilar protocols | Similar (IP) |
| **OSI Layer** | Any layer (4-7) | Network (Layer 3) |
| **Processing** | More complex | Simpler |
| **Examples** | IoT gateway, VoIP gateway | IP router |

**Gateway:**
- Connects networks with different protocols
- Translates between protocols
- Example: ZigBee to IP gateway (IoT)
- Example: VoIP to PSTN gateway
- Can operate at any OSI layer

**Router:**
- Routes packets between IP networks
- Uses routing tables
- Works at Network Layer
- No protocol translation (both sides IP)

**Note:** The term "default gateway" commonly refers to a router on the local network that forwards traffic to external networks.

**TL;DR:** Gateway = protocol translation between dissimilar networks. Router = IP packet routing between similar networks.

**Keywords:** Gateway, router, protocol translation, OSI layers, default gateway, dissimilar networks

---

### Q97: What is Ethernet and how does CSMA/CD work?

**Answer:** Ethernet is a LAN technology. CSMA/CD is a media access control method used in legacy Ethernet to handle collisions.

**Polished Answer:**

**Ethernet:**
- IEEE 802.3 standard for LAN
- Most widely used networking technology
- Speeds: 10 Mbps to 400 Gbps
- Uses frames with MAC addressing
- Modern: Switched, full-duplex

**CSMA/CD (Carrier Sense Multiple Access with Collision Detection):**
- Used in legacy shared Ethernet (hubs)
- Half-duplex operation

**How CSMA/CD works:**
1. **Carrier Sense:** Listen before transmitting
2. **Multiple Access:** Multiple devices share medium
3. **Collision Detection:** Detect collisions during transmission

**Collision handling:**
1. Detect collision
2. Send jam signal (inform others)
3. Wait random time (exponential backoff)
4. Retry transmission

**Why obsolete:**
- Modern networks use switches
- Full-duplex eliminates collisions
- Dedicated bandwidth per port
- No shared medium

**TL;DR:** Ethernet = LAN technology. CSMA/CD = legacy collision handling. Modern networks use switches and full-duplex.

**Keywords:** Ethernet, CSMA/CD, collision detection, IEEE 802.3, full-duplex, media access control

---

### Q98: What is the difference between IP multicast and broadcast?

**Answer:** Broadcast sends to all devices. Multicast sends only to devices that have joined a specific group.

**Polished Answer:**

| Feature | Broadcast | Multicast |
|---------|-----------|-----------|
| **Recipients** | All devices | Only group members |
| **Efficiency** | Inefficient (all receive) | Efficient (only interested) |
| **Scope** | Local subnet | Can span networks |
| **Address** | 255.255.255.255 | 224.0.0.0 - 239.255.255.255 |
| **Management** | No management | IGMP for group joins |

**Broadcast:**
- Sent to every device on subnet
- All devices must process
- Inefficient for large networks
- Example: ARP requests

**Multicast:**
- Only group members receive
- Efficient bandwidth usage
- Requires group management (IGMP)
- Example: Video conferencing, streaming

**Multicast addresses:**
- IPv4: 224.0.0.0 to 239.255.255.255
- Well-known: 224.0.0.1 (all hosts), 224.0.0.2 (all routers)

**TL;DR:** Broadcast = all devices. Multicast = only group members. Multicast more efficient for group communication.

**Keywords:** Multicast, broadcast, IP multicast, IGMP, group communication, multicast address

---

### Q99: What is a network protocol and what are common protocols?

**Answer:** A network protocol defines rules for communication between devices. Common protocols include HTTP, DNS, SMTP, TCP, IP.

**Polished Answer:**

**What is a Network Protocol?**
A set of rules and standards that define how devices communicate over a network, including message format, transmission, and error handling.

**Common Protocols by Layer:**

**Application Layer:**
- HTTP/HTTPS (web browsing)
- DNS (name resolution)
- SMTP (email sending)
- FTP (file transfer)
- SSH (secure remote access)

**Transport Layer:**
- TCP (reliable, connection-oriented)
- UDP (unreliable, connectionless)

**Network Layer:**
- IP (addressing and routing)
- ICMP (diagnostics)
- OSPF/BGP (routing protocols)

**Data Link Layer:**
- Ethernet
- Wi-Fi (802.11)
- ARP

**Physical Layer:**
- Ethernet physical standards
- Fiber optic standards

**Protocol functions:**
- Addressing
- Error detection
- Flow control
- Congestion control
- Security

**TL;DR:** Protocols = communication rules. Layer-specific: HTTP (app), TCP/UDP (transport), IP (network).

**Keywords:** Network protocol, HTTP, DNS, TCP, IP, protocol layers, communication rules

---

### Q100: What is QoS (Quality of Service) and why is it important?

**Answer:** QoS prioritizes network traffic to ensure performance for critical applications.

**Polished Answer:**

**What is QoS?**
Quality of Service - a set of technologies that manage network resources by prioritizing certain types of traffic, ensuring performance for critical applications.

**Why QoS is important:**
- Bandwidth is limited
- Different traffic has different requirements
- Ensures real-time apps work well
- Prevents non-critical traffic from degrading critical services

**QoS mechanisms:**

**1. Classification:**
- Identify traffic types (voice, video, data)
- Mark packets with priority tags
- Based on IP, port, protocol, application

**2. Prioritization:**
- Give higher priority to sensitive traffic
- Voice/video > interactive > bulk
- Scheduling algorithms

**3. Queuing:**
- Different queues for different priorities
- Priority queue for real-time
- Weighted Fair Queuing for fair allocation

**4. Traffic Shaping:**
- Control traffic rate
- Smooth bursty traffic
- Prevent congestion

**5. Policing:**
- Enforce bandwidth limits
- Drop or mark excess traffic

**QoS metrics:**
- Latency (delay)
- Jitter (delay variation)
- Packet loss
- Bandwidth

**TL;DR:** QoS = traffic prioritization. Ensures critical apps perform. Uses classification, queuing, shaping.

**Keywords:** QoS, Quality of Service, traffic prioritization, bandwidth management, latency, jitter, queuing

---

## SUMMARY TABLE: ALL 100 QUESTIONS BY CATEGORY

| Category | Questions |
|----------|-----------|
| **OSI & Models** | Q1, Q56 |
| **TCP/UDP** | Q2, Q3, Q7, Q11, Q19, Q20, Q67, Q70, Q71, Q72, Q73, Q74, Q75, Q76, Q82 |
| **IP & Addressing** | Q5, Q10, Q14, Q15, Q32, Q33, Q34, Q55, Q65, Q66, Q84 |
| **DNS & HTTP** | Q4, Q18, Q22, Q40, Q41, Q80, Q81, Q89 |
| **Routing** | Q25, Q27, Q28, Q29, Q30, Q31, Q77, Q78, Q86, Q91, Q92 |
| **Switching & LAN** | Q8, Q17, Q24, Q35, Q36, Q37, Q38, Q39, Q85, Q87, Q93, Q94, Q97, Q98 |
| **Security** | Q6, Q9, Q21, Q53, Q59, Q60 |
| **Wireless** | Q57, Q58 |
| **Performance** | Q26, Q45, Q46, Q47, Q51, Q52, Q64, Q68, Q69, Q95, Q100 |
| **Email** | Q63 |
| **General** | Q49, Q50, Q61, Q62, Q83, Q88, Q90, Q96, Q99 |

# Sockets vs WebSockets — Are They in Your Question Bank?

## Short Answer

**Sockets:** ❌ Not directly as a standalone question. The word "socket" appears only in **Q23 (Port Numbers)** as a one-line mention (`IP + Port = Socket`). There is no dedicated question explaining socket programming, socket types, or the socket API.

**WebSockets:** ❌ **Completely missing.** There is **zero** coverage of WebSockets anywhere in the 100 questions — not as a heading, not as a sub-topic, not even a keyword.

---

## Where "Socket" Appears in the Given Material

Only **one** place:

> **Q23: What is the role of port numbers in transport-layer communication?**
> *"The combination of IP address and port is called a socket, and it uniquely identifies a communication endpoint."*

That's it. No further explanation of:
- What a socket actually is (OS-level abstraction)
- Socket types (Stream vs Datagram vs Raw)
- Socket API calls (`socket()`, `bind()`, `listen()`, `accept()`, `connect()`)
- Berkeley sockets vs Winsock
- TCP socket lifecycle

---

## What's Missing (Gaps You Should Fill)

### Sockets — Missing Coverage

| Topic | Status |
|-------|--------|
| Socket definition (IP:Port endpoint) | ⚠️ Only mentioned in Q23 |
| Stream socket (TCP) vs Datagram socket (UDP) | ❌ Missing |
| Socket API lifecycle (bind → listen → accept → connect) | ❌ Missing |
| Blocking vs non-blocking sockets | ❌ Missing |
| Socket buffers and backlogs | ❌ Missing |
| Raw sockets | ❌ Missing |
| Socket vs Port distinction | ❌ Missing |

### WebSockets — Missing Coverage

| Topic | Status |
|-------|--------|
| WebSocket protocol (RFC 6455) | ❌ Missing |
| Full-duplex vs half-duplex communication | ❌ Missing |
| WebSocket handshake (HTTP Upgrade) | ❌ Missing |
| WebSocket vs HTTP polling vs SSE | ❌ Missing |
| Persistent connection model | ❌ Missing |
| Use cases (chat, live feeds, gaming) | ❌ Missing |
| ws:// vs wss:// | ❌ Missing |

---

## Should You Add Them? (Yes — Especially for SDE Roles)

**Sockets** are frequently asked in **backend/system design** rounds:
- "How does a TCP server handle multiple clients?" → socket + `select`/`epoll`/threads
- "What happens when you call `accept()`?" → socket backlog, three-way handshake

**WebSockets** are frequently asked in **full-stack / real-time systems** rounds:
- "How would you build a chat app?" → WebSocket vs long polling
- "Why not just use HTTP?" → overhead, latency, bidirectional need

---

## Suggested Additions (If You Want to Extend the Bank)

**Q101: What is a socket? How does it relate to IP and port?**
> A socket is an OS-level communication endpoint identified by an IP address + port number + protocol (TCP/UDP). It's the programming interface (API) through which applications send/receive data over a network. Stream sockets = TCP; Datagram sockets = UDP.

**Q102: What is a WebSocket? How does it differ from HTTP?**
> WebSocket (RFC 6455) is a full-duplex, persistent communication protocol over a single TCP connection. It starts with an HTTP Upgrade handshake, then switches to a bidirectional frame-based protocol. Unlike HTTP (request-response, stateless), WebSocket allows server-initiated messages with minimal overhead — ideal for chat, live dashboards, and gaming.

**Q103: WebSocket vs HTTP Long Polling vs Server-Sent Events (SSE)?**
> - **HTTP Polling:** Client repeatedly requests; high overhead, latency.
> - **Long Polling:** Server holds request until data available; better but still per-message overhead.
> - **SSE:** Server → client only, one-way streaming over HTTP.
> - **WebSocket:** Full-duplex, persistent, lowest latency — best for bidirectional real-time apps.

---

## TL;DR

- **Sockets:** Only a passing mention in Q23. Not a dedicated question.
- **WebSockets:** Entirely absent from the 100 questions.
- **Recommendation:** Add both if preparing for backend, system design, or full-stack SDE roles — they're common follow-ups to TCP/HTTP questions.
