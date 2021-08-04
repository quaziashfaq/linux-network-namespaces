# Lab 1

## Create namespace and veth pairs

Creating and Listing Network Namespaces
```
  $ sudo ip netns add netns1
```

To verify that the network namespace has been created, use this command:
```
$ sudo ip netns list
netns1
```

We should see your network namespace `netns1` listed there, ready for you to use.

Assigning Interfaces to Network Namespaces:
```
  $ sudo ip link add veth1 type veth peer name ceth1
```

We can verify that the veth pair was created using this command:
```
$ sudo ip link list
7: ceth1@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 42:15:05:91:3e:98 brd ff:ff:ff:ff:ff:ff
8: veth1@ceth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 62:70:b6:31:37:04 brd ff:ff:ff:ff:ff:ff
```

We should see a pair of veth interfaces (using the names you assigned in the command above) listed there. Right now, they both belong to the “default” or “global” namespace, along with the physical interfaces.

Let’s say that We want to connect the global namespace to the netns1 namespace. To do that, you’ll need to move one of the veth interfaces to the netns1 namespace using this command:
```
$ sudo ip link set ceth1 netns netns1
```

If we then run the ip link list command again, we will see that the ceth1 interface has disappeared from the list. It’s now in the netns1 namespace, so to see it you’d need to run this command:
```
# ip netns exec netns1 ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
7: ceth1@if8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 42:15:05:91:3e:98 brd ff:ff:ff:ff:ff:ff link-netnsid 0

# ip netns exec netns1 ip addr show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
7: ceth1@if8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 42:15:05:91:3e:98 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```
Now we are able to see ceth1 in the list along with loopback interface(lo). Both link state is down.

From the global namespace we can see that `veth1` is connected to namespace `netns1`.
```
root@debian:~/scripts/namespaces# ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:b3:96:d0 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 65166sec preferred_lft 65166sec
    inet6 fe80::a00:27ff:feb3:96d0/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:3c:31:79 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.11/24 brd 192.168.56.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe3c:3179/64 scope link
       valid_lft forever preferred_lft forever
8: veth1@if7: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 62:70:b6:31:37:04 brd ff:ff:ff:ff:ff:ff link-netns netns1
```


## Configuring Interface IP in Network Namespaces

Configuring IP Address on veth1 in  “default” or “global” Namespaces
```
$ sudo ip addr add 172.20.0.1/16 dev veth1
$ sudo ip link set dev veth1 up
root@debian:~/scripts/namespaces# ip addr show veth1
8: veth1@if7: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether 62:70:b6:31:37:04 brd ff:ff:ff:ff:ff:ff link-netns netns1
    inet 172.20.0.1/16 scope global veth1
       valid_lft forever preferred_lft forever
```
LOWERLAYERDOWN might mean that its peer interface `ceth1` is not up yet.

Now we configure IP in `netns1` and set their state up
```
$ sudo ip netns exec netns1 ip addr add 172.20.0.11/16 dev ceth1
$ sudo ip netns exec netns1 ip link set dev ceth1 up
$ sudo  ip netns exec netns1 ip link set lo up
```

Now we check the ip config and link state in global and `netns1` namespace
In global:
```
root@debian:~/scripts/namespaces# ip addr show veth1
8: veth1@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 62:70:b6:31:37:04 brd ff:ff:ff:ff:ff:ff link-netns netns1
    inet 172.20.0.1/16 scope global veth1
       valid_lft forever preferred_lft forever
    inet6 fe80::6070:b6ff:fe31:3704/64 scope link
       valid_lft forever preferred_lft forever
```

In `netns1` namespace
```
root@debian:~/scripts/namespaces# ip netns exec netns1 ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
7: ceth1@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 42:15:05:91:3e:98 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.20.0.11/16 scope global ceth1
       valid_lft forever preferred_lft forever
    inet6 fe80::4015:5ff:fe91:3e98/64 scope link
       valid_lft forever preferred_lft forever
```

## Now we check the communication between global and `netns1` namespace:

From global to `netns1`
```
root@debian:~/scripts/namespaces# ping -c 2 172.20.0.11
PING 172.20.0.11 (172.20.0.11) 56(84) bytes of data.
64 bytes from 172.20.0.11: icmp_seq=1 ttl=64 time=0.147 ms
64 bytes from 172.20.0.11: icmp_seq=2 ttl=64 time=0.037 ms

--- 172.20.0.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 33ms
rtt min/avg/max/mdev = 0.037/0.092/0.147/0.055 ms
```

