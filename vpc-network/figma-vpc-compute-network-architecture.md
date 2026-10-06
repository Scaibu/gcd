# GCP VPC and Compute Networking Architecture (Figma)

[Open the editable GCP VPC and Compute Networking — In Depth FigJam board](https://www.figma.com/board/Kd7WWDvyHMlGiGOshplrse/)

The board uses separate colors for client/external systems, the global load-balancing edge, the VPC boundary, and regional subnets. It covers:

- A global custom-mode VPC with regional subnets in `us-central1` and `europe-west1`.
- Compute Engine instances managed by regional managed instance groups, with global Application Load Balancer backends and cross-region failover.
- Global VPC routes and distributed firewall rules applied to workload interfaces.
- Internet-bound egress through Cloud NAT and private Google API access through Private Google Access.
- Hybrid connectivity using HA VPN and Cloud Router BGP route exchange.

The subnet CIDRs shown (`10.1.0.0/24` and `10.2.0.0/24`) are illustrative examples from the repository's architecture notes, not deployment settings.

## Detailed diagrams on the board

These are Figma-rendered vector previews stored in this repository, so they display directly on GitHub. Open the board above to edit them in FigJam.

### 1. Global VPC and regional subnets

![Figma diagram showing a global VPC with regional subnets and zonal VMs](./figma-networking-assets/01-global-vpc-subnets.svg)

### 2. Subnet addressing and reserved IPs

![Figma diagram showing a subnet address range, reserved addresses, and secondary ranges](./figma-networking-assets/02-subnet-addressing.svg)

### 3. Compute Engine packet lifecycle

![Figma sequence diagram showing packet flow through a Compute Engine VM interface and VPC fabric](./figma-networking-assets/03-packet-lifecycle.svg)

### 4. Route selection and next hops

![Figma diagram showing subnet, custom, dynamic, and default route selection](./figma-networking-assets/04-route-selection.svg)

### 5. Stateful firewall evaluation

![Figma diagram showing rule priority, default firewall behavior, and connection tracking](./figma-networking-assets/05-firewall-evaluation.svg)

### 6. Egress, Cloud NAT, and Google APIs

![Figma diagram showing Cloud NAT, external IP, and Private Google Access egress paths](./figma-networking-assets/06-egress-cloud-nat.svg)

### 7. Global load balancer and regional MIGs

![Figma diagram showing global load balancing, health checks, and regional managed instance groups](./figma-networking-assets/07-global-load-balancer.svg)

### 8. HA VPN and Cloud Router BGP

![Figma diagram showing redundant HA VPN tunnels and Cloud Router BGP routes](./figma-networking-assets/08-ha-vpn-bgp.svg)

### 9. Shared VPC host and service projects

![Figma diagram showing centralized Shared VPC administration and service project workloads](./figma-networking-assets/09-shared-vpc.svg)

### 10. VPC Network Peering and isolation

![Figma diagram showing reciprocal VPC peering and non-transitive route exchange](./figma-networking-assets/10-vpc-peering.svg)

### 11. Subnet and secondary range growth

![Figma diagram showing capacity assessment, CIDR expansion, and validation](./figma-networking-assets/11-range-growth.svg)

### 12. Network diagnostics and observability

![Figma diagram showing Connectivity Tests, VPC Flow Logs, Packet Mirroring, and diagnosis](./figma-networking-assets/12-diagnostics.svg)

## Related repository references

- [VPC high- and low-level design](./hld-lld-design.md)
- [VPC and subnet command reference](./shell-commands.md)
- [Compute Engine architecture and CLI references](../compute-engine/README.md)
- [VPC case studies: topology, packet lifecycle, Compute integration, and configuration](./casestudy/README.md)
- [VPC official references](./references.md)
