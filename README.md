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

# VM to VM throughput and latency without acceleration network

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

# VM to VM performance with basic load balancer

Azure by default provide a free load balancer.<br>
This load balancer required that VM should be same AVAILABILITY SET. <br>
In this setup, we create a basic load balancer and put 2 VM in same availability set with acceleration networking in the backend pool.<br>
We will test the bandwidth and latency impact when adding basic load balancer. <br> 
For network throughput test. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-LB%20bw%20with%20Acc.PNG)

For Network latency test. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-LB%20lat%20with%20Acc.PNG)

From the result, we can see D8sV3 VM egress throughput is 3.28Gbps with single TCP thread. CWND value is 1.32MB. 
Netwokr latency is 81us.

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

# VM to VM performance in different region

We setup two D8sV3 VM with acceleration network at Azure East Asia and Central US region.<br>
We will first test the two VM network latency, this will have big impact for network throughput.<br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20lat%20with%20Acc%20cross%20region.PNG)

From the result, one way network latency is 111ms, the round trip latency is 222ms. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20bw%20with%20Acc%20cross%20region.PNG)

If we use the default setup, single TCP thread throughput can only be only 107Mbps because network latency.<br>
Also we see that CWND is 6.05MB. <br>
If we wants to increase the single TCP thread throughput, we must increase the TCP send and receive buffer.<br>
Basiclly, Throughput = buffer size / latency. If we wants to get 4Gbps throughput with 230ms latency network, buffer size should be around 120MB.<br>
We modify system TCP parameter to increase the buffer size.<br>
```
echo 'net.core.wmem_max=131072000' >> /etc/sysctl.conf
echo 'net.core.rmem_max=131072000' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem= 10240 87380 131072000' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem= 10240 87380 131072000' >> /etc/sysctl.conf
```

After the setup, we retest with iperf3 single tcp thread and get below result.<br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20bw%20with%20Acc%20cross%20region%20tcp.PNG)

# Summary

From the below table, accelerate network will improve the end to end network latency.<br>
Lower latency will reduce the CWND when reaching the same level TCP throughput.<br>
New standard load balancer will only add small latency for end to end network latecnty.<br>

Parameters      | VM-VM without Acc | VM-VM with Acc | VM-LB-VM with Acc |VM-SLB-VM with Acc | VM-VM cross region|
----------------| ------------------|----------------|-------------------|-------------------|-------------------|
Throughput      | 3.65Gbps          | 3.82Gbps       | 3.28Gbps          | 3.28Gbps          | 3.72Gbps          |
CWND            | 3.27MB            | 1.36MB         | 1.31MB            | 1.61MB            | 102MB             |
Latency         | 146us             | 40us           | 81us              | 53us              | 222ms             |


