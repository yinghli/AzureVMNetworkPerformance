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

Beside network throughput, some customers want to know Azure VM packet processing capability, as known as network interface forwarding rate when packet size is 64 bytes. <br>
Qperf have a capability to send different message size when testing TCP bandwidth. We will use this feature to test Azure VM packet forwarding rate information. <br>
We will test single CPU core message rate processing capability and total VM capability. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/Single%20Core%20PPS.PNG)

From the result, we can see that single CPU core can get 603Kpps when message size is 1 byte. <br>
So we will scale this test using script and leverage total 8 CPU in the VM. <br>
On server side, we open 8 qperf on parallel and use taskset to assign each qperf to run on each CPU core. <br>
```
for cpu in {0..7}
do
 taskset -c $cpu ./qperf -lp $[cpu+10000] &
done

```
On client side, we open 8 qperf on parallel and do same taskset as server side. We collect the msg_rate output and sum all CPU core to get VM total message rate. <br>
```
#!/bin/bash
for cpu in {0..7}
do
 taskset -c $cpu ./qperf -vv -m 1 -t 30 -lp $[cpu+10000] 10.0.2.5 tcp_bw quit | grep msg_rate &
done
```

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/Multi%20Core%20PPS.PNG)
From the result, we can see when 8 core process the packet, each qperf can only handle 400K message rate. <br>
We sum all 8 thread together, D8sV3 VM total message processing rate is about 3.1Mpps. 

# VM to VM performance with basic load balancer

Azure by default provide a free load balancer.<br>
This load balancer required that VM should be same AVAILABILITY SET. <br>
In this setup, we create a basic load balancer and put 2 VM in same availability set with acceleration networking in the backend pool.<br>
We setup another VM with acceleration netwokring and send traffic to LB frontend IP address. <br>
We also need to define the load labance rule in order that testing traffic can pass the load balancer. We use TCP 12000 for testing.<br>
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
Because of HA port support, we don't need to define any specify port for load balance rules.<br>
We will test the bandwidth and latency impact when adding new SLB. 

For network throughput test. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-SLB%20bw%20with%20Acc.PNG)

For Network latency test. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-SLB%20lat%20with%20Acc.PNG)

From the result, we can see D8sV3 VM egress throughput is 3.28Gbps with single TCP thread. CWND value is 1.61MB. 
Netwokr latency is 53us.

# VM to VM performance in different region

In Azure, there are 3 methods can link those two region VM together. <br>
First one is using VM public IP(PIP), VM can talk directly with their PIP. <br>
Second one is using VNET-VNET IPSec VPN. Each region VNET will setup an IPSec VPN gateway, two gateways will setup an IPSec VPN tunnel, VM can talk each other via tunnel. <br>
Third one is using new feature called global VNET peering. This feature allow different region VM can talk directly without any gateway support. <br> 
We will first test the two VM network latency, this will have significant impact for network throughput. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20lat%20Cross%20PIP.PNG)

From the result, one way network latency is 83ms, the round trip latency is 166ms. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20bw%20Cross%20PIP.PNG)

If we use the default setup, single TCP thread throughput can only be only 131Mbps because of high network latency.<br>
Also we see that CWND is 6.02MB. <br>
If we wants to increase the single TCP thread throughput, we must increase the TCP send and receive buffer.<br>
Basiclly, if there is no packet drop, TCP Throughput = buffer size / latency. If we wants to get 4Gbps throughput with 166ms latency network, buffer size should be around 85MB.<br>
We modify system TCP parameter to increase the buffer size to 128MB.<br>
```
echo 'net.core.wmem_max=131072000' >> /etc/sysctl.conf
echo 'net.core.rmem_max=131072000' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem= 10240 87380 131072000' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem= 10240 87380 131072000' >> /etc/sysctl.conf
sysctl -p
```

After the setup, we retest with iperf3 single tcp thread and get below result.<br>
We see that throughput is around 2Gbps and have packet retry. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20bw%20tcp%20tune%20Cross%20PIP.PNG)

Second, we will setup two IPSec VPN gateway and test the latency and throughput.<br>
We choose VPNGw1 which the performance is 650Mbps. <br>

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20bw%20Cross%20DIP%20vpn.PNG)

Third, we will setup global VNET peering to test both latency and throughput.

![](https://github.com/yinghli/azure-vm-network-performance/blob/master/VM-VM%20bw%20Cross%20DIP%20peer.PNG)

# Summary

From the below table, accelerate network will improve the end to end network latency.<br>
Lower latency will reduce the CWND when reaching the same level TCP throughput.<br>
New standard load balancer will only add small latency for end to end network latecnty.<br>

Parameters      | VM-VM without Acc | VM-VM with Acc | VM-LB-VM with Acc |VM-SLB-VM with Acc | 
----------------| ------------------|----------------|-------------------|-------------------|
Throughput      | 3.65Gbps          | 3.82Gbps       | 3.28Gbps          | 3.28Gbps          |
CWND            | 3.27MB            | 1.36MB         | 1.31MB            | 1.61MB            |
Latency         | 146us             | 40us           | 81us              | 53us              |

Below the summary of VM to VM cross region test result.
VPN gateway is the bottle neck of the performance. 
Global VNET peer have minimal performance impact comparing with direct PIP connection.

Parameters      | VM-VM with PIP    | VM-VM with VPN | VM-VM with Peer |
----------------| ------------------|----------------|---------------- |
Throughput      | 2.12Gbps          | 549Mbps        | 1.76Gbps        |
CWND            | 24.5MB            | 12.2MB         | 33.7MB          |            
Latency         | 186ms             | 168ms          | 186ms           |


