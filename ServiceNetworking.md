### Traffic sent from a pod to a service fronting a bunch of pods (e.g. traffic from an app layer pod sent to the service fronting DB pods)
TBD
### Traffic sent from an external client (VM in the same or a connected network) to a pod through a service (Node port)
- Step 1.1 : Client VM initiates the connection to the node port service exposed at 10.240.0.4:32000  
  - src: 10.241.0.4
  - dest: 10.240.0.4:32000
- Step 1.2: Kube proxy programs the Linux kernel to NAT the traffic received at the node port to be sent to the clusterIP (that is created automatically)  
  - src : 10.241.0.4
  - dest: 172.31.0.5:443
  - Step 1.2.1: Kube proxy also programs the Linux kernel to intercept the traffic sent to a cluster ip (172.31.0.5 in this case) and destination NAT the traffic to one of the pods served by this service (load balancing of sorts)  
    - dest: 10.241.0.4 translated to 
    - dest: 10.244.1.2 (Note: this is a pod in a node different from the one that received the traffic on its node port. The pods could be scheduled on different nodes based on the pod placement constraints)
    - **Note**: 
    1. The diagram shows a simplified version of the data that the NAT table would hold. In this case, the Cluster IP would be translated to one of the 2 dest pod addresses (10.244.0.3 or 10.244.1.2). 
    2. When the service and the corresponding pod deployments are created, kube proxy would add the rules in the NAT table. The pod addresses would be the endpoint objects as interpreted by K8s. 
    3. If the service configuration has this config enabled ("internaltrafficpolicy:Local")[https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/#using-service-internal-traffic-policy], then KubeProxy will not add the endpoint/podIp on Node-2. The NAT table would then contain only one endpoint (as assumed in this diagram)
    4. Also, if the result of the endpoint selection from the NAT process results in the endpoint in the same node, then steps 1.3 through 1.6 will not be applicable
  - Step 1.2.2: There is a need to deliver the response from Node-1 (10.240.0.4) because that is the node that the client understands sending a request to. However the pod that handles this request is in Node-2 (10.240.0.5). Hence the Kernel also does a source NAT
    - src: 10.241.0.4 translated to
    - src: 10.240.0.4
- Step 1.3: The linux kernel (virtual router) sends the packets destined for anything but the pod cidr of the local node (10.244.0.0/24) to the node's eth0 interface
- Steps 1.4 and 1.5 - The request is sent from Node-1 to Node-2 and then through the rules programmed by the kernel, the traffic finally reaches the target pod (10.244.1.2) via the bridge (cbr0). The individual hops have been skipped for brevity. Refer to the pod networking documentation for more information
- Step 1.6: The pod reverses the source and destination 
- Step 2.0: The Linux kernel should be able to reverse the source and destination to the node-ip and the VM's ip respectively  
  - src: 10.240.0.4
  - dest: 10.241.0.4
