# IP Whitelisting & VPC Peering

Security patterns used to establish isolated, secure network boundaries between systems.
- **IP Whitelisting:** Enforcing strict access control by only allowing connections from a whitelisted set of public or private IP addresses.
- **VPC Peering:** A networking connection that routes traffic between isolated Virtual Private Clouds (VPC) privately using private IP addresses.
- **VPC PrivateLink:** Exposing services privately across VPC boundaries without exposing the entire network.

## Interview Questions & Answers

### Q1: Why is VPC Peering vastly more secure than IP Whitelisting over the public internet?
- **Answer:** Public internet bypass. VPC Peering connects virtual networks directly using AWS's or Google's private global fiber network. Traffic never traverses the public internet, preventing packet sniffing, interception, or external DDoS attempts. It routes via private IP addresses (`10.x.x.x`), keeping the internal infrastructure invisible to the public web.