From `netns1` namespace to global
```
# ip netns exec netns1 ping -c 2 172.20.0.1
PING 172.20.0.1 (172.20.0.1) 56(84) bytes of data.
64 bytes from 172.20.0.1: icmp_seq=1 ttl=64 time=0.039 ms
64 bytes from 172.20.0.1: icmp_seq=2 ttl=64 time=0.043 ms

--- 172.20.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1077ms
rtt min/avg/max/mdev = 0.039/0.041/0.043/0.002 ms
```

From global namespace, we check the tcpdump output
```
# tcpdump -ennvvq -i veth1
tcpdump: listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
22:50:31.417415 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 50414, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 172.20.0.1: ICMP echo request, id 3947, seq 1, length 64
22:50:31.417429 62:70:b6:31:37:04 > 42:15:05:91:3e:98, IPv4, length 98: (tos 0x0, ttl 64, id 33732, offset 0, flags [none], proto ICMP (1), length 84)
    172.20.0.1 > 172.20.0.11: ICMP echo reply, id 3947, seq 1, length 64
22:50:32.434849 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 50481, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 172.20.0.1: ICMP echo request, id 3947, seq 2, length 64
22:50:32.434891 62:70:b6:31:37:04 > 42:15:05:91:3e:98, IPv4, length 98: (tos 0x0, ttl 64, id 33846, offset 0, flags [none], proto ICMP (1), length 84)
    172.20.0.1 > 172.20.0.11: ICMP echo reply, id 3947, seq 2, length 64
22:50:36.594756 62:70:b6:31:37:04 > 42:15:05:91:3e:98, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.11 tell 172.20.0.1, length 28
22:50:36.595052 42:15:05:91:3e:98 > 62:70:b6:31:37:04, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.1 tell 172.20.0.11, length 28
22:50:36.595066 62:70:b6:31:37:04 > 42:15:05:91:3e:98, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Reply 172.20.0.1 is-at 62:70:b6:31:37:04, length 28
22:50:36.595071 42:15:05:91:3e:98 > 62:70:b6:31:37:04, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Reply 172.20.0.11 is-at 42:15:05:91:3e:98, length 28
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel
```

## Now we ping other interfaces of the global namespaces
First, we check the `iptables` rules. There are 5 tables
- filter
- nat
- mangle
- raw
- security

Check your tables with this command:
```
$ sudo iptables -t <table_name> -L -v
```

My machine is a fresh debian box. It has all the default policies. No customization done. Therefore, it will be easier to understand.

In my global namespace, there are 2 NICs and 2 IPs are attached here.
| NIC    | IP            | GW       |
|--------|---------------|----------|
| enp0s3 | 10.0.2.15     | 10.0.2.2 |
| enp0s8 | 192.168.56.21 |          |

Now I will try to ping 192.168.56.21 from `netns1` namespace, but it will fail because there is no route defined inside. Let's see:
```
# sudo ip netns exec netns1 ping 192.168.56.21
ping: connect: Network is unreachable

# sudo ip netns exec netns1 ip route
172.20.0.0/16 dev ceth1 proto kernel scope link src 172.20.0.11
```

Now I add the route and I can ping
```
$ sudo ip netns exec netns1 ip route add default via 172.20.0.1

$ sudo ip netns exec netns1 ip route
default via 172.20.0.1 dev ceth1
172.20.0.0/16 dev ceth1 proto kernel scope link src 172.20.0.11
```

Now we ping and we check the `tcpdump` output and also check the physical (mac) addresses to understand which NIC is replying.
All IP's can be thought as logical names of the machine (i.e. global namespace). So when `netns1` is pinging any IP, the global namespace is going to reply back from `veth1`.

Global Namespace IP and MAC addresses:
```
# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:b3:96:d0 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 61683sec preferred_lft 61683sec
    inet6 fe80::a00:27ff:feb3:96d0/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:3c:31:79 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.11/24 brd 192.168.56.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe3c:3179/64 scope link
       valid_lft forever preferred_lft forever
8: veth1@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 62:70:b6:31:37:04 brd ff:ff:ff:ff:ff:ff link-netns netns1
    inet 172.20.0.1/16 scope global veth1
       valid_lft forever preferred_lft forever
    inet6 fe80::6070:b6ff:fe31:3704/64 scope link
       valid_lft forever preferred_lft forever
```

