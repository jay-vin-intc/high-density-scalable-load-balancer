# High-Density-Scalable-Load-Balancer
## Introduction
The High-Density Scalable Load Balancer (HDSLB) is a software implementation of a Layer 4 Load Balancer.  This is a derivative of DPVS which was originally released by iQIYI (https://github.com/iqiyi/dpvs).  DPVS is a software base Layer 4 Load balancer based on the Data Plane Developer Kit and the Linux Virtual Server.

![image](https://user-images.githubusercontent.com/12433658/205763622-d0d1b107-fa6e-4d2b-ab51-bc751b24ac67.png)

DPVS was created to improve the performance of load balancing for software implementations using standard off the shelf compute nodes using 10 Gigabit per second (Gbps) network interface cards (NICs).
HDSLB has used DPVS as the base for creating a modern software implementation for performance targets greater than 10 Gbps.  HDLSB used the architecture of DPVS and made modifications and additions as shown in the red boxes of the diagram.

![image](https://user-images.githubusercontent.com/12433658/205763864-287ef968-6565-4086-9ad1-261b10282612.png)

The objective was to improve performance using NICs with throughputs exceeding 25 Gbps and target performance of 100 Million packets per second while maintaining 100 million sessions per node.  In addition, the throughput should increase linearly with an increase in the utilized CPU cores.
To achieve these objectives HDSLB has refactored DPVS with four modifications
1.	Refactoring the software architecture and accelerating the data path by creating fast path for Parser, Flow Classification, Connection Tracking, Session Creation, Scheduling, Encapsulation and Transmit. A slow path was created for SNAT, ICMP and ARP processing.
2.	Splitting the data path into micro jobs for each function in order to eliminate cache overflows
3.	Vectorizing packet flow for each micro job to enable prefetching to the CPU cache to reduce misses and memory latency
4.	Parsing the 5-tuple across the micro job and the flow classification feature in the Intel Ethernet E810 Network Interface Card to offload flow classification to the NIC without an overflow.

## Quick Start
### Test Environment 

* Linux Distribution: CentOS 8.1 
* Kernel: 4.18.0-147.8.1.el8_1.x86_64 
* CPU: Intel(R) Xeon(R) Gold 6252N CPU @ 2.30GHz 
* NIC: Intel Corporation Ethernet Controller XXV710 for 25GbE SFP28 (rev 02) 
* Memory: 320G with two NUMA nodes 
* GCC: gcc version 8.3.1 20190507 (Red Hat 8.3.1-4)

Other environments should be OK if DPDK works, pls check dpdk.org for more info.
* Pls check this link for NICs supported by DPDK: http://dpdk.org/doc/nics.
* Note flow-director (fdir) is needed for Full-NAT and SNAT mode with multi-cores.

### Clone HDSLB
```
$ git clone https://github.com/intel/high-density-scalable-load-balancer hdslb
$ cd hdslb
```

Well, let's start from DPDK then.

### DPDK setup.
Currently, ```dpdk-19.11.3``` is used for ```HDSLB```.

  You can skip this section if experienced with DPDK, and refer to link for details.

```
$ wget https://fast.dpdk.org/rel/dpdk-19.11.3.tar.xz   # download from dpdk.org if link failed.
$ tar vxf dpdk-19.11.3.tar.xz
```

### DPDK patchs
There's a patch for DPDK ```kni``` driver for hardware multicast, apply it if needed (for example, launch ```ospfd``` on ```kni``` device).

  Assume that we are in DPVS root dir and dpdk-19.11.3 is under it, pls note it's not mandatory, just for convenience.

```
$ cd <path-of-hdslb>
$ cp patch/dpdk-stable-19.11/*.patch dpdk-19.11.3/
$ cd dpdk-19.11.3/
$ patch -p 1 < 0002-support-large_memory.patch
$ patch -p 1 < 0003-net-i40e-support-rx-markid-ofb.patch  
```


### DPDK build and install
Now build DPDK and export ```RTE_SDK``` env variable for DPDK app (DPVS).
```
$ cd dpdk-19.11.3/
$ make config T=x86_64-native-linuxapp-gcc
Configuration done
$ make # or "make -j40" to save time, where 40 is the cpu core number.
$ export RTE_SDK=$PWD
```

In our tutorial, ```RTE_TARGET``` is not set, the value is "build" by default, thus DPDK libs and header files can be found in ```dpdk-19.11.3/build```.
Now to set up DPDK hugepage, our test environment is NUMA system. For single-node system pls refer to [link](http://doc.dpdk.org/guides/linux_gsg/sys_reqs.html).
```
$ # for NUMA machine
$ echo 150 > /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages
$ echo 150 > /sys/devices/system/node/node1/hugepages/hugepages-1048576kB/nr_hugepages

$ mkdir /mnt/huge
$ mount -t hugetlbfs nodev /mnt/huge
```

Install kernel modules and bind NIC with ```vfio-pci``` driver. We use 2 NICs for Full-NAT cluster. Assuming ```eth0``` will be used for HDSLB/DPDK, and another standalone Linux NIC for debug, for example, ```eth1```.
```
$ modprobe vfio-pci
$ cd dpdk-19.11.3

$ insmod build/kmod/rte_kni.ko

$ ./usertools/dpdk-devbind.py --status
$ ifconfig eth0 down  # assuming eth0 is 0000:18:00.0
$ ifconfig eth1 down  # assuming eth1 is 0000:1a:00.0
$ ./usertools/dpdk-devbind.py -b vfio-pci 0000:18:00.0 0000:1a:00.0
```

```dpdk-devbind.py -u``` can be used to unbind driver and switch it back to Linux driver like ```i40e```. You can also use ```lspci``` or ```ethtool -i eth0``` to check the NIC PCI bus-id. Pls see [DPDK](https://www.dpdk.org/) site for details.

### Build HDSLB
It's simple, just set ```RTE_SDK``` and build it.
```
$ cd dpdk-19.11.3/
$ export RTE_SDK=$PWD
$ cd <path-of-hdslb>
Note: If use 7-12 cores to support 100M session. Please set CFLAGS `DPVS_MAX_LADDR_PERCORE=256` and `LB_CONN_POOL_SIZE_PERCORE=16777216` in file `src/config.mk`.
$ make # or "make -j40" to speed up.
$ make install
```


  May need to install dependencies, like ```openssl```, ```popt``` and ```numactl```, e.g., ```yum install popt-devel``` (CentOS).

Output files are installed to ```hdslb/bin```.
```
$ ls bin/
dpip  hdslb  ipvsadm  keepalived
```

* ```hdslb``` is the main program.
* ```dpip``` is the tool to set IP address, route, vlan, neigh etc.
* ```ipvsadm``` and ```keepalived``` come from LVS, both are modified.


### Launch HDSLB
Now, ```hdslb.conf``` must be put at ```/etc/hdslb.conf```, just copy it from ```conf/hdslb.conf.sample```.
```
$ cp conf/hdslb.conf.sample /etc/hdslb.conf
```
and start DPVS,
```
$ cd <path-of-hdslb>/bin
$ ./hdslb &
```
Check if it's get started:
```
$ ./dpip link show
1: dpdk0: socket 0 mtu 1500 rx-queue 8 tx-queue 8
    UP 10000 Mbps full-duplex fixed-nego promisc-off
    addr A0:36:9F:9D:61:F4 OF_RX_IP_CSUM OF_TX_IP_CSUM OF_TX_TCP_CSUM OF_TX_UDP_CSUM
```
If you see this message, well done, ```DPVS``` is working with NIC ```dpdk0```!

> Don't worry if you see this error,
```
EAL: Error - exiting with code: 1
  Cause: ports in DPDK RTE (2) != ports in hdslb.conf(1)
```

It means the NIC used by DPVS is not matching /etc/hdslb.conf. Pls use dpdk-devbind to adjust the NIC number or modify hdslb.conf. We'll improve this part to make DPVS smarter to avoid modify config file when NIC count is not match.

### Test Full-NAT Load Balancer
The test topology:
![image](https://user-images.githubusercontent.com/12433658/206008673-c3469c94-4147-4a94-b5e5-5916956697a5.png)

The setting includes:

* ip-addresses and routes for DPDK LAN/WAN network.
* VIP on WAN interface (dpdk1)
* FNAT service (vip:vport) and related RS
* FNAT mode needs at least one LIP on LAN interface (dpdk0)
```
#!/bin/sh -

# add VIP to WAN interface
./dpip addr add 10.0.0.100/32 dev dpdk1

# route for WAN/LAN access
# add routes for other network or default route if needed.
./dpip route add 10.0.0.0/16 dev dpdk1
./dpip route add 192.168.100.0/24 dev dpdk0

# add service <VIP:vport> to forwarding, scheduling mode is RR.
# use ipvsadm --help for more info.
./ipvsadm -A -t 10.0.0.100:80 -s rr

# add two RS for service, forwarding mode is FNAT (-b)
./ipvsadm -a -t 10.0.0.100:80 -r 192.168.100.2 -b
./ipvsadm -a -t 10.0.0.100:80 -r 192.168.100.3 -b

# add at least one Local-IP (LIP) for FNAT on LAN interface
./ipvsadm --add-laddr -z 192.168.100.200 -t 10.0.0.100:80 -F dpdk0
```
And you can use the commands below to check what's just set:
```
$ ./dpip addr show
inet 10.0.0.100/32 scope global dpdk1
     valid_lft forever preferred_lft forever
inet 192.168.100.200/32 scope global dpdk0
     valid_lft forever preferred_lft forever sa_used 0 sa_free 1032176 sa_miss 0
```
```
$ ./dpip route show
inet 10.0.0.100/32 via 0.0.0.0 src 0.0.0.0 dev dpdk1 mtu 1500 tos 0 scope host metric 0 proto auto
inet 192.168.100.200/32 via 0.0.0.0 src 0.0.0.0 dev dpdk0 mtu 1500 tos 0 scope host metric 0 proto auto
inet 192.168.100.0/24 via 0.0.0.0 src 0.0.0.0 dev dpdk0 mtu 1500 tos 0 scope link metric 0 proto auto
inet 10.0.0.0/16 via 0.0.0.0 src 0.0.0.0 dev dpdk1 mtu 1500 tos 0 scope link metric 0 proto auto
```
```
$ ./ipvsadm  -ln
IP Virtual Server version 0.0.0 (size=0)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.0.0.100:80 rr
  -> 192.168.100.2:80             FullNat 1      0          0
  -> 192.168.100.3:80             FullNat 1      0          0
```
```
$ ./ipvsadm  -G
VIP:VPORT            TOTAL    SNAT_IP              CONFLICTS  CONNS
10.0.0.100:80        1
                              192.168.100.200      0          0
```
And now to verify if FNAT (two-arm) works. I've setup Nginx server on RS (with TOA module) to response the HTTP request with Client's real IP and port. The response format is plain text (not html).
```
client$ curl 10.0.0.100
Your ip:port : 10.0.0.48:37177
```

### Configure Tutorial
More configure examples can be found in the [Tutorial Document](http://github.com/intc/high-density-scalable-load-balancer/master/doc/tutorial.md). Including:

* WAN-to-LAN ```Full-NAT``` reverse proxy.
* Direct Route (```DR```) mode setup.
* Master/Backup model (```keepalived```) setup.
* OSPF/ECMP cluster model setup.
* ```SNAT``` mode for Internet access from internal network.
* Virtual Devices (```Bonding```, ```VLAN```, ```kni```, ```ipip/GRE```).
* ```UOA``` module to get real UDP client IP/port in ```FNAT```.
* ... and more ...
