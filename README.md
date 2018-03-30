# Azure-VM-Network-Performance
This post will show how to test Azure VM networking performance. <br>

Azure offers a variety of VM sizes and types, each with a different mix of performance capabilities. <br>
The network bandwidth allocated to each virtual machine is metered on egress (outbound) traffic from the virtual machine. <br>
All network traffic leaving the virtual machine is counted toward the allocated limit, regardless of destination. <br>
Ingress is not metered or limited directly. <br>

For example, in this post, we will open Standard D8s v3 virtual machine as testing VM. <br>
This VM target network egress throughput is 4Gbps and support up to 4 network interface. <br>
Please note that multiple interface is used to provide different subnet design, no for bonding or increase network throughput. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/D8sV3.PNG)

# VM to VM throughput and latency within same region

# VM to VM with acceleration network performance 

# VM to VM throughput and latency cross region

# VM to VM performance with standard load balancer
