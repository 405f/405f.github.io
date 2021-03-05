---
title: "Tcpdump"
date: 2021-03-05T12:47:38+08:00
draft: false
---
tcpdump的使用方法

### ifconfig 查看主机的网卡
###### output example 

```text
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> mtu 16384
	options=1203<RXCSUM,TXCSUM,TXSTATUS,SW_TIMESTAMP>
	inet 127.0.0.1 netmask 0xff000000
	inet6 ::1 prefixlen 128
	inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1
	nd6 options=201<PERFORMNUD,DAD>
gif0: flags=8010<POINTOPOINT,MULTICAST> mtu 1280
stf0: flags=0<> mtu 1280
en5: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	ether ac:de:48:00:11:22
	inet6 fe80::aede:48ff:fe00:1122%en5 prefixlen 64 scopeid 0x4
	nd6 options=201<PERFORMNUD,DAD>
	media: autoselect (100baseTX <full-duplex>)
	status: active
ap1: flags=8802<BROADCAST,SIMPLEX,MULTICAST> mtu 1500
	options=400<CHANNEL_IO>
	ether 3e:22:fb:51:2e:dd
	media: autoselect
	status: inactive
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	options=400<CHANNEL_IO>
	ether 3c:22:fb:51:2e:dd
	inet6 fe80::105b:1c88:5ee6:98e2%en0 prefixlen 64 secured scopeid 0x6
	inet 192.168.1.181 netmask 0xfffffe00 broadcast 192.168.1.255
	nd6 options=201<PERFORMNUD,DAD>
	media: autoselect
	status: active
awdl0: flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
	options=400<CHANNEL_IO>
	ether 5e:58:54:9f:ee:ce
	inet6 fe80::5c58:54ff:fe9f:eece%awdl0 prefixlen 64 scopeid 0x7
	nd6 options=201<PERFORMNUD,DAD>
	media: autoselect
	status: active
llw0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	options=400<CHANNEL_IO>
	ether 5e:58:54:9f:ee:ce
	inet6 fe80::5c58:54ff:fe9f:eece%llw0 prefixlen 64 scopeid 0x8
	nd6 options=201<PERFORMNUD,DAD>
	media: autoselect
	status: active
en3: flags=8963<UP,BROADCAST,SMART,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
	options=460<TSO4,TSO6,CHANNEL_IO>
	ether 82:f4:80:27:64:05
	media: autoselect <full-duplex>
	status: inactive
en4: flags=8963<UP,BROADCAST,SMART,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
	options=460<TSO4,TSO6,CHANNEL_IO>
	ether 82:f4:80:27:64:04
	media: autoselect <full-duplex>
	status: inactive
en1: flags=8963<UP,BROADCAST,SMART,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
	options=460<TSO4,TSO6,CHANNEL_IO>
	ether 82:f4:80:27:64:01
	media: autoselect <full-duplex>
	status: inactive
en2: flags=8963<UP,BROADCAST,SMART,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
	options=460<TSO4,TSO6,CHANNEL_IO>
	ether 82:f4:80:27:64:00
	media: autoselect <full-duplex>
	status: inactive
bridge0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	options=63<RXCSUM,TXCSUM,TSO4,TSO6>
	ether 82:f4:80:27:64:01
	Configuration:
		id 0:0:0:0:0:0 priority 0 hellotime 0 fwddelay 0
		maxage 0 holdcnt 0 proto stp maxaddr 100 timeout 1200
		root id 0:0:0:0:0:0 priority 0 ifcost 0 port 0
		ipfilter disabled flags 0x0
	member: en1 flags=3<LEARNING,DISCOVER>
	        ifmaxaddr 0 port 11 priority 0 path cost 0
	member: en2 flags=3<LEARNING,DISCOVER>
	        ifmaxaddr 0 port 12 priority 0 path cost 0
	member: en3 flags=3<LEARNING,DISCOVER>
	        ifmaxaddr 0 port 9 priority 0 path cost 0
	member: en4 flags=3<LEARNING,DISCOVER>
	        ifmaxaddr 0 port 10 priority 0 path cost 0
	nd6 options=201<PERFORMNUD,DAD>
	media: <unknown type>
	status: inactive
utun0: flags=8051<UP,POINTOPOINT,RUNNING,MULTICAST> mtu 1380
	inet6 fe80::fdfa:d397:31b3:da80%utun0 prefixlen 64 scopeid 0xe
	nd6 options=201<PERFORMNUD,DAD>
utun1: flags=8051<UP,POINTOPOINT,RUNNING,MULTICAST> mtu 2000
	inet6 fe80::df55:bfea:de7e:9001%utun1 prefixlen 64 scopeid 0xf
	nd6 options=201<PERFORMNUD,DAD>

```
en0: 表示第一块网卡
mut: 最大传输单元
status: 网卡的使用状态

### tcpdump example
-
 ```txt
sudo tcpdump host baidu.com -i en0
```

#### www.baidu.com 

sudo tcpdump host www.baidu.com

wget www.baidu.com

```txt

➜  network sudo tcpdump  host www.baidu.com
```
通过执行这个命令可以得到[tcp.log](https://github.com/dtest11/dtest11.github.io/blob/master/playground/network/tcp.log#L1)
通过这个文件我们可以分析下tcp的三次握手
- Flags 参数
1. [S]：SYN同步标识
2. [.]：.表示ACK确认标识
3. [S.]：SYN同步标识，以及确认[S]的ACK
4. [P.]：PSH,push推送，数据传输
5. [R.]：RST,连接重置
6. [F.]：FIN结束连接
7. [DF]：Don't Fragment（不要碎裂），当DF=0时，表示允许分片，一般-v时才有这个标识
8. [FP.]：标记FIN、PUSH、ACK组合，这样做是为了提升网络效率，减少数据来回确认等

- seq为序列号
- ack为确认码
- win为滑动窗口大小
- length为承载的数据(payload)长度length，如果没有数据则为0
