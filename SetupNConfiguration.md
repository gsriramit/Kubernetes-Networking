Basic Configuration and Setup

	1. Create the Linux Network namespaces
	```
    sudo ip netns add $NSI
    sudo ip netns add $NS2
    ```
    2. Creating the veth pairs
    ```
    sudo ip link add veth10 type veth peer name eth0
    sudo ip link add veth20 type veth peer name eth0
    ```
    3. Adding the veth pairs to the namespaces
    ```
    sudo ip link set eth0 netns $NS1
    sudo ip link set eth0 netns $NS2
    ```
    4. Configuring interfaces in the namespaces with IP addresses
    ```
    sudo ip netns exec $NS1 ip addr add 10.244.0.2 dev eth0
    sudo ip netns exec $NS2 ip addr add 10.244.0.3 dev eth0
    ```
    5. Enabling the interfaces inside the network namespaces
    ```
    sudo ip netns exec $NS1 ip link set dev eth0 up
    sudo ip netns exec $NS2 ip link set dev eth0 up
    ```
    6. Creating the bridge and assigning IP address to it
    ```
    sudo ip link add cbr0 type bridge
    ip link show type bridge
    ip link show cbr0
    ```
    7. Adding the interface on the other end of the veth pairs to the bridge
    ```
    sudo ip link set dev veth10 master cbr0
    sudo ip link set dev veth20 master cbr0
    ```
    8. Assigning an IP address to the bridge and enabling it
    ```
    sudo ip addr add $BRIDGE_IP dev cbr0
    sudo ip link set dev crb0 up
    ```
    9. Enabling the pod interfaces connected to the bridge
    ```
    sudo ip link set dev veth10 up
    sudo ip link set dev veth20 up
    ```
    10. Setting the loopback interfaces in the network namespaces
    ```
     sudo ip netns exec $NS1 ip link set lo up
     sudo ip netns exec $NS2 ip link set lo up
     sudo ip netns exec $NS1 ip a
     sudo ip netns exec $NS2 ip a
     ```
     11. Setting the default route in the network namespaces
     ```
     sudo ip netns exec $NS1 ip route add default via $BRIDGE_IP dev eth0
     sudo ip netns exec $NS2 ip route add default via $BRIDGE_IP dev eth0
     ```
     12. Setting the default route on the node for traffic to reach the network namespaces 
     ```
     sudo ip route add $TO_BRIDGE_SUBNET via $TO_NODE_IP dev eth0
     ```
     13. Enabling IP forwarding on the node
     ```
     sudo sysctl -w net.ipv4.ip_forward=1
     ```
     14. Routes programmed by Kube proxy on the Linux Kernel (IP tables) to handle the traffic a) that originates from the pod namespaces and b) external traffic
     ```
     sudo ip route add $BRIDGE_SUBNET via $BRIDGE_IP dev cbr0
     sudo ip route add default via $NODE_IP dev eth0
     ```
     15. NAT rules programmed by kube proxy in the NAT table of the Linux Ip-tables
     ```
     # reference: https://www.karlrupp.net/en/computer/nat_tutorial

     sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE 
     # (OR)
     iptables --table nat --append POSTROUTING --out-interface eth0 -j MASQUERADE
     # -t nat	 	select table "nat" for configuration of NAT rules.
     # -A POSTROUTING	 	Append a rule to the POSTROUTING chain (-A stands for "append").
     # -o eth0 	this rule is valid for packets that leave on the etho0 network interface of the node (-o stands for "output")
     # -j MASQUERADE  the action that should take place is to 'masquerade' packets, i.e. replacing the sender's address (i.e. the pod's address) by the nodes's address (i.e. the VM's address).
     ```
