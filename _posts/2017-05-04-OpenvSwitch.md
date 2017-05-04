---
title: "OpenvSwitch之Interface表解析"
layout: post
date: 2017-05-04 12:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
- networks
- openvswitch
- virtualization
star: true
category: blog
author: Haojing Ma
description: OpenvSwitch interface table description
---

| OvsDB表名 | 功能 |
| ------------- | ------------- |
| Open_vSwitch | OVS的配置项 |
| Bridge(重要) | OVS模拟的桥 |
| Port(重要) | OVS模拟的桥上的网口 |
| Interface(重要) | 插入OVS模拟的网口的网络设备 |
| Flow_Table | 流表配置项 |
| QoS | 网络质量管理配置 |
| Queue | QoS输出的队列 |
| Mirror | 网口镜像 |
| Controller | openflow控制器配置信息 |
| Manager | OVSDB管理链接 |
| NetFlow | NetFlow配置信息 |
| SSL | SSL配置信息 |
| sFlow | sFlow配置信息 |
| IPFIX | IPFIX配置信息 |
| Flow_Sample_Collector_Set | Flow_Sample_Collector_Set配置信息 |
| AutoAttach | AutoAttach配置信息 |
---

Interface是openvswitch核心概念之一，对应模拟的是交换机中插入port的网卡设备。一个Port通常只能有一个interface，但也可以有多个interfaces(Bond)。
要了解Interface可以从它在OVSDB中的属性入手。

本章基于openvswtich 2.7.x版本。

总览：

- *core feature*
    - **name**---不可变的string类型，在table中必须是唯一标示
    - **ifindex**---Int类型，是SNMP MIB-II的一个索引值。有了这个可以和SNMP或者sFlow无缝适配。
    - **mac_in_use**---String类型，interface正在使用的MAC地址
    - **mac**---String类型，以太网地址为这个interface设置的，如果没有设置，本地的interface会设置为在bridge ports中最小编号的mac地址，或者Port记录中的mac地址。其他的internal interfaces则随机生成。external interfaces则记录其硬件上的MAC地址。
    - **error**
    - *OpenFlow Port Number*
	   - **ofport**---当在openvswitch中创建了一个interface，就会分配一个OpenFlow port number。
	   - **ofport_request**---为一个interface申请一个ofport。

- *System-Specific Details*	
    - **type**---String类型，(iface_types)支持的类型有system(如eth0)，internal(模拟网络设备，名字如果是和bridge的名字一样则叫local interface)，tap(一个tun/tap设备)，geneve(以太网通过geneve隧道)，gre(RFC2890)，ipsec_gre(RFC2890 over ipsec tunnel)，vxlan(基于以UDP为基础的VXLAN协议上的以太网隧道)，lisp(一个3层的隧道，还在实验阶段)，stt（Stateless TCP Tunnel，），patch(一对虚拟设备，用来模拟插线电缆)

