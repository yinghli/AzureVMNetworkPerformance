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

To learn how many network interfaces different Azure VM sizes support, please check [here](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes-general)

# VM to VM throughput and latency within same region

We setup two D8sV3 VM at Azure East Asia region. 
Using [iperf3](https://iperf.fr/) to test network throughput and [qperf](https://www.opsdash.com/blog/network-performance-linux.html) to test latency. <br>

For network throughput test. <br>
Server side use default setup ``` iperf3 -s ``` <br>
Client side use single thread and lastest 30 second ``` iperf3 -c 10.0.2.4 -t 30``` <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20bw%20without%20Acc.PNG)

For Network latency test. <br>
Server side use default setup ``` qperf ``` <br>
Client side use tcp latency setup ``` qperf -v 10.0.2.4 tcp_lat``` <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20lat%20without%20Acc.PNG)

From the result, we can see D8sV3 VM egress throughput is 3.65Gbps with single TCP thread. CWND value is 3.27MB. 
Netwokr latency is 146us.

# VM to VM with acceleration network performance 

All detail information about acceleration network, please check [here](https://docs.microsoft.com/en-us/azure/virtual-network/create-vm-accelerated-networking-cli) <br>

We setup two D8sV3 VM with acceleration network at Azure East Asia region. <br>

For network throughput test. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20bw%20with%20Acc.PNG)

For Network latency test. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20lat%20with%20Acc.PNG)

From the result, we can see D8sV3 VM egress throughput is 3.82Gbps with single TCP thread. CWND value is 1.36MB. 
Netwokr latency is 40us.

# VM to VM performance with standard load balancer

Azure released [standard load balancer(SLB)](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-standard-overview). This SLB support low latency load sharing and [HA port](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-ha-ports-overview).<br>
In this post, we setup a SLB, put 2 VM with acceleration networking in the backend pool.<br>
We setup another VM with acceleration netwokring and send traffic to SLB frontend IP address. <br>
We will test the bandwidth and latency impact when adding new SLB. 

For network throughput test. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-SLB%20bw%20with%20Acc.PNG)

For Network latency test. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-SLB%20lat%20with%20Acc.PNG)

From the result, we can see D8sV3 VM egress throughput is 3.28Gbps with single TCP thread. CWND value is 1.61MB. 
Netwokr latency is 53us.

# Summary

Parameters      | VM-VM     | VM-VM with Accelerate | VM-SLB-VM with accelerate|
----------------| ----------|-----------------------|--------------------------|
Throughput      | 3.65Gbps  | 3.82Gbps              | 3.28Gbps                 |
CWND            | 3.27MB    | 1.36MB                | 1.61MB                   |
Latency         | 146us     | 40us                  | 53us                     |


