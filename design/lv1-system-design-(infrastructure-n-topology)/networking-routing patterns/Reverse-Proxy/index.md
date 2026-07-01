# Reverse Proxy Architecture

A Reverse Proxy sits in front of backend web servers, intercepting all incoming client requests and routing them to the appropriate internal microservices.
- **Reverse Proxy vs. Forward Proxy:** A **Forward Proxy** sits in front of clients (hides client IPs from the internet), while a **Reverse Proxy** sits in front of servers (hides backend infrastructure from clients).
- **Core Functions:** TLS/SSL Termination, Gzip/Brotli compression, static file caching, port forwarding, and backend route masking.

## Interview Questions & Answers

### Q1: Why is TLS termination highly recommended at the Reverse Proxy level?
- **Answer:** CPU offloading. Decrypting TLS packets is highly CPU-intensive. By terminating TLS at the Reverse Proxy (e.g., Nginx), your backend application microservices can communicate over unencrypted HTTP privately inside the VPC, saving backend CPU cycles and simplifying certificate management to a single perimeter point.