- *Tunnel Options*
	- **options: remote_ip**---String类型，远程隧道端口。必须是个单播地址ipv4或者ipv6，或者是一个关键词flow。如果设置成flow，即remote_ip=flow则flow这个动作(set_field)必须指定tun_dst或tun_ipv6_dst为远程隧道端口的IP。
	- **options: local_ip**---String类型，本地隧道端口。必须是个单播地址ipv4或者ipv6，或者是一个关键词flow。如果设置成flow，即local_ip=flow则flow这个动作(set_field)必须指定tun_src或者tun_ipv6_src为本地隧道端口的IP。
	- **options: in_key**---String类型，隧道接收的包，包的key值含以下三者之一：0，包中如果没有key或者这个key是0，则表示没有这个属性；一个确定的24bit(Geneve，VXLAN，LISP)，32bit（GRE），64bit（STT）数字，隧道会接收以上指定key的包；关键字flow，隧道会接收任意key的包，key会被替换成tun_id，并使用tun_id去匹配流表。
	- **options: out_key**---String类型，隧道发送的包，包的key值被设置为：0，包如果被设置为0，则发送的包不会带有该属性，等同于不设置改属性；一个确定的24bit(Geneve，VXLAN，LISP)，32bit（GRE），64bit（STT）数字，隧道发送的包会带上该key；关键字flow，隧道会设置key值。
	- **options: key**---String类型，如果in_key和out_key是同一数值，则可以合并为该属性。
	- **options: tos**---String类型，在封包的时候添加进ToS字段。
	- **options: ttl**---String类型，可以设置为inherit、数字、不设置，TTL字段会被设置在封装包中，该属性表示该包在路由前最大经过的网段数量。inherit表示从inner packet(必须要是ipv4或ipv6)中拷贝TTL字段，不设置则默认为64。
	- **options: df_default**---布尔类型，如果为true表示不分包bit会被设置在封装包头中，去让MTU发现。
	- *Tunnel Options: vxlan only*
		- **options: exts**---以逗号为分隔的vxlan扩展功能，目前支持的属性有：gbp，VXLAN-GBP允许传输一个包的组策略上下文在vxlan隧道中。具体详见https://tools.ietf.org/html/draft%E2%88%92smith%E2%88%92vxlan%E2%88%92group%E2%88%92policy
	- *Tunnel Options: gre, ipsec_gre, geneve, vxlan*
        - **options: csum** --- String类型，只可能是True、False。在发包中计算封包头的校验，GRE默认支持，vxlan和geneve则需要kernel > 4.0

- *Patch Option:*
    - **options: peer**--- String类型。patch另一端的interface名字。

- *PMD(Poll Mode Driver) Options:*
    - **options: n_rxq**--- String类型，实际是Int。表示为PMD netdev创建几个rx queue。如果设置成0或者更小，则创建1个rx queue。DPDK vhost interface 并不支持该参数。
    - **options: dpdk-devargs**--- String类型。指定PCI地址绑定该port给物理设备。
    - **other_config: pmd-rxq-affinity**--- String类型。PMD设备rx queue绑定至核。
    - **options: vhost-server-path**--- String类型。指定为一个socket，该socket和QEMU创建出来的vhost用户设备相连接。
    - **options: n_rxq_desc**--- String类型，实际是Int，1～4096。指定DPDK类型的端口rx queue size。
    - **options: n_txq_desc**--- String类型， 实际是Int，1～4096。指定DPDK类型的端口tx queue size。

- *mtu*
    - **mtu**--- Integer类型。指定interface的MTU(Maximum Transmission Unit)属性。
    - **mtu_request**--- Integer类型。请求该interface的MTU值。  

- *Interface Status:* 5分钟更新一次，并不是所有的interface都有以下属性，虚拟interface就没有
    - **admin_status**--- String类型，up or down值。该物理链路管理状态。
    - **link_status**--- String类型，up or down值。该物理链路状态。
    - **link_resets**--- Integer类型。该interface的link_status变化次数。
    - **link_speed**--- Integer类型。该interface协商之后网口的速度。
    - **duplex**--- String类型，half or full 值。该interface是全双工还是半双工模式。
    - **lacp_current**--- Boolean类型，探查该interface的LACP状态。
    - **status**--- Sting:String类型的映射组。
    - **status: driver_name**--- 该interface的设备驱动名字。
    - **status: driver_version**--- 该interface的设备驱动版本。
    - **status: firmware_version**--- 该interface的固件版本。
    - **status: source_ip**--- 隧道所用的源ip。
    - **status: tunnel_egress_iface**--- 隧道所用的出口interface。
    - **status: tunnel_egress_iface_carrier**--- up or down。隧道所用的出口interface的状态。

- *Statistics: Successful transmit and receive counters: *
    - **statistics: rx_packets**--- 接收到的包个数。
    - **statistics: rx_bytes** --- 接收到的数据总量。
    - **statistics: tx_packets** --- 发送的包个数。
    - **statistics: tx_bytes** --- 发送的数据总量。

