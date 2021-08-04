Linux Network Namespaces:
--------------------------
**Namespaces:**
Namespaces are a feature of the Linux kernel that partitions kernel resources such that one set of processes sees one set of resources while another set of processes sees a different set of resources. The feature works by having the same namespace for a set of resources and processes, but those namespaces refer to distinct resources. Resources may exist in multiple spaces. Examples of such resources are process IDs, hostnames, user IDs, file names, and some names associated with network access, and interprocess communication.

**Linux Network Namespaces:**
In a network namespace, the scoped ‘identifiers’ are network devices; so a given network device, such as eth0, exists in a particular namespace. Linux starts up with a default network namespace, so if your operating system does not do anything special, that is where all the network devices will be located. But it is also possible to create further non-default namespaces, and create new devices in those namespaces, or to move an existing device from one namespace to another.

Each network namespace also has its own routing table, and in fact this is the main reason for namespaces to exist. A routing table is keyed by destination IP address, so network namespaces are what you need if you want the same destination IP address to mean different things at different times - which is something that OpenStack Networking requires for its feature of providing overlapping IP addresses in different virtual networks.

Each network namespace also has its own set of iptables (for both IPv4 and IPv6). So, you can apply different security to flows with the same IP addressing in different namespaces, as well as different routing.

I will do a 3-part lab here to explain the situation:

* [Lab - 1](lab1.md)
  - Create a namespace.
  - Create peer network interfaces
  - Ping between global namespace and local namespace
* Lab - 2
* Lab - 3

Now we will go to netns1 namespace with sh shell.
```
$ sudo ip netns exec netns1 sh
$ ip route
$ route -n
$ ping 172.20.0.11
$ ping 172.20.0.1
$ arp -a
```
We will do the same for another network namespace.
```
$ sudo ip netns add netns2
$ sudo ip link add veth2 type veth peer name ceth2 netns netns2
$ sudo ip netns exec netns2 ip addr add 172.20.0.12/16 dev ceth2
$ sudo ip netns exec netns2 ip link set dev ceth2 up
$ sudo ip netns exec netns2 ip link set lo up
$ sudo ip addr add 172.20.0.2/16 dev veth2
$ sudo ip link set dev veth2 up
$ ping 172.20.0.12 -I veth2
```

