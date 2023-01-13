## Architecture Diagram
![Aks-Scenarios - AKS-DNS-Networking](https://user-images.githubusercontent.com/13979783/212348577-ff3f4cbb-83c1-4416-a5e1-5bb7c380fce1.png)

## DNS Resolution from the context of worker pods
### Concepts
1. Kubernetes publishes information about Pods and Services which is used to program DNS. Kubelet configures Pods' DNS so that running containers can lookup Services by name rather than IP
2. Services defined in the cluster are assigned DNS names. By default, a client Pod's DNS search list includes the Pod's own namespace and the cluster's default domain
3. With only CoreDNS's Kubernetes plugin specified, the plugin will default to the zone specified in the server’s block. It will handle all queries in that zone and connect to Kubernetes in-cluster (ref: https://coredns.io/plugins/kubernetes/). By default, the zone specified in the server block would be **cluster.local**
```
kubernetes [ZONES...] {
    endpoint URL
    tls CERT KEY CACERT
    kubeconfig KUBECONFIG [CONTEXT]
    namespaces NAMESPACE...
    labels EXPRESSION
    pods POD-MODE
    endpoint_pod_names
    ttl TTL
    noendpoints
    fallthrough [ZONES...]
    ignore empty_service
}
```
4. When CoreDNS starts with the kubernetes plugin enabled, it connects to the Kubernetes API and synchronize all object watches.
   - By default the kubernetes plugin watches Endpoints via the **discovery.EndpointSlices** API. However the **api.Endpoints** API is used instead if the Kubernetes version does not support the EndpointSliceProxying feature gate by default (i.e. Kubernetes version < 1.19).
5. CoreDNS provides mechanisms to extend the DNS resolution process through the forward plugin (ref: https://coredns.io/plugins/kubernetes/#stubdomains-and-upstreamnameservers)
   - When the custom domains and the corresponding forward lookup servers are specified, CoreDNS forwards the lookup requests to those custom DNS servers through the network connectivity established from the host.
```
cluster.local:53 {
    kubernetes cluster.local
}
example.local {
    forward . 10.100.0.10:53
}

. {
    forward . 8.8.8.8:53
}
```
6. Pods that use the "Default" DNS policy would be using the DNS configuration of the underlying node on which the pods are scheduled. So, name resolution requests for external public domains such as Microsoft.com and Azure specific domains such as **<sitename>.azurewebsites.net** would be sent to Azure DNS (i.e. 168.63.129.16)

### Resolution Flows
#### Lookup of services and pods in the Cluster.Local Zone
**General Points**  
- CoreDNS's kubernetes plugin should have read the endpoint information from the API server by querying the appropriate API endpoint. As a result the plugin should now be having sufficient information to resolve lookup requests of pods and services within the cluster domain
- The kubelet would have configured the pods to use the name servers for lookup requests. In cluster where CoreDNS runs, clusterIP of the coreDNS service is configured in the pod's **resolv.conf** file for requests to cluster.local and its subdomains  
![image](https://user-images.githubusercontent.com/13979783/212342391-4abc0f6c-eca2-4309-8d0b-4b8cabeb02bb.png)
- CoreDNS's configuration i.e the configmap resource that holds the **CoreFile** consists of a configuration that specifies that the **Kubernetes** plugin would handle resolution requests for cluster.local  
 ![image](https://user-images.githubusercontent.com/13979783/212342918-3106405f-f965-44df-af65-8e74aec22b95.png)  
**Flow Steps**
- A worker pod does a name resolution request for nginx-service
  - **Note**: If the worker pod and the service are in the same namespace, then NSlookup using just the service name would suffice. If the pod and the service are in different namespaces, then the pod should try to resolve *nginx-service.default.svc.cluster.local*  
 ![image](https://user-images.githubusercontent.com/13979783/212343865-7e2ae008-50ed-4539-9445-d8c4a33018a2.png)
- The requests would be sent to CoreDNS service, as it is the DNS server that has been configured in the pod's /etc/resolv.conf by kubelet
- CoreDNS would then use the Kubernetes plugin to resolve the request as it is for the cluster.local domain  
#### Lookup of External Domains
- A worker pod does a name resolution request for google.com
- CoreDNS service receives the request. According the resolution graph, the kubernetes plugin would not be able to handle the request and the same would be passed down to the foward plugin.
- The forward plugin, according to the default configuration forwards the request to /etc/resolv.conf
- The behavior under the hoods would be for the coreDNS pods to forward the request to the Azure DNS (168.63.129.16) provided that is what is configured at the Virtual Network level
  - If a custom DNS server had been configured at the vnet, then CoreDNS would have forwarded the requests to the specified DNS server
  - Another way of forwarding the requests to a custom DNS server would be updating the **forward** block of the CoreFile. Note: This cannot be done manually and has to be done by applying a custom config-map. Next section contains multiple references to how this can be done  
![image](https://user-images.githubusercontent.com/13979783/212345606-950119ea-322f-4813-8d5e-dfc55ca7edb1.png)  

### Example of a resolution Graph
    
![image](https://user-images.githubusercontent.com/13979783/212345971-cfacf8e1-c712-42cd-a5cb-9a29f7550765.png)  
 src:https://www.youtube.com/watch?v=qRiLmLACYSY&ab_channel=CNCF%5BCloudNativeComputingFoundation%5D  
    
### Further Reading

1. Customizing CoreDNS
   - https://learn.microsoft.com/en-us/azure/aks/coredns-custom
   - https://medium.com/@rishasi/custom-dns-configuration-with-coredns-aks-kubernetes-599ecfb46b94
   - https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/
2. DNS for services and Pods - Kubernetes official documentation
   - https://medium.com/@rishasi/custom-dns-configuration-with-coredns-aks-kubernetes-599ecfb46b94
   - Pod's DNS policy - https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy
3. Documentation of the Kubernetes plugin -CoreDNS official documentation
   - https://coredns.io/plugins/kubernetes/
4. Configuring multiple custom upstream nameservers. This is required to handle azure internal domain names while also using a common upstream server
   - https://www.danielstechblog.io/setting-custom-upstream-nameservers-for-coredns-in-azure-kubernetes-service/
5. Debugging DNS requests of pods and services
   - https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/
6. Understanding CoreDNS in K8s - CNCF presentation
   - https://www.youtube.com/watch?v=qRiLmLACYSY&ab_channel=CNCF%5BCloudNativeComputingFoundation%5D
7. A closer look at CoreDNS (an amazing write-up)
   - https://blog.opstree.com/2020/06/16/a-closer-look-at-coredns/

## Future Exploration
Node-Local DNS Cache to reduce the time taken for name resolution requests  
https://www.youtube.com/watch?v=XbkViBUuScE