- *Statistics: Receive error*
    - **statistics: rx_dropped** --- 接收到的丢弃包个数。
    - **statistics: rx_frame_err** --- 接收到错误校准帧个数。
    - **statistics: rx_crc_err** --- 接收到的crc校验错误个数。
    - **statistics: rx_errors** --- 接收到的总错误的个数。

- *Statistics: Transmit errors*
	- **statistics: tx_dropped** --- 发送出错的包个数。
	- **statistics: collisions** --- 发送冲突的个数。
	- **statistics: tx_errors** --- 发送错误的总个数。

- *Ingress policy* 这些设置项为该interface设置了包的接收策略。如果interface是物理网卡，则设置了外部流量进入的策略(限制)。如果interface是虚拟网卡，则设置了虚拟机发送的流量的策略(限制)。ingress policy并没有egress QoS那么的精确和高效。
	- **ingress_policing_rate**--- integer，>0。指定网卡接收速率，单位kbps.
	- **ingress_policing_burst**--- integer, >0。指定最大爆发包的大小，单位kb。

- *BFD* 

- *CFD* 802.1ag  Connectivity  Fault Management

- *Bonding Configuration*
    - **other_config: lacp-port-id**--- String类型，Int类型[1,65535]。指定该interface的LACP id。
    - **other_config: lacp-port-priority**--- String类型，Int数值[1,65535]。指定该interface的LACP priority。
    - **other_config: lacp-aggregation-key**--- String类型，Int数值[1,65535]。指定该interface的LACP aggregation key。

- *Virtual Machine Identifiers*
	- **external_ids : attached-mac**--- String类型。该mac地址被编译进该interface的虚拟设备。
	- **external_ids : iface-id**--- String类型。一个系统对于该interface的唯一标示，通常和xs-vif-uuid一样。
	- **external_ids : iface-status**--- String类型，要么是active要么是inactive。虚拟化平台有时会把多个interface和一个iface-id绑定，但于此同时只有一个interface会处于工作状态，其余则处于inactive状态。
	- **external_ids : xs-vif-uuid**--- String类型。The virtual interface associated with this interface.
	- **external_ids : xs-network-uuid**--- String类型。The virtual network to which this interface is attached.
	- **external_ids : vm-id**--- String类型。The  VM to which this interface belongs. On XenServer, this will be the same as external_ids:xs-vm-uuid.
	- **external_ids : xs-vm-uuid**--- String类型。The VM to which this interface belongs.

- *Auto Attach Configuration*
	- **lldp : enable** --- String类型, either true or false。True to enable LLDP on this Interface. If  not  specified,  LLDP will be disabled by default.

- *Flow control Configuration*---以太网流控制IEEE 802.1Qbb，其中提供了通过MAC帧控制链路层的流。只有网卡支持dpdk的才支持该配置。
	- **options : rx-flow-ctrl**--- String类型, either true or false。当设置成true时，物理端口上所设置的Rx flow生效。当设置成false时，则不生效。
	- **options : tx-flow-ctrl**--- String类型, either true or false。当设置成true时，物理端口上所设置的Tx flow生效。当设置成false时，则不生效。
	- **options : flow-ctrl-autoneg**--- String类型, either true or false。Set  to true to enable flow control auto negotiation on physical ports. By default, auto-neg is disabled.

- *Rx CheckSum offload Configuration*--- 网络设备支持dpdk才支持该配置。只有incoming packets才有的功能。
	- **options : rx-checksum-offload**--- String, either true or false.Set to false to disble Rx checksum  offloading  on  dpdk-physical ports. By default, Rx checksum offload is enabled.

- *Common Column*
	- **other_config:** map of string-string pairs
	- **external_ids:** map of string-string pairs

参考：http://openvswitch.org/support/dist-docs/ovs-vswitchd.conf.db.5.html

