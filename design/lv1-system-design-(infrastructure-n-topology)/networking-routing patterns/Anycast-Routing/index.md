# Anycast Routing

Anycast is a network routing technique where multiple physical servers distributed across the globe share the exact same IP address.
- **Routing:** Border Gateway Protocol (BGP) automatically routes client IP packets to the physically nearest Anycast server node.
- **Failover:** If an Anycast node crashes, BGP automatically re-routes traffic to the next nearest server node, offering built-in high availability and latency optimization.

## Interview Questions & Answers

### Q1: How is Anycast utilized by modern CDNs and DNS providers?
- **Answer:** CDNs (like Cloudflare) and DNS providers (like Google's `8.8.8.8`) announce their service IP addresses using Anycast. No matter where the user is physically located, requesting the IP automatically routes their packet to the nearest CDN edge node, minimizing physical latency and network hops. It also provides excellent DDoS protection by geographically dispersing traffic spikes across global edge centers.
