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
<picture>  
- CoreDNS's configuration i.e the configmap resource that holds the **CoreFile** consists of a configuration that specifies that the **Kubernetes** plugin would handle resolution requests for cluster.local  
<picture>  
**Flow Steps**
- A worker pod does a name resolution request for nginx-service
  - **Note**: If the worker pod and the service are in the same namespace, then NSlookup using just the service name would suffice. If the pod and the service are in different namespaces, then the pod should try to resolve *nginx-service.default.svc.cluster.local*
- The requests would be sent to CoreDNS service, as it is the DNS server that has been configured in the pod's /etc/resolv.conf by kubelet
- CoreDNS would then use the Kubernetes plugin to resolve the request as it is for the cluster.local domain