`netns1` interface IP and MAC address:
```
root@debian:~/scripts/namespaces# ip netns exec netns1 ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
7: ceth1@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 42:15:05:91:3e:98 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.20.0.11/16 scope global ceth1
       valid_lft forever preferred_lft forever
    inet6 fe80::4015:5ff:fe91:3e98/64 scope link
       valid_lft forever preferred_lft forever
```

Pinging from `netns1`(172.20.0.11) to 192.168.56.11:
```
root@debian:~/scripts/namespaces# ip netns exec netns1 ping -c 2 192.168.56.11
PING 192.168.56.11 (192.168.56.11) 56(84) bytes of data.
64 bytes from 192.168.56.11: icmp_seq=1 ttl=64 time=0.035 ms
64 bytes from 192.168.56.11: icmp_seq=2 ttl=64 time=0.052 ms

--- 192.168.56.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 3ms
rtt min/avg/max/mdev = 0.035/0.043/0.052/0.010 ms
```

Tcpdump output in `netns1`:
```
root@debian:~# ip netns exec netns1 tcpdump -ennvvq -i ceth1
tcpdump: listening on ceth1, link-type EN10MB (Ethernet), capture size 262144 bytes
^C23:33:48.168381 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 4732, offset 0, flags [DF], proto ICMP
(1), length 84)
    172.20.0.11 > 192.168.56.11: ICMP echo request, id 4006, seq 1, length 64
23:33:48.168401 62:70:b6:31:37:04 > 42:15:05:91:3e:98, IPv4, length 98: (tos 0x0, ttl 64, id 190, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.56.11 > 172.20.0.11: ICMP echo reply, id 4006, seq 1, length 64
23:33:49.170723 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 4926, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.11: ICMP echo request, id 4006, seq 2, length 64
23:33:49.170752 62:70:b6:31:37:04 > 42:15:05:91:3e:98, IPv4, length 98: (tos 0x0, ttl 64, id 276, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.56.11 > 172.20.0.11: ICMP echo reply, id 4006, seq 2, length 64
```

Tcpdump output from `veth1` in `global` namespace:
```
root@debian:~# tcpdump -ennvvq -i veth1
tcpdump: listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
23:33:48.168387 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 4732, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.11: ICMP echo request, id 4006, seq 1, length 64
23:33:48.168400 62:70:b6:31:37:04 > 42:15:05:91:3e:98, IPv4, length 98: (tos 0x0, ttl 64, id 190, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.56.11 > 172.20.0.11: ICMP echo reply, id 4006, seq 1, length 64
23:33:49.170732 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 4926, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.11: ICMP echo request, id 4006, seq 2, length 64
23:33:49.170751 62:70:b6:31:37:04 > 42:15:05:91:3e:98, IPv4, length 98: (tos 0x0, ttl 64, id 276, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.56.11 > 172.20.0.11: ICMP echo reply, id 4006, seq 2, length 64
23:33:53.202740 62:70:b6:31:37:04 > 42:15:05:91:3e:98, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.11 tell 172.20.0.1, length 28
23:33:53.202853 42:15:05:91:3e:98 > 62:70:b6:31:37:04, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.1 tell 172.20.0.11, length 28
23:33:53.202861 62:70:b6:31:37:04 > 42:15:05:91:3e:98, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Reply 172.20.0.1 is-at 62:70:b6:31:37:04, length 28
23:33:53.202864 42:15:05:91:3e:98 > 62:70:b6:31:37:04, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Reply 172.20.0.11 is-at 42:15:05:91:3e:98, length 28
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel
```

Now here is a table of interfaces, their mac addresses and assigned IPs:
| Interface | MAC Address       | IP            |
|-----------|-------------------|---------------|
| enp0s3    | 08:00:27:b3:96:d0 | 10.0.2.15     |
| enp0s8    | 08:00:27:3c:31:79 | 192.168.56.21 |
| veth1     | 62:70:b6:31:37:04 | 172.20.0.1    |
| ceth1     | 42:15:05:91:3e:98 | 172.20.0.11   |

Here we see that though the ICMP reqesut is coming from ceth1 to the IP `192.168.56.11`, which is assigned to `enp0s8`, the reply frame was coming from `veth1(62:70:b6:31:37:04)`.


