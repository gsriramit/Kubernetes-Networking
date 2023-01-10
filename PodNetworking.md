## Routing Scenarios
### Traffic routed between pods on the same node
Scenario: Pod A (10.244.0.2) communicates with Pod B (10.244.0.3)  
- The default route set on the pod network namespaces direct all the traffic originating from these to the bridge through their etho0 interfaces
- As both the pods are connected to the bridge (which is a virtual layer 2 switch), they should be able to reach each other directly through the bridge. 

### Traffic routed between pods on different nodes
Scenario: Pod A (10.244.0.2) communicates with Pod B (10.244.1.2)  
- The default route set on the pod network namespaces direct all the traffic originating from these to the bridge through their etho0 interfaces
- The bridge would send the packets to the Linux kernel managed virtual router as the destination address is not an interface that is connected directly to the bridge
- The kernel's IP tables would be programmed with a default route to forward the packets to the node's eth0 interface
- The packets are then handled by the node's azure network interface (10.240.0.4)
- The packets contain a destination address of 10.244.1.2 and the node sees them as not destined for itself
- 2 important configurations at the node and subnet help forward the packets further appropriately
  - IP forwarding has been enabled at the node. This lets the node forward the packets that are not destined to itself. This is the common configuration of NVAs
  - The UDR configured on the subnet now instructs the packets destined for (10.244.1.2) to be sent to the target node (10.240.0.5) as the next hop
- The custom route configured on the Node-2 in the kernel's IP tables would forward packets destined for the pod CIDR (10.244.1.0/24) to the cbr0 interface of the bridge
- The bridge now sends the packets to the targeted pod (10.244.1.2) at the pod's eth0 interface through the veth (veth10) pair
- There would be no need for SNAT or DNAT in this flow
- The return traffic would follow the similar process to reach the pod that initiated the traffic

### Traffic routed between a pod and a Virtual machine in the same virtual network (VM on a different subnet). 2-way flows
Scenario: Pod A (10.244.0.2) communicates with Virtual Machine (10.241.0.4)
**Request Flow**  
- The default route set on the pod network namespaces direct all the traffic originating from these to the bridge through their etho0 interfaces
- The bridge would send the packets to the Linux kernel managed virtual router as the destination address is not an interface that is connected directly to the bridge
- The kernel's IP tables would be programmed with a default route to forward the packets to the node's eth0 interface
- Also programmed in the IP tables' NAT table would be a rule to SNAT the traffic to the IP address of the node (src: 10.240.0.4 dest: 10.241.0.4)
- The packets are then handled by the node's azure network interface (10.240.0.4) at this point
- As the destination is also an IP address within the same network (or could be a connected network), the packets would be sent through the MS backbone network to the destination node  
**Response Flow**  
- The response sent by the VM would have the srcIp as 10.241.0.5 and the destIp as 10.240.0.4
- The packets would be sent to Node-1's ANI through the MS backbone network
- The packets received on Node-1's would now need to be source NAT'd to 10.244.0.2. The Linux Kernel's RAW table (this is yet to be validated) is supposed to maintain the state of the traffic and hence would be able to associate the return traffic to the request sent earlier
- Based on the NAT configuration, the destination would be NAT'd to 10.244.0.2 (from 10.241.0.4)
- The custom route programmed in the kernel would send all traffic destined to 10.244.0.0/24 to the bridge on cbr0. So now the packets are sent from the node's eth0 to the bridge on cbr0
-  The bridge would now be able to send the packets to Pod A through its veth pair

### Traffic routed between a pod and an external URL/IP address (e.g. Storage account or Container Registry)
Scenario: Pod A (10.244.0.2) communicates with a Storage account on its public IP (with or without Service Endpoints)
**Request Flow**  
- The default route set on the pod network namespaces direct all the traffic originating from these to the bridge through their etho0 interfaces
- The bridge would send the packets to the Linux kernel managed virtual router as the destination address is not an interface that is connected directly to the bridge
- The kernel's IP tables would be programmed with a default route to forward the packets to the node's eth0 interface
- Also programmed in the IP tables' NAT table would be a rule to SNAT the traffic to the IP address of the node (src: 10.240.0.4 dest: 10.241.0.4)
- The packets are then handled by the node's azure network interface (10.240.0.4) at this point
- As the destination is a public IP address, the packets have to be SNAT'd here to use a public IP address. The public IP address used depends on a number of factors. 
  - If these nodes are placed behind a public load balancer, then the LB's public IP can be used for egress traffic
  - If there is an instance level public IP, then that would be used (this is not usually done)
  - If there is a NAT gateway associated public IP address configured to this subnet, then that would be used
  - If no explicit public IP addresses are available, then there would be an Azure provided ephemeral public IP that would be used
- Src: PIP of the NAT gateway, destination : PIP of the Azure Storage Account 
**Response Flow**  
- The response sent by the Storage account would have the srcIp as its PIP and the destIp as the NAT Gateway's PIP
- The packets would now be DNAT'd to the Node-1's private IP (10.240.0.2)
- The packets would be sent to Node-1's ANI through the MS backbone network
- The packets received on Node-1's would now need to be source NAT'd to 10.244.0.2. The Linux Kernel's RAW table (this is yet to be validated) is supposed to maintain the state of the traffic and hence would be able to associate the return traffic to the request sent earlier
- Based on the NAT configuration, the destination would be NAT'd to 10.244.0.2 (from 10.241.0.4)
- The custom route programmed in the kernel would send all traffic destined to 10.244.0.0/24 to the bridge on cbr0. So now the packets are sent from the node's eth0 to the bridge on cbr0
-  The bridge would now be able to send the packets to Pod A through its veth pair

