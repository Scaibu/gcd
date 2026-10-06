# GKE Networking Architecture (Figma)

[Open the editable GKE Networking — In Depth FigJam board](https://www.figma.com/board/iaDzdSJX1eyF3lQHWup2tC/)

The board illustrates a color-coded GKE networking reference, including:

- Public ingress through Cloud Armor and the global external Application Load Balancer to GKE NEG pod endpoints.
- Regional subnet primary range for node addresses, and secondary ranges for VPC-native Pod alias IPs and Service ClusterIP addresses.
- GKE nodes, pods, Service endpoint selection, and same-cluster pod traffic.
- Authorized access to the private GKE control-plane endpoint from administrators and node kubelets.
- VPC firewall policy and routing, outbound internet access through Cloud NAT, and Google API access through Private Google Access.

## Detailed diagrams on the board

These are Figma-rendered vector previews stored in this repository, so they display directly on GitHub. Open the board above to edit them in FigJam.

### 1. VPC-native IP allocation

![Figma diagram showing GKE VPC-native node, Pod, and Service address allocation](./figma-networking-assets/01-vpc-native-ip-allocation.svg)

### 2. Pod-to-Pod datapath

![Figma diagram showing same-node and cross-node Pod traffic](./figma-networking-assets/02-pod-to-pod-datapath.svg)

### 3. External ingress to a Pod

![Figma sequence diagram showing external ingress through Cloud Armor, load balancing, NEG, and Pod policy](./figma-networking-assets/03-external-ingress.svg)

### 4. Egress routing choices

![Figma diagram showing GKE egress through Cloud NAT, Private Google Access, and hybrid connectivity](./figma-networking-assets/04-egress-routing.svg)

### 5. Cluster DNS resolution

![Figma sequence diagram showing Kubernetes, VPC private, and public DNS resolution](./figma-networking-assets/05-cluster-dns.svg)

### 6. Layered network security

![Figma diagram showing Cloud Armor, VPC firewall, and Kubernetes NetworkPolicy layers](./figma-networking-assets/06-layered-security.svg)

### 7. Gateway API reconciliation

![Figma diagram showing Gateway API resources reconciled into Google Cloud load-balancing resources](./figma-networking-assets/07-gateway-api.svg)

### 8. Private control-plane connectivity

![Figma diagram showing private GKE API access from administrators and node agents](./figma-networking-assets/08-private-control-plane.svg)

### 9. Kubernetes Service exposure paths

![Figma diagram comparing ClusterIP, NodePort, LoadBalancer, and Gateway exposure](./figma-networking-assets/09-service-exposure.svg)

### 10. Multi-zone availability and failover

![Figma diagram showing GKE multi-zone health checks, surviving replicas, and rescheduling](./figma-networking-assets/10-multi-zone-failover.svg)

## Related repository references

- [Unified Kubernetes, CNI, and GCP VPC networking guide](./networking-shell-command.md)
- [Kubernetes high- and low-level design](./hld-lld-design.md)
- [Kubernetes command reference](./shell-commands.md)
- [Kubernetes official references](./references.md)

This is a reference architecture. CIDR ranges and optional components should be checked against the target cluster configuration before implementation.