## Now we will try to ping a IP out of the global namespace.
| Interface | MAC Address       | IP            | IP out of machine |
|-----------|-------------------|---------------|-------------------|
| enp0s3    | 08:00:27:b3:96:d0 | 10.0.2.15     | 10.0.2.2          |
| enp0s8    | 08:00:27:3c:31:79 | 192.168.56.21 | 192.168.56.1      |
| veth1     | 62:70:b6:31:37:04 | 172.20.0.1    |                   |
| ceth1     | 42:15:05:91:3e:98 | 172.20.0.11   |                   |

Now we try to ping 192.168.56.1. But it fails.
```
root@debian:~/scripts/namespaces# ip netns exec netns1 ping -c 3 192.168.56.1
PING 192.168.56.1 (192.168.56.1) 56(84) bytes of data.

--- 192.168.56.1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 41ms

root@debian:~# ip netns exec netns1 tcpdump -ennvvq -i ceth1
tcpdump: listening on ceth1, link-type EN10MB (Ethernet), capture size 262144 bytes
00:01:47.419576 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 49964, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.1: ICMP echo request, id 4032, seq 1, length 64
00:01:48.434745 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 50163, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.1: ICMP echo request, id 4032, seq 2, length 64
00:01:49.458771 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 50270, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.1: ICMP echo request, id 4032, seq 3, length 64
00:01:52.562704 42:15:05:91:3e:98 > 62:70:b6:31:37:04, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.1 tell 172.20.0.11, length 28
00:01:52.562759 62:70:b6:31:37:04 > 42:15:05:91:3e:98, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Reply 172.20.0.1 is-at 62:70:b6:31:37:04, length 28


root@debian:~# tcpdump -ennvvq -i veth1
tcpdump: listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
00:01:47.419581 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 49964, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.1: ICMP echo request, id 4032, seq 1, length 64
00:01:48.434756 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 50163, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.1: ICMP echo request, id 4032, seq 2, length 64
00:01:49.458780 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 50270, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.1: ICMP echo request, id 4032, seq 3, length 64
00:01:52.562715 42:15:05:91:3e:98 > 62:70:b6:31:37:04, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.1 tell 172.20.0.11, length 28
00:01:52.562757 62:70:b6:31:37:04 > 42:15:05:91:3e:98, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Reply 172.20.0.1 is-at 62:70:b6:31:37:04, length 28


root@debian:~# tcpdump -ennvvq -i enp0s8 \(arp or icmp \)
tcpdump: listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel

```
The ICMP packet is going through `ceth1` to `veth1`. But it's not going out through `enp0s8` to `192.168.56.1` (to another machine).

### Troubleshooting

So I have to tell Linux to pass my packets from the internal namespace to enp0s8. We have an agent to do this kind of stuff. It's called `netfilter`. Actually every network packets/segments go through this `netfilter` agent. We can make `netfilter` do many tasks with `iptables` command. 

There are 5 tables. Each table got a number of CHAIN of rules. I need to read more about it and make a separate notes.
- filter
- nat
- mangle
- raw
- security

Shortly, at first, we need to add some rule that will forward my packet to/through enp0s8.

root@debian:~# iptables -t nat -L -v
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    3   252 MASQUERADE  all  --  any    any     172.20.0.0/16        anywhere

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination


**Set us up to have responses from the network.**
```
* -t specifies the table to which the commands should be directed to. By default, it's `filter`.
* -A specifies that we're appending a rule to the chain that we tell the name after it.
* -s specifies a source address (with a mask in this case).
* -j specifies the target to jump to (what action to take).
```

Here is the command to add SNAT routing
```
$ sudo iptables -t nat -A POSTROUTING -s 172.20.0.0/16 -j MASQUERADE


root@debian:~# iptables -t nat -L -v
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```


By default, the IPv4 policy in Linux kernel disables support for IP forwarding, which prevents boxes running Red Hat Enterprise Linux from functioning as dedicated edge routers. 

To check the current value, run the following command:
```
# sysctl -n net.ipv4.ip_forward
0
```

To enable IP forwarding, run the following command:
```
# sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
```

If this command is run via shell prompt, then the setting is not remembered after a reboot. To permanently set forwarding, edit  the /etc/sysctl.conf file. Find and edit the following line, replacing 0 with 1:
```
net.ipv4.ip_forward=1
```

