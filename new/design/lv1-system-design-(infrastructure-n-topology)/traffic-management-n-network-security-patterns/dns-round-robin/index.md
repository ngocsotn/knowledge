# DNS Round Robin

DNS Round Robin is a simple load balancing mechanism where a DNS server returns a list of multiple physical IP addresses for a single domain name, rotating the ordering of IPs with each DNS query.
- **Speed:** Sub-millisecond lookup speeds; zero gateway performance overhead.
- **Drawback:** Lacks real-time server health checks; browsers cache DNS records based on TTL, meaning clients may continue attempting to connect to a crashed server.

## Interview Questions & Answers

### Q1: What is the main limitation of DNS Round Robin for system failover?
- **Answer:** DNS Caching. Browsers, operating systems, and intermediary ISP DNS resolvers heavily cache DNS record IP addresses based on TTL. If a backend server crashes, the DNS server can remove its IP from the rotation, but clients who already have the IP cached will continue sending traffic to the crashed node until their local TTL expires, resulting in user-facing failures.