**NOT REQUIRED TO READ BELOW: TOO MUCH DETAILS**
```
[root@pelican ~]# sudo ip netns exec netns2 ping -c 3 172.20.0.2
PING 172.20.0.2 (172.20.0.2) 56(84) bytes of data.

--- 172.20.0.2 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2097ms

[root@pelican ~]# sudo ip netns exec netns2 ip addr show ceth2
7: ceth2@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 5a:27:9b:e3:82:53 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.20.0.12/16 scope global ceth2
       valid_lft forever preferred_lft forever
    inet6 fe80::5827:9bff:fee3:8253/64 scope link
       valid_lft forever preferred_lft forever
[root@pelican ~]#
[root@pelican ~]# sudo ip addr show veth2
8: veth2@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 42:43:18:bf:4e:06 brd ff:ff:ff:ff:ff:ff link-netns netns2
    inet 172.20.0.2/16 scope global veth2
       valid_lft forever preferred_lft forever
    inet6 fe80::4043:18ff:febf:4e06/64 scope link
       valid_lft forever preferred_lft forever
[root@pelican ~]#
[root@pelican ~]# sudo ip addr show veth1
6: veth1@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 26:17:84:f3:08:23 brd ff:ff:ff:ff:ff:ff link-netns netns1
    inet 172.20.0.1/16 scope global veth1
       valid_lft forever preferred_lft forever
    inet6 fe80::2417:84ff:fef3:823/64 scope link
       valid_lft forever preferred_lft forever
[root@pelican ~]#


[root@pelican ~]# tcpdump -ennqtvv -i veth2 \(arp or icmp\)
tcpdump: listening on veth2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
5a:27:9b:e3:82:53 > 42:43:18:bf:4e:06, IPv4, length 98: (tos 0x0, ttl 64, id 52090, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.12 > 172.20.0.2: ICMP echo request, id 25539, seq 1, length 64
5a:27:9b:e3:82:53 > 42:43:18:bf:4e:06, IPv4, length 98: (tos 0x0, ttl 64, id 52193, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.12 > 172.20.0.2: ICMP echo request, id 25539, seq 2, length 64
5a:27:9b:e3:82:53 > 42:43:18:bf:4e:06, IPv4, length 98: (tos 0x0, ttl 64, id 52232, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.12 > 172.20.0.2: ICMP echo request, id 25539, seq 3, length 64
5a:27:9b:e3:82:53 > 42:43:18:bf:4e:06, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.2 tell 172.20.0.12, length 28
42:43:18:bf:4e:06 > 5a:27:9b:e3:82:53, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Reply 172.20.0.2 is-at 42:43:18:bf:4e:06, length 28

[root@pelican ~]# tcpdump -ennqtvv -i veth1 \(arp or icmp\)
tcpdump: listening on veth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
26:17:84:f3:08:23 > ff:ff:ff:ff:ff:ff, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.12 tell 172.20.0.2, length 28
26:17:84:f3:08:23 > ff:ff:ff:ff:ff:ff, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.12 tell 172.20.0.2, length 28
26:17:84:f3:08:23 > ff:ff:ff:ff:ff:ff, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.12 tell 172.20.0.2, length 28
^C

[root@pelican ~]# ip route
default via 10.0.2.2 dev enp0s3 proto dhcp metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
172.20.0.0/16 dev veth1 proto kernel scope link src 172.20.0.1 <----------------- This route is in effect.
172.20.0.0/16 dev veth2 proto kernel scope link src 172.20.0.2 <----------------- This route is NOT in effect.
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.21 metric 101

```
I am trying to ping from `netns2` to the `veth2` IP. Ping is failing. We see from tcpdump output, `veth2` is getting ICMP echo reqeust from `ceth2` but since global namespace (owner of `veth2`) does not know what's the MAC address of `ceth2`, it's broadcasting MAC address query packet but the packets are going through `veth1`. It's happening because, the current route table is saying that packets to 172.20.0.0/16 should go through veth1. Since the 1st rule matches, the 2nd rule is not matched.

**NO NEED TO READ TILL THE ABOVE LINES**



**Create a bridge device naming it `br0` and set it up.**
  We do not need IP address for veth1 & veth2 because we are going to create bridge interface br0 and we will Add veth1 & veth2 interface to the bridge by setting the bridge device as their master.
  * Remove IP address for veth1 & veth
```
 $ sudo ip addr del 172.20.0.1/16  dev veth1
 $ sudo ip addr del 172.20.0.2/16  dev veth2

```
Create a bridge device:
```
$ sudo ip link add br0 type bridge
$ sudo ip link set br0 up
```
  **Add the veth1 & veth2 interface to the bridge by setting the bridge device as their master.**
```
$ sudo ip link set veth1 master br0
$ sudo ip link set veth2 master br0
```
  **Set the address of the `br0` interface (bridge device)**