Execute the following command to enable the change to the sysctl.conf file:
```
# sysctl -p /etc/sysctl.conf
```

Now it's working.
At first, the interface details for reference:

| Interface | MAC Address       | IP            | Connected to      | IP out of machine |
|-----------|-------------------|---------------|-------------------|-------------------|
| vboxnet0  | 0a:00:27:00:00:00 | 192.168.56.1  | Laptop            |                   |
| enp0s3    | 08:00:27:b3:96:d0 | 10.0.2.15     | VM in VirtualBox  | 10.0.2.2          |
| enp0s8    | 08:00:27:3c:31:79 | 192.168.56.21 | VM in VirtualBox  | 192.168.56.1      |
| veth1     | 62:70:b6:31:37:04 | 172.20.0.1    | VM in VirtualBox  |                   |
| ceth1     | 42:15:05:91:3e:98 | 172.20.0.11   | `netns` namespace |                   |


Here is the output.

Pinging the my laptop's IP (192.168.56.1) from `netns1`
```
root@debian:~/scripts/namespaces# ip netns exec netns1 ping -c 2 192.168.56.1
PING 192.168.56.1 (192.168.56.1) 56(84) bytes of data.
64 bytes from 192.168.56.1: icmp_seq=1 ttl=63 time=0.248 ms
64 bytes from 192.168.56.1: icmp_seq=2 ttl=63 time=0.264 ms

--- 192.168.56.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 23ms
rtt min/avg/max/mdev = 0.248/0.256/0.264/0.008 ms
```

Here I am monitoring `netns1` interface `ceth1`. The echo reply from 192.168.56.1 is coming through the interface `veth1`.
```
root@debian:~# ip netns exec netns1 tcpdump -ennvvq -i ceth1 -l | tee
tcpdump: listening on ceth1, link-type EN10MB (Ethernet), capture size 262144 bytes
21:03:11.956640 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 44249, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.1: ICMP echo request, id 4791, seq 1, length 64
21:03:11.956876 62:70:b6:31:37:04 > 42:15:05:91:3e:98, IPv4, length 98: (tos 0x0, ttl 63, id 3779, offset 0, flags [none], proto ICMP
(1), length 84)
    192.168.56.1 > 172.20.0.11: ICMP echo reply, id 4791, seq 1, length 64
21:03:12.979069 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 44262, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.1: ICMP echo request, id 4791, seq 2, length 64
21:03:12.979315 62:70:b6:31:37:04 > 42:15:05:91:3e:98, IPv4, length 98: (tos 0x0, ttl 63, id 3825, offset 0, flags [none], proto ICMP
(1), length 84)
    192.168.56.1 > 172.20.0.11: ICMP echo reply, id 4791, seq 2, length 64
21:03:17.170812 42:15:05:91:3e:98 > 62:70:b6:31:37:04, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.1 tell 172.20.0.11, length 28
21:03:17.170991 62:70:b6:31:37:04 > 42:15:05:91:3e:98, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.11 tell 172.20.0.1, length 28
21:03:17.171007 42:15:05:91:3e:98 > 62:70:b6:31:37:04, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Reply 172.20.0.11 is-at 42:15:05:91:3e:98, length 28
21:03:17.171033 62:70:b6:31:37:04 > 42:15:05:91:3e:98, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Reply 172.20.0.1 is-at 62:70:b6:31:37:04, length 28
```

