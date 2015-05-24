---
layout:     post
title:      Linux Kernel Network Framework 
date:       2015-05-18 19:54
summary:    Linux kernel network stack is the most interesting part I ever dig, in this series, I like to introduce you the whole framework of network stack in Linux kernel, in this post, I show you the stack architecture, and later, I will dig it bottom up, layer by layer.
categories: network stack
---

Talking about network, I believe everyone knows Internet, each day, we live and work, we can't get ride of it. but how our computer, or phone talk to each other, or how network works, that alway be the interesting thing we like to know, it is a new world of our lives, it is the basic of communication, which connect all of us.  
  
In this series, I like to show something about network, I hopes it can help you to figure out how network works. To make network manageable, and network devices can communicate with each other, no matter what OS they run, we need some unified rules to define how network device should react when it receives something from others, which came out a global network organization, it created a unified network model, which splits network into 7 layers, each layer has its own mission and self-defined functionality, which includes what information it should take, how it should communication with other layers, etc. These layers are <strong>physical layer</strong>, <strong>data link layer</strong>, <strong>network layer</strong>, <strong>transport layer</strong>, <strong>session layer</strong>, <strong>representation layer</strong>, and <strong>application layer</strong>, but to keep things simple, in practical use, we merged session layer and representation layer into application layer, which become only 5 layer in practice.  
  
  
<big><strong>Functionality definition</strong></big>:  
<strong>Physical Layer</strong>: it provides capability that network message transmit on physical media.  
<strong>Data Link Layer</strong>: it defines local address, which locatable in local network, provide network management capability in local network.  
<strong>Network Layer</strong>: it defines global address database, which locatable in global network.  
<strong>Transport Layer</strong>: it provides connection management capability.  
<strong>Application Layer</strong>: it provides communication capability between network devices, convert messages into human readable format.  
  
  
In Linux, there are 4 layers in the bottom implemented in kernel space, and application layer is implemented in user space, in this series, I mainly focus on kernel stack, the major part of physical layer is device driver of hardware, it differ from media to media, like WIFI, Ethernet, ATM, GPON,  etc, so in Linux kernel, the most unified part is from data link layer to transport layer, and that is my topic.  


![network framework](/images/Network_framework_state.png)
<center><small>Figure 1: Linux kernel network framework</small></center>  


##Data Link Layer
  
In order to communication with physical layer, it provide something interfaces, like network interrupt handler, poll API, etc, which can deliver packets up and down.  
Receiving data from physical layer, it usually do in interrupt, as network throughput capability goes higher, it already become a bottleneck, so there are several optimization here, like NAPI, network interrupt threading.  
If source network interface data received from is inside bridge, data will be delivered into bridge stack, or it will take off layer 2 header and deliver packets up to network layer.  
when there are some packet need to send out, it will insert its layer 2 header, like MAC address, VLAN, etc, into packet, and deliver packet to target network interface's transmitting queue.  
In bridge stack, packets will be handled based on layer 2 header, like MAC address, VLAN, briefly speaking, packets in bridge stack have two direction, one is deliver packet up to network layer if target MAC address is local machine, the other is deliver packets down to physical layer if target MAC address is other host, before reaching target, packets need to pass through layer 2 firewall system.  
  

##Network Layer

when network layer init, it registers its supported layer 3 protocol type in a linked list, like IPV4, IPV6, and provide a receiving HOOK function, like ip_rcv for IPV4, which used to receive packets from layer 2, so when data link layer deliver packets up, it will iterate this linked list, and call its receiving HOOK function if layer 3 protocol match.  
In network layer, packets will be handled based on IP header, its target direction depends on the result of route table look-up, up to transport layer if target IP address is local machine, or down to data link layer if target IP address is other host, before reaching target, packets need to pass through layer 3 firewall system.  
when packet need to deliver up to transport layer, it will take off layer 3 header before delivering.  
when packet need to deliver down to data link layer, it will insert layer 3 header based on the result of route table look-up, and deliver packets to target transmitting queue.  
  

##Transport Layer  
  
The major task in this layer is connection management, whenever devices communicate, they need to establish a connection first, it is done on this layer via socket, which create a structure called <code>struct sock</code> to record and maintain all the information of this connection, and insert it into hash table of each supported protocol, like UDP, TCP.  
when network layer deliver packets up, it will find the connection related in hash table, if succeed, it take off layer 4 header, append packets into socket's receiving queue, and wake up application in user space, which will receive packets into application layer, if no connection find in hash table, packets will be dropped. 
when application send message down, it store at socket's transmitting queue, transport layer will dequeue from transmitting queue, and find the destination in route table based on information in <code>struct sock</code>, before delivering packets down, it will fill layer 4 header, and calculate checksum.  

##conclusion  

each layer in Linux kernel is self-organized, it just cares what input from neighbor layers, what output to neighbor layers, which have better simplicity and flexibility. When deliver packet down and up, it just manipulate the data pointer, no real memory operation, it can benefit memory management, and make data delivering between layers faster. Kernel use linked list and hash table to manage protocol in each layer, it can grow up dynamically, and protocol designing is modularized, so it can keeping updating at runtime, it offer much scalability, and we can benefit from this design technical, for our design, we can learn a lot from it.  

This is the whole network stack framework in Linux kernel, actually, they are a lot I don't show here, in next post, I will focus the details in each layers, any questions, don't hesitate to drop me a email.  




 