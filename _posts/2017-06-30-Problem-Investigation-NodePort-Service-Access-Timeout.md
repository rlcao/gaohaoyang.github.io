---
layout: post
title:  "Problem Investigation: Node Port Service Access from Non-K8s-Nodes Timed Out"
categories: Network
tags:  Weave Network ProblemAnalysis
author: RLCAO
---
* content
{:toc}
## Background
Today, I have setup a k8s cluster using weave network. The versions will be mentioned in later parts of this blog. However, it will time out when accessing a service through node port of the service, if originated from non-k8s-nodes, while it works as expect if originated from k8s cluster nodes. This is really wired issue which should not happen in the first place.

I am going to collect the related data, and explain to you what happened under the surface. I hope you will learn something regarding weave network and iptables. 

Sections of this blog:
* [Environment](#environment)
* [Problem](#problem)
* [Gathered Facts](#gathered-facts)
* [Analysis](#analysis)
* [Explaination](#explaination)
* [Solution](#solution)

## Environment
1. Kubernetes nodes:
```text
[root@A01-R06-I13-30 ~]# kubectl get nodes -o wide
NAME                       STATUS    AGE       VERSION   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION
a01-r06-i12-109.jd.local   Ready     28d       v1.6.4    <none>        CentOS Linux 7 (Core)   3.10.0-327.28.3.el7.x86_64
a01-r06-i12-129.jd.local   Ready     28d       v1.6.4    <none>        CentOS Linux 7 (Core)   3.10.0-327.28.3.el7.x86_64
a01-r06-i13-30.jd.local    Ready     29d       v1.6.4    <none>        CentOS Linux 7 (Core)   3.10.0-327.28.3.el7.x86_64
a01-r06-i13-41.jd.local    Ready     28d       v1.6.4    <none>        CentOS Linux 7 (Core)   3.10.0-327.28.3.el7.x86_64
a01-r06-i13-96.jd.local    Ready     28d       v1.6.4    <none>        CentOS Linux 7 (Core)   3.10.0-327.28.3.el7.x86_64
```
2. Kubernetes services
```text
[root@A01-R06-I13-30 ~]# kc -n cloud-op get services -o wide
NAME               CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE       SELECTOR
pga-hawkeye        10.106.234.218   <nodes>       80:30502/TCP     21h       app=pga-hawkeye
```
3. Kubernetes pods
```text
[root@A01-R06-I13-30 ~]# kc -n cloud-op get pods -o wide
NAME                                READY     STATUS    RESTARTS   AGE       IP          NODE
pga-hawkeye-3696951108-t2p54        1/1       Running   0          21h       10.32.0.1   a01-r06-i12-109.jd.local
pga-hawkeye-3696951108-zz00k        1/1       Running   0          21h       10.36.0.2   a01-r06-i13-41.jd.local
```
4. Weave version
```text
[root@A01-R06-I13-30 ~]# kubectl -n kube-system describe pod weave-net-2l4gg | grep image -i
    Image:              bj01rg.jcloud.com:5000/weave-kube:1.9.6
    Image ID:           docker-pullable://bj01rg.jcloud.com:5000/weave-kube@sha256:b7007a1cac96295de89cd898c805530ef116567acac1bda6f199079ea26a988f
    Image:              bj01rg.jcloud.com:5000/weave-npc:1.9.6
    Image ID:           docker-pullable://bj01rg.jcloud.com:5000/weave-npc@sha256:9efc2c641c87f61d3db2288135a3fd09e1ef81a508a80b4afd4a3078c4446fab
```
5. IPtables contents
```text
# Generated by iptables-save v1.4.21 on Fri Jun 30 15:47:14 2017
*raw
:PREROUTING ACCEPT [12132081:4723414726]
:OUTPUT ACCEPT [12062601:5787932783]
-A PREROUTING -d 192.168.0.0/24 -p tcp -m tcp --dport 80 -j TRACE
COMMIT
# Completed on Fri Jun 30 15:47:14 2017
# Generated by iptables-save v1.4.21 on Fri Jun 30 15:47:14 2017
*nat
:PREROUTING ACCEPT [3:217]
:INPUT ACCEPT [3:217]
:OUTPUT ACCEPT [19:1012]
:POSTROUTING ACCEPT [19:1012]
:DOCKER - [0:0]
:KUBE-MARK-DROP - [0:0]
:KUBE-MARK-MASQ - [0:0]
:KUBE-NODEPORTS - [0:0]
:KUBE-POSTROUTING - [0:0]
:KUBE-SEP-45DFFRCB4GG3GG7O - [0:0]
:KUBE-SEP-CAXV7P7L5B7H2L3D - [0:0]
:KUBE-SEP-DE47FFHF5HNKFMFA - [0:0]
:KUBE-SEP-EWLN57M3Q4TLD7XG - [0:0]
:KUBE-SEP-HEDYP6XBU6RFUTEO - [0:0]
:KUBE-SEP-MEUVLUM4BVAUFHM5 - [0:0]
:KUBE-SEP-RV4OGJNKSAPDBJAD - [0:0]
:KUBE-SEP-SPOPWFFR4N3SSTWV - [0:0]
:KUBE-SEP-ZRSQLFEEEQYMTQU4 - [0:0]
:KUBE-SERVICES - [0:0]
:KUBE-SVC-ERIFXISQEP7F7OF4 - [0:0]
:KUBE-SVC-HVR2LWQSX2X7NHPX - [0:0]
:KUBE-SVC-NPX46M4PTMTKRN6Y - [0:0]
:KUBE-SVC-SKCJ5GLWZD33IDDL - [0:0]
:KUBE-SVC-TCOU7JCQXEZGVUNU - [0:0]
:KUBE-SVC-XGLOHA7QRQ3V22RZ - [0:0]
:KUBE-SVC-ZH6ROOLLFSIBNEVY - [0:0]
:WEAVE - [0:0]
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A POSTROUTING -j WEAVE
-A DOCKER -i docker0 -j RETURN
-A KUBE-MARK-DROP -j MARK --set-xmark 0x8000/0x8000
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-NODEPORTS -p tcp -m comment --comment "kube-system/dashboard-nodeport:" -m tcp --dport 31087 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "kube-system/dashboard-nodeport:" -m tcp --dport 31087 -j KUBE-SVC-ZH6ROOLLFSIBNEVY
-A KUBE-NODEPORTS -p tcp -m comment --comment "cloud-op/pga-hawkeye:pga-hawkeye" -m tcp --dport 30502 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "cloud-op/pga-hawkeye:pga-hawkeye" -m tcp --dport 30502 -j KUBE-SVC-SKCJ5GLWZD33IDDL
-A KUBE-NODEPORTS -p tcp -m comment --comment "cloud-op/kibana-prod-gz01:port-kibana-prod-gz01" -m tcp --dport 31201 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "cloud-op/kibana-prod-gz01:port-kibana-prod-gz01" -m tcp --dport 31201 -j KUBE-SVC-HVR2LWQSX2X7NHPX
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
-A KUBE-SEP-45DFFRCB4GG3GG7O -s 172.19.13.30/32 -m comment --comment "default/kubernetes:https" -j KUBE-MARK-MASQ
-A KUBE-SEP-45DFFRCB4GG3GG7O -p tcp -m comment --comment "default/kubernetes:https" -m recent --set --name KUBE-SEP-45DFFRCB4GG3GG7O --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 172.19.13.30:6443
-A KUBE-SEP-CAXV7P7L5B7H2L3D -s 10.44.0.1/32 -m comment --comment "kube-system/kube-dns:dns-tcp" -j KUBE-MARK-MASQ
-A KUBE-SEP-CAXV7P7L5B7H2L3D -p tcp -m comment --comment "kube-system/kube-dns:dns-tcp" -m tcp -j DNAT --to-destination 10.44.0.1:53
-A KUBE-SEP-DE47FFHF5HNKFMFA -s 10.36.0.0/32 -m comment --comment "kube-system/dashboard-nodeport:" -j KUBE-MARK-MASQ
-A KUBE-SEP-DE47FFHF5HNKFMFA -p tcp -m comment --comment "kube-system/dashboard-nodeport:" -m tcp -j DNAT --to-destination 10.36.0.0:9090
-A KUBE-SEP-EWLN57M3Q4TLD7XG -s 10.36.0.2/32 -m comment --comment "cloud-op/pga-hawkeye:pga-hawkeye" -j KUBE-MARK-MASQ
-A KUBE-SEP-EWLN57M3Q4TLD7XG -p tcp -m comment --comment "cloud-op/pga-hawkeye:pga-hawkeye" -m tcp -j DNAT --to-destination 10.36.0.2:80
-A KUBE-SEP-HEDYP6XBU6RFUTEO -s 10.40.0.1/32 -m comment --comment "cloud-op/kibana-prod-gz01:port-kibana-prod-gz01" -j KUBE-MARK-MASQ
-A KUBE-SEP-HEDYP6XBU6RFUTEO -p tcp -m comment --comment "cloud-op/kibana-prod-gz01:port-kibana-prod-gz01" -m tcp -j DNAT --to-destination 10.40.0.1:5601
-A KUBE-SEP-MEUVLUM4BVAUFHM5 -s 10.44.0.1/32 -m comment --comment "kube-system/kube-dns:dns" -j KUBE-MARK-MASQ
-A KUBE-SEP-MEUVLUM4BVAUFHM5 -p udp -m comment --comment "kube-system/kube-dns:dns" -m udp -j DNAT --to-destination 10.44.0.1:53
-A KUBE-SEP-RV4OGJNKSAPDBJAD -s 10.32.0.1/32 -m comment --comment "cloud-op/pga-hawkeye:pga-hawkeye" -j KUBE-MARK-MASQ
-A KUBE-SEP-RV4OGJNKSAPDBJAD -p tcp -m comment --comment "cloud-op/pga-hawkeye:pga-hawkeye" -m tcp -j DNAT --to-destination 10.32.0.1:80
-A KUBE-SEP-SPOPWFFR4N3SSTWV -s 10.35.0.0/32 -m comment --comment "cloud-op/kibana-prod-gz01:port-kibana-prod-gz01" -j KUBE-MARK-MASQ
-A KUBE-SEP-SPOPWFFR4N3SSTWV -p tcp -m comment --comment "cloud-op/kibana-prod-gz01:port-kibana-prod-gz01" -m tcp -j DNAT --to-destination 10.35.0.0:5601
-A KUBE-SEP-ZRSQLFEEEQYMTQU4 -s 10.36.0.0/32 -m comment --comment "kube-system/kubernetes-dashboard:" -j KUBE-MARK-MASQ
-A KUBE-SEP-ZRSQLFEEEQYMTQU4 -p tcp -m comment --comment "kube-system/kubernetes-dashboard:" -m tcp -j DNAT --to-destination 10.36.0.0:9090
-A KUBE-SERVICES -d 10.96.0.10/32 -p udp -m comment --comment "kube-system/kube-dns:dns cluster IP" -m udp --dport 53 -j KUBE-SVC-TCOU7JCQXEZGVUNU
-A KUBE-SERVICES -d 10.96.0.10/32 -p tcp -m comment --comment "kube-system/kube-dns:dns-tcp cluster IP" -m tcp --dport 53 -j KUBE-SVC-ERIFXISQEP7F7OF4
-A KUBE-SERVICES -d 10.99.131.78/32 -p tcp -m comment --comment "kube-system/kubernetes-dashboard: cluster IP" -m tcp --dport 80 -j KUBE-SVC-XGLOHA7QRQ3V22RZ
-A KUBE-SERVICES -d 10.111.15.164/32 -p tcp -m comment --comment "kube-system/dashboard-nodeport: cluster IP" -m tcp --dport 9090 -j KUBE-SVC-ZH6ROOLLFSIBNEVY
-A KUBE-SERVICES -d 10.106.234.218/32 -p tcp -m comment --comment "cloud-op/pga-hawkeye:pga-hawkeye cluster IP" -m tcp --dport 80 -j KUBE-SVC-SKCJ5GLWZD33IDDL
-A KUBE-SERVICES -d 10.97.190.174/32 -p tcp -m comment --comment "cloud-op/kibana-prod-gz01:port-kibana-prod-gz01 cluster IP" -m tcp --dport 5601 -j KUBE-SVC-HVR2LWQSX2X7NHPX
-A KUBE-SERVICES -d 10.96.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
-A KUBE-SVC-ERIFXISQEP7F7OF4 -m comment --comment "kube-system/kube-dns:dns-tcp" -j KUBE-SEP-CAXV7P7L5B7H2L3D
-A KUBE-SVC-HVR2LWQSX2X7NHPX -m comment --comment "cloud-op/kibana-prod-gz01:port-kibana-prod-gz01" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-SPOPWFFR4N3SSTWV
-A KUBE-SVC-HVR2LWQSX2X7NHPX -m comment --comment "cloud-op/kibana-prod-gz01:port-kibana-prod-gz01" -j KUBE-SEP-HEDYP6XBU6RFUTEO
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-45DFFRCB4GG3GG7O --mask 255.255.255.255 --rsource -j KUBE-SEP-45DFFRCB4GG3GG7O
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -j KUBE-SEP-45DFFRCB4GG3GG7O
-A KUBE-SVC-SKCJ5GLWZD33IDDL -m comment --comment "cloud-op/pga-hawkeye:pga-hawkeye" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-RV4OGJNKSAPDBJAD
-A KUBE-SVC-SKCJ5GLWZD33IDDL -m comment --comment "cloud-op/pga-hawkeye:pga-hawkeye" -j KUBE-SEP-EWLN57M3Q4TLD7XG
-A KUBE-SVC-TCOU7JCQXEZGVUNU -m comment --comment "kube-system/kube-dns:dns" -j KUBE-SEP-MEUVLUM4BVAUFHM5
-A KUBE-SVC-XGLOHA7QRQ3V22RZ -m comment --comment "kube-system/kubernetes-dashboard:" -j KUBE-SEP-ZRSQLFEEEQYMTQU4
-A KUBE-SVC-ZH6ROOLLFSIBNEVY -m comment --comment "kube-system/dashboard-nodeport:" -j KUBE-SEP-DE47FFHF5HNKFMFA
-A WEAVE -s 10.32.0.0/12 -d 224.0.0.0/4 -j RETURN
-A WEAVE ! -s 10.32.0.0/12 -d 10.32.0.0/12 -j MASQUERADE
-A WEAVE -s 10.32.0.0/12 ! -d 10.32.0.0/12 -j MASQUERADE
COMMIT
# Completed on Fri Jun 30 15:47:14 2017
# Generated by iptables-save v1.4.21 on Fri Jun 30 15:47:14 2017
*filter
:INPUT ACCEPT [1804:576773]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [1786:714776]
:DOCKER - [0:0]
:DOCKER-ISOLATION - [0:0]
:KUBE-FIREWALL - [0:0]
:KUBE-SERVICES - [0:0]
:WEAVE-NPC - [0:0]
:WEAVE-NPC-DEFAULT - [0:0]
:WEAVE-NPC-INGRESS - [0:0]
-A INPUT -j KUBE-FIREWALL
-A INPUT -d 172.17.0.1/32 -i docker0 -p tcp -m tcp --dport 6783 -j DROP
-A INPUT -d 172.17.0.1/32 -i docker0 -p udp -m udp --dport 6783 -j DROP
-A INPUT -d 172.17.0.1/32 -i docker0 -p udp -m udp --dport 6784 -j DROP
-A INPUT -i docker0 -p udp -m udp --dport 53 -j ACCEPT
-A INPUT -i docker0 -p tcp -m tcp --dport 53 -j ACCEPT
-A FORWARD -i docker0 -o weave -j DROP
-A FORWARD -j DOCKER-ISOLATION
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A FORWARD -o weave -j WEAVE-NPC
-A FORWARD -o weave -m state --state NEW -j NFLOG --nflog-group 86
-A FORWARD -o weave -j DROP
-A FORWARD -i weave ! -o weave -j ACCEPT
-A FORWARD -o weave -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A OUTPUT -j KUBE-FIREWALL
-A DOCKER-ISOLATION -j RETURN
-A KUBE-FIREWALL -m comment --comment "kubernetes firewall for dropping marked packets" -m mark --mark 0x8000/0x8000 -j DROP
-A WEAVE-NPC -m state --state RELATED,ESTABLISHED -j ACCEPT
-A WEAVE-NPC -d 224.0.0.0/4 -j ACCEPT
-A WEAVE-NPC -m state --state NEW -j WEAVE-NPC-DEFAULT
-A WEAVE-NPC -m state --state NEW -j WEAVE-NPC-INGRESS
-A WEAVE-NPC -m set --match-set weave-local-pods src -m set ! --match-set weave-local-pods dst -j ACCEPT
-A WEAVE-NPC-DEFAULT -m set --match-set weave-m:]I^rFiD#Rh4(7R=Vw*T#o3A dst -j ACCEPT
-A WEAVE-NPC-DEFAULT -m set --match-set weave-k?Z;25^M}|1s7P3|H9i;*;MhG dst -j ACCEPT
-A WEAVE-NPC-DEFAULT -m set --match-set weave-4vtqMI+kx/2]jD%_c0S%thO%V dst -j ACCEPT
-A WEAVE-NPC-DEFAULT -m set --match-set weave-iuZcey(5DeXbzgRFs8Szo]+@p dst -j ACCEPT
COMMIT
# Completed on Fri Jun 30 15:47:14 2017 
```                                                                                            
6. IPset list on kubernetes master node 
```text
Name: weave-local-pods                                 
Type: hash:ip                                         
Revision: 1                                           
Header: family inet hashsize 1024 maxelem 65536       
Size in memory: 16544                                 
References: 2                                         
Members:                                               
10.44.0.1                                             
Name: weave-m:]I^rFiD#Rh4(7R=Vw*T#o3A                 
Type: hash:ip                                         
Revision: 1                                           
Header: family inet hashsize 1024 maxelem 65536       
Size in memory: 16528                                 
References: 1                                         
Members:                                               
Name: weave-k?Z;25^M}|1s7P3|H9i;*;MhG                 
Type: hash:ip                                         
Revision: 1                                           
Header: family inet hashsize 1024 maxelem 65536       
Size in memory: 16528                                 
References: 1                                         
Members:                                               
Name: weave-4vtqMI+kx/2]jD%_c0S%thO%V                 
Type: hash:ip                                         
Revision: 1                                           
Header: family inet hashsize 1024 maxelem 65536       
Size in memory: 16528                                 
References: 1                                         
Members:                                               
Name: weave-iuZcey(5DeXbzgRFs8Szo]+@p                 
Type: hash:ip                                         
Revision: 1                                           
Header: family inet hashsize 1024 maxelem 65536       
Size in memory: 16544                                 
References: 1                                         
Members:                                               
10.44.0.1 
```

## Problem
Access pga-hawkeye from nodes outside of kubernetes will timeout
```text
[root@A06-R12-302F0204-I33-31 ~]# curl 172.19.13.30:30502  
# Above command line will time out
```
However, accessing from nodes inside kubernetes will succeed
```text
[root@A01-R06-I13-30 ~]#  curl 172.19.13.30:30502
# Above command will generate response content
......
```

## Gathered Facts
1. WEAVE-NPC logs
```text
DEBU: 2017/06/29 10:32:27.744648 AddPod ignored for pod cloud-op/pga-hawkeye-3696951108-zz00k on node
DEBU: 2017/06/29 10:32:27.744839 AddPod ignored for pod cloud-op/pga-hawkeye-3696951108-t2p54 on node
...... 
WARN: 2017/06/30 07:54:54.250360 TCP connection from 172.27.33.31:37276 to 10.32.0.1:80 blocked by Weave NPC.
WARN: 2017/06/30 07:55:02.266348 TCP connection from 172.27.33.31:37276 to 10.36.0.2:80 blocked by Weave NPC.
WARN: 2017/06/30 07:55:18.282355 TCP connection from 172.27.33.31:37276 to 10.36.0.2:80 blocked by Weave NPC.
WARN: 2017/06/30 07:55:50.346385 TCP connection from 172.27.33.31:37276 to 10.36.0.2:80 blocked by Weave NPC.
......
```
2. TCP Dump on nodes hosting service pods, when there is successful service access
```text
16:04:40.888420 fe:14:a7:27:4f:01 > 4e:7c:72:b0:ad:eb, ethertype IPv4 (0x0800), length 66: 10.44.0.0.27860 > 10.36.0.2.http: Flags [S], seq 3709077129, win 43690, options [mss 65495,nop,nop,sackOK,nop,wscale 9], length 0
    0x0000:  4e7c 72b0 adeb fe14 a727 4f01 0800 4500  N|r......'O...E.
    0x0010:  0034 2b96 4000 4006 fadc 0a2c 0000 0a24  .4+.@.@....,...$
    0x0020:  0002 6cd4 0050 dd14 0689 0000 0000 8002  ..l..P..........
    0x0030:  aaaa 652d 0000 0204 ffd7 0101 0402 0103  ..e-............
    0x0040:  0309                                     ..
16:04:40.888472 4e:7c:72:b0:ad:eb > fe:14:a7:27:4f:01, ethertype IPv4 (0x0800), length 66: 10.36.0.2.http > 10.44.0.0.27860: Flags [S.], seq 2402753578, ack 3709077130, win 26720, options [mss 1336,nop,nop,sackOK,nop,wscale 9], length 0
    0x0000:  fe14 a727 4f01 4e7c 72b0 adeb 0800 4500  ...'O.N|r.....E.
    0x0010:  0034 0000 4000 4006 2673 0a24 0002 0a2c  .4..@.@.&s.$...,
    0x0020:  0000 0050 6cd4 8f37 1c2a dd14 068a 8012  ...Pl..7.*......
    0x0030:  6860 1478 0000 0204 0538 0101 0402 0103  h`.x.....8......
    0x0040:  0309                                     ..
16:04:40.889203 fe:14:a7:27:4f:01 > 4e:7c:72:b0:ad:eb, ethertype IPv4 (0x0800), length 54: 10.44.0.0.27860 > 10.36.0.2.http: Flags [.], ack 1, win 86, length 0
    0x0000:  4e7c 72b0 adeb fe14 a727 4f01 0800 4500  N|r......'O...E.
    0x0010:  0028 2b97 4000 4006 fae7 0a2c 0000 0a24  .(+.@.@....,...$
    0x0020:  0002 6cd4 0050 dd14 068a 8f37 1c2b 5010  ..l..P.....7.+P.
    0x0030:  0056 9f07 0000                           .V....
16:04:40.889370 fe:14:a7:27:4f:01 > 4e:7c:72:b0:ad:eb, ethertype IPv4 (0x0800), length 136: 10.44.0.0.27860 > 10.36.0.2.http: Flags [P.], seq 1:83, ack 1, win 86, length 82
......
```

## Problem Analysis
1. iptables process flow
![image of iptables process flow](https://s3.amazonaws.com/cp-s3/wp-content/uploads/2015/09/08085516/iptables-Flowchart.jpg)
2. Kubernetes chains includes following:
* KUBE-SERVICES: entry point of kubernetes services
* KUBE-NODEPORTS: entry point of kubernetes nodeport
* KUBE-SVC-<>: specific service entry for a service in kubectl get services
* KUBE-SEP-<>: service endpoint of specific service
* KUBE chains to update dst/port for data-streams goes to kubernetes services
3. Weave chains
* WEAVE: in/out bidirection masquerade for datastreams going in/out of weave network
* WEAVE-NPC: ipset filtering, which is managed by weave-npc who will get notified whenever there is topology change in kubernetes master.
4. Hookpoint of above mentioned chains
* PREROUTING -> nat -> KUBE-SERVICES
* OUTPUT -> nat -> KUBE-SERVICES
* FORWARD ->filter -> WEAVE-NPC
* POSTROUTING ->nat ->WEAVE

## Explaination
Access pga-hawkeye from nodes outside of kubernetes will timeout. This is because datastreams from outside will go through chains:
1. PREROUTING, where dst and port get changed by KUBE-SERVICES chains
2. FORWARD, where WEAVE-NPC takes place, and packets gets blocked, since the src/port does not match following rule:
```text
-A WEAVE-NPC -m set --match-set weave-local-pods src -m set ! --match-set weave-local-pods dst -j ACCEPT
```

However, accessing from nodes inside kubernetes will succeed. This is becuase datastreams originated from hosts will go through chains:
1. OUTPUT, where dst and port get changed by KUBE-SERVICES chains
2. POSTROUTING, where WEAVE takes place and masquerade happens, which updated the src/port 

## Solution
Update a rule in custom chain WEAVE-NPC created by weave-npc to accept datastreams originated outside of k8s nodes.
* The original rule in WEAVE-NPC chain
```text
# datastream originated from local pods, goes to non local pods will be accepted
-A WEAVE-NPC -m set --match-set weave-local-pods src -m set ! --match-set weave-local-pods dst -j ACCEPT
```
* Update this chain to be following
```text
# any datastream goes to non local pods will be accepted
-A WEAVE-NPC -m set ! --match-set weave-local-pods dst -j ACCEPT
```