I am seeing the same result from `global` namespace interface `veth1` that `veth1` is sending the reply to `ceth1`
```
root@debian:~# tcpdump -ennvvq -i veth1
tcpdump: listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
21:03:11.956642 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 44249, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.1: ICMP echo request, id 4791, seq 1, length 64
21:03:11.956873 62:70:b6:31:37:04 > 42:15:05:91:3e:98, IPv4, length 98: (tos 0x0, ttl 63, id 3779, offset 0, flags [none], proto ICMP
(1), length 84)
    192.168.56.1 > 172.20.0.11: ICMP echo reply, id 4791, seq 1, length 64
21:03:12.979073 42:15:05:91:3e:98 > 62:70:b6:31:37:04, IPv4, length 98: (tos 0x0, ttl 64, id 44262, offset 0, flags [DF], proto ICMP (1), length 84)
    172.20.0.11 > 192.168.56.1: ICMP echo request, id 4791, seq 2, length 64
21:03:12.979311 62:70:b6:31:37:04 > 42:15:05:91:3e:98, IPv4, length 98: (tos 0x0, ttl 63, id 3825, offset 0, flags [none], proto ICMP
(1), length 84)
    192.168.56.1 > 172.20.0.11: ICMP echo reply, id 4791, seq 2, length 64
21:03:17.170782 62:70:b6:31:37:04 > 42:15:05:91:3e:98, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.11 tell 172.20.0.1, length 28
21:03:17.171011 42:15:05:91:3e:98 > 62:70:b6:31:37:04, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Request who-has 172.20.0.1 tell 172.20.0.11, length 28
21:03:17.171024 62:70:b6:31:37:04 > 42:15:05:91:3e:98, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Reply 172.20.0.1 is-at 62:70:b6:31:37:04, length 28
21:03:17.171026 42:15:05:91:3e:98 > 62:70:b6:31:37:04, ARP, length 42: Ethernet (len 6), IPv4 (len 4), Reply 172.20.0.11 is-at 42:15:05:91:3e:98, length 28
```

The ICMP packets are seen exchangin between `enp0s8` and `vboxnet0`
```
root@debian:~# tcpdump -ennvvq -i enp0s8 \(arp or icmp\) -l | tee
tcpdump: listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
21:03:11.956662 08:00:27:3c:31:79 > 0a:00:27:00:00:00, IPv4, length 98: (tos 0x0, ttl 63, id 44249, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.56.11 > 192.168.56.1: ICMP echo request, id 4791, seq 1, length 64
21:03:11.956862 0a:00:27:00:00:00 > 08:00:27:3c:31:79, IPv4, length 98: (tos 0x0, ttl 64, id 3779, offset 0, flags [none], proto ICMP
(1), length 84)
    192.168.56.1 > 192.168.56.11: ICMP echo reply, id 4791, seq 1, length 64
21:03:12.979097 08:00:27:3c:31:79 > 0a:00:27:00:00:00, IPv4, length 98: (tos 0x0, ttl 63, id 44262, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.56.11 > 192.168.56.1: ICMP echo request, id 4791, seq 2, length 64
21:03:12.979299 0a:00:27:00:00:00 > 08:00:27:3c:31:79, IPv4, length 98: (tos 0x0, ttl 64, id 3825, offset 0, flags [none], proto ICMP
(1), length 84)
    192.168.56.1 > 192.168.56.11: ICMP echo reply, id 4791, seq 2, length 64
^C4 packets captured
4 packets received by filter
0 packets dropped by kernel
```


In my laptop, vboxnet0 ip is 192.168.56.1
```
magpie% ip addr show vboxnet0
7: vboxnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 0a:00:27:00:00:00 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.1/24 brd 192.168.56.255 scope global vboxnet0
       valid_lft forever preferred_lft forever
    inet6 fe80::800:27ff:fe00:0/64 scope link
       valid_lft forever preferred_lft forever

```

I can also see the exchange of ICMP packets in the tcpdump output.
```
[root@magpie ~]# tcpdump -ennvvqt -i vboxnet0 \(arp or icmp\)
tcpdump: listening on vboxnet0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
08:00:27:3c:31:79 > 0a:00:27:00:00:00, IPv4, length 98: (tos 0x0, ttl 63, id 61909, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.56.11 > 192.168.56.1: ICMP echo request, id 4795, seq 1, length 64
0a:00:27:00:00:00 > 08:00:27:3c:31:79, IPv4, length 98: (tos 0x0, ttl 64, id 35109, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.56.1 > 192.168.56.11: ICMP echo reply, id 4795, seq 1, length 64
08:00:27:3c:31:79 > 0a:00:27:00:00:00, IPv4, length 98: (tos 0x0, ttl 63, id 62042, offset 0, flags [DF], proto ICMP (1), length 84)
    192.168.56.11 > 192.168.56.1: ICMP echo request, id 4795, seq 2, length 64
0a:00:27:00:00:00 > 08:00:27:3c:31:79, IPv4, length 98: (tos 0x0, ttl 64, id 35307, offset 0, flags [none], proto ICMP (1), length 84)
    192.168.56.1 > 192.168.56.11: ICMP echo reply, id 4795, seq 2, length 64

```