```
$ bridge link show br0
$ sudo ip addr add 172.20.0.10/16 brd + dev br0
```
Now we can ping both ways.
```
$ ping -c 2 172.20.0.11
PING 172.20.0.11 (172.20.0.11) 56(84) bytes of data.
64 bytes from 172.20.0.11: icmp_seq=1 ttl=64 time=0.047 ms
64 bytes from 172.20.0.11: icmp_seq=2 ttl=64 time=0.041 ms

--- 172.20.0.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1059ms
rtt min/avg/max/mdev = 0.041/0.044/0.047/0.003 ms


# ping -c 2 172.20.0.12
PING 172.20.0.12 (172.20.0.12) 56(84) bytes of data.
64 bytes from 172.20.0.12: icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from 172.20.0.12: icmp_seq=2 ttl=64 time=0.099 ms

--- 172.20.0.12 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1033ms
rtt min/avg/max/mdev = 0.050/0.074/0.099/0.024 ms


$ sudo ip netns exec netns1 ping -c 2 172.20.0.10
PING 172.20.0.10 (172.20.0.10) 56(84) bytes of data.
64 bytes from 172.20.0.10: icmp_seq=1 ttl=64 time=0.052 ms
64 bytes from 172.20.0.10: icmp_seq=2 ttl=64 time=0.063 ms

--- 172.20.0.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1035ms
rtt min/avg/max/mdev = 0.052/0.057/0.063/0.005 ms


$ sudo ip netns exec netns2 ping -c 2 172.20.0.10
PING 172.20.0.10 (172.20.0.10) 56(84) bytes of data.
64 bytes from 172.20.0.10: icmp_seq=1 ttl=64 time=0.049 ms
64 bytes from 172.20.0.10: icmp_seq=2 ttl=64 time=0.057 ms

--- 172.20.0.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1053ms
rtt min/avg/max/mdev = 0.049/0.053/0.057/0.004 ms


#Pinging from netns1 to netns2
$ sudo ip netns exec netns1 ping -c 2 172.20.0.12
PING 172.20.0.12 (172.20.0.12) 56(84) bytes of data.
64 bytes from 172.20.0.12: icmp_seq=1 ttl=64 time=0.130 ms
64 bytes from 172.20.0.12: icmp_seq=2 ttl=64 time=0.084 ms

--- 172.20.0.12 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 19ms
rtt min/avg/max/mdev = 0.084/0.107/0.130/0.023 ms


#Pinging from netns1 to netns2
$ sudo ip netns exec netns2 ping -c 2 172.20.0.11
PING 172.20.0.11 (172.20.0.11) 56(84) bytes of data.
64 bytes from 172.20.0.11: icmp_seq=1 ttl=64 time=0.110 ms
64 bytes from 172.20.0.11: icmp_seq=2 ttl=64 time=0.043 ms

--- 172.20.0.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.043/0.076/0.110/0.034 ms

#If pinging between 2 namespaces are failing, check if there is forward policy set to drop.
#iptables -L -v

$ sudo ip route
default via 10.0.2.2 dev enp0s3 proto dhcp metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
172.20.0.0/16 dev br0 proto kernel scope link src 172.20.0.10 <---------------------- This is the one route
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.21 metric 101
```

**Now we try pinging another IP of global namespace**
```

```
```
[root@pelican ~]# ip netns exec netns1 ping -c 2 10.0.2.2
ping: connect: Network is unreachable
```
The namespace `netns1` does not have the route.
```
[root@pelican ~]# ip netns exec netns1 ip route
172.20.0.0/16 dev ceth1 proto kernel scope link src 172.20.0.11
```
**So we add the default gateway in all the network namespace.**
```
$ sudo ip netns exec netns1 ip route add default via 172.20.0.10
$ sudo ip netns exec netns2 ip route add default via 172.20.0.10
```


* Set us up to have responses from the network.
* -t specifies the table to which the commands should be directed to. By default, it's `filter`.
* -A specifies that we're appending a rule to the chain that we tell the name after it.
* -s specifies a source address (with a mask in this case).
* -j specifies the target to jump to (what action to take).
```
$ sudo iptables -t nat -A POSTROUTING -s 172.20.0.0/16 -j MASQUERADE
```
**Enable packet forwarding**
```
$ sudo sysctl -w net.ipv4.ip_forward=1
sujit@srv:~$ sudo ip netns exec netns2 ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=127 time=30.3 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=127 time=27.9 ms
Now (finally), we’re good! We have connectivity all the way:
```

* the host can direct traffic to an application inside a namespace;
* an application inside a namespace can direct traffic to an application in the host;
* an application inside a namespace can direct traffic to another application in another namespace; and
* an application inside a namespace can access the internet.


Reference:
  https://ops.tips/blog/using-network-namespaces-and-bridge-to-isolate-servers/
  https://unix.stackexchange.com/questions/524052/how-to-connect-a-namespace-to-a-physical-interface-through-a-bridge-and-veth-pai
